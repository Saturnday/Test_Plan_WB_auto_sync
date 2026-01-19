# Test Plan: Автозагрузка WB stocks

## 1. Функциональные сценарии (happy path)
- Создание автозагрузки  
  - POST /api/wb/auto-sync/create {client_id, enabled:true, interval_hours:6, data_type:stocks}  
  - Ожид.: 200 OK, body содержит id; запись в `auto_sync_configs`; job в BullMQ (wait) с payload {client_id, data_type, interval_hours}.
- Просмотр статуса  
  - GET /api/wb/auto-sync/status?client_id=5  
  - Ожид.: 200 OK с полями enabled, interval_hours, last_sync_at, next_sync_at, status.
- Выполнение по расписанию  
  - interval_hours=1 (тест/стейдж: симуляция) → job: wait → active → completed.  
  - Ожид.: данные сохранены в schema `client_5.wb_stocks`; `updated_at` обновлён; запись лога / `last_sync_at`.
- Обновление настроек  
  - PUT /api/wb/auto-sync/update {client_id, interval_hours:12}  
  - Ожид.: 200 OK; `auto_sync_configs` обновлена; `next_sync_at` пересчитан.
- Отключение/включение  
  - PUT ... {enabled:false} → 200 OK; будущие job отменены/не планируются; настройки сохраняются. Включение создаёт новый job.
- Удаление  
  - DELETE /api/wb/auto-sync/delete?client_id=5 → 200/204; запись удалена; job удалён.

## 2. API тесты
- Валидация:  
  - Отсутствует `client_id` → 400  
  - `client_id <= 0` → 400  
  - `interval_hours < 1` или `> 24` → 400  
  - `data_type != "stocks"` → 400
- Авторизация/доступ:  
  - Без JWT → 401; невалидный/истёкший token → 401; token клиента A для client_id=B → 403.
- Дубли:  
  - Повторный POST для same client_id/data_type → 409 Conflict ("Auto-sync already exists").
- CRUD поведение:  
  - PUT/DELETE несуществующей конфигурации → 404.
- Ожидаемые коды: 200, 400, 401, 403, 404, 409, 500.

Примеры запросов:
```bash
# Create
curl -X POST http://host/api/wb/auto-sync/create \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{"client_id":5,"enabled":true,"interval_hours":6,"data_type":"stocks"}'

# Status
curl -X GET "http://host/api/wb/auto-sync/status?client_id=5" -H "Authorization: Bearer <token>"

# Update
curl -X PUT http://host/api/wb/auto-sync/update -H "Authorization: Bearer <token>" \
  -d '{"client_id":5,"interval_hours":12}'

# Delete
curl -X DELETE "http://host/api/wb/auto-sync/delete?client_id=5" -H "Authorization: Bearer <token>"
```

## 3. Очередь BullMQ — что проверить
- Job creation: после POST job присутствует в очереди `wb-auto-sync:wait` с payload {client_id, data_type, interval_hours} и связывается с config_id.
- Lifecycle: wait → active → completed; проверка timestamps и duration.
- Планирование: после success создаётся следующий job через `interval_hours` (next_sync_at).
- Retry: при ошибке WB API — 3 попытки с экспоненциальным backoff; затем job → failed и уведомление/лог. Проверить номера попыток и delays.
- Отключение: при `enabled=false` отмена будущих job.
- Конкурентность: не допускается параллельный запуск двух job одного клиента (no double runs).

## 4. База данных — проверки в PostgreSQL
- `auto_sync_configs`: запись создана/обновлена с полями (client_id, data_type, enabled, interval_hours, last_sync_at, next_sync_at, status, error_count). Уникальность (client_id,data_type)=1.
  - Проверка: SELECT * FROM auto_sync_configs WHERE client_id = 5;
- `wb_stocks` в schema клиента (`client_5`): данные вставлены/обновлены (UPDATE, не дубли), `updated_at` свежий.
  - Проверка: SET search_path TO client_5; SELECT COUNT(*) FROM wb_stocks WHERE updated_at > NOW() - INTERVAL '1 hour';
- Логи синхронизаций: start/end, count, errors.
- Ошибки БД: корректная обработка при отсутствии schema/table и при недоступности БД (retry → failed + уведомление).
- Индексы и производительность при больших объёмах.

## 5. Мультитенантность — как проверить изоляцию данных
- API: token клиента A не позволяет CRUD для client_id=B → 403. GET /status чужого client_id → 403.
- DB: данные client_5 в `client_5` schema, не видны в `client_10`. Проверить: SET search_path TO client_5; SELECT COUNT(*) FROM wb_stocks; затем для client_10 — разные результаты.
- Queue: jobs разных клиентов независимы; ошибка у одного не влияет на другой.
- Убедиться, что запросы используют корректный `search_path` или RLS для изоляции.

## 6. Негативные сценарии — что может пойти не так
- WB API: 503/502/timeout → retry 3× → failed; 429 → backoff/Retry-After; невалидный JSON → parse error → retry/log.
- DB: connection timeout / pool exhausted / disk full / missing schema → retry/fail + уведомление; constraint violations — логи/корректная обработка.
- Redis/BullMQ: Redis down → создание/job ошибки; worker crash → jobs в wait; job persistence после рестарта.
- Конкуренция: simultaneous POST → 409; PUT+DELETE race → консистентность.
- Логирование/уведомления: при критических ошибках — лог с client_id, job_id, attempt, stack/message; уведомление после финального failure.

## 7. Граничные случаи (edge cases)
- Большой объём данных (100k+): обработка batch-ами, не превышает memory/timeout; проверять duration, пагинацию, batch-size.
- Job дольше интервала: interval=1h, job 1.5h → следующая задача не стартует, нет дублей.
- Изменение interval во время выполнения: текущая задача завершается; следующий job по новому interval.
- Удаление/отключение во время выполнения: текущая задача завершаетcя; новые не планируются.
- Минимум/максимум interval_hours = 1/24 валидны; дробные/нечисловые → 400.
- Часовые пояса/DST: система использует UTC или корректно рассчитывает next_sync_at.

## 8. Приоритизация
- P0 (CRITICAL): создание автозагрузки; job выполняется по расписанию; данные сохраняются в правильной schema; мультитенантность (нет утечек); API auth; базовая валидация; retry при ошибках WB; отключение автозагрузки.
- P1 (HIGH): полная API валидация и коды ошибок; жизненный цикл очереди; логирование; обработка невалидного ответа WB; БД-ошибки.
- P2 (MEDIUM): большие данные/производительность; конкурентные операции; edge-cases (job > interval, изменение interval).
- P3 (LOW): DST/таймзоны; UI/уведомления; мониторинг/метрики.