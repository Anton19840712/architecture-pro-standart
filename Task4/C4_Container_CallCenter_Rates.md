# C4 Container Diagram: Передача ставок в кол-центры

## Описание

Эта диаграмма детализирует контейнеры, участвующие в передаче ставок в кол-центры (фокус на новых компонентах).

---

## Новые контейнеры / Изменения

### Rate Management Service - новые модули

#### 1. **API Integration Module** [Component внутри Rate Management Service] (НОВЫЙ)
- **Технологии:** Java Spring Boot, Spring Web
- **Ответственность:**
  - REST API для внешних систем
  - Endpoint: `GET /api/v1/rates` (с кэшированием)
  - Endpoint: `POST /api/v1/webhooks/subscribe`
  - Webhook-уведомления при обновлении ставок
  - API Key аутентификация
  - Rate limiting (100 req/min per API Key)
- **Связи:**
  - ← API Gateway: "Запросы от системы кол-центра"
  - → Redis Cache: "Чтение кэшированных ставок"
  - → Rates DB: "Чтение ставок (если не в кэше)"
  - → Система кол-центра (Webhook): "POST-уведомления"

#### 2. **File Export Module** [Component внутри Rate Management Service] (НОВЫЙ)
- **Технологии:** Java Spring Boot, Spring Batch, Spring Integration SFTP, Apache POI
- **Ответственность:**
  - Экспорт ставок в CSV/Excel файлы
  - SFTP-клиент для загрузки файлов
  - Scheduler (triggered events + cron fallback)
  - Retry-логика (5 попыток, exponential backoff)
  - Логирование результатов экспорта
  - Мониторинг (Prometheus metrics)
- **Связи:**
  - → Rates DB: "Чтение ставок для экспорта"
  - → Export Logs DB: "Запись логов операций"
  - → SFTP-сервер партнёра: "Загрузка CSV-файлов"
  - → Prometheus: "Метрики экспортов"

#### 3. **Export Logs DB** [Database Table внутри Rates DB] (НОВЫЙ)
- **Технологии:** PostgreSQL (таблица в Rates DB)
- **Схема:**
  - `export_logs`:
    - `id`, `export_date`, `file_name`, `file_size`, `destination` (partner/callcenter)
    - `status` (success/failed), `error_message`, `retry_count`, `completed_at`
- **Ответственность:**
  - Хранение истории экспортов
  - Трейсинг неудачных попыток

---

### Система кол-центра - новые компоненты

#### 4. **Rates Integration Adapter** [Container: Spring Boot Application] (НОВЫЙ)
- **Технологии:** Java 11, Spring Boot, Spring Web, Scheduled Tasks
- **Ответственность:**
  - Webhook-receiver для уведомлений от Rate Management Service
  - HTTP-клиент для вызова API Rate Management Service
  - Scheduled task (fallback): обновление каждые 60 минут
  - Сохранение ставок в БД системы кол-центра
  - Retry-логика (3 попытки)
  - Логирование и мониторинг
- **Порты:** 8082 (HTTP internal, webhook endpoint)
- **Связи:**
  - ← Rate Management Service (Webhook): "POST /webhooks/rates-updated"
  - → Rate Management Service (HTTP API): "GET /api/v1/rates"
  - → Call Center DB (deposit_rates_cache): "Сохранение ставок"
  - → Prometheus: "Метрики обновлений"

#### 5. **Deposit Rates Cache** [Database Table внутри Call Center DB] (НОВЫЙ)
- **Технологии:** PostgreSQL (таблица в Call Center DB)
- **Схема:**
  - `deposit_rates_cache`:
    - `product_id`, `product_name`, `rate`, `term_months`, `min_amount`, `max_amount`
    - `currency`, `updated_at`, `last_sync_at`
- **Ответственность:**
  - Локальное хранение актуальных ставок
  - Быстрый доступ для операторов

#### 6. **Deposits UI for Operators** [Web Page внутри Call Center Web App] (НОВЫЙ)
- **Технологии:** React.js, TypeScript
- **Ответственность:**
  - Интерфейс просмотра депозитов и ставок
  - Таблица продуктов с фильтрацией (срок, сумма, тип)
  - Индикатор актуальности ставок:
    - Зелёный: обновлено < 24 часов
    - Жёлтый: обновлено 24-48 часов (предупреждение)
    - Красный: обновлено > 48 часов (ошибка, эскалация)
  - Timestamp последнего обновления
- **Связи:**
  - → Call Center DB (deposit_rates_cache): "Чтение ставок"
  - ← Оператор кол-центра: "Просматривает ставки"

---

### Партнёрский кол-центр (внешняя система)

#### 7. **SFTP Server** [Container: SFTP Server] (НОВЫЙ)
- **Технологии:** OpenSSH SFTP Server
- **Адрес:** `sftp.partner-callcenter.com:22`
- **Ответственность:**
  - Приём файлов со ставками от Rate Management Service
  - SSH-key аутентификация
  - Хранение файлов в `/incoming/rates/`
- **Связи:**
  - ← Rate Management Service (File Export Module): "Загрузка CSV-файлов"
  - → Partner Call Center System: "Файлы доступны для импорта"

#### 8. **Partner Call Center System** [Container: External Application] (НОВЫЙ)
- **Технологии:** Proprietary (система партнёра, детали неизвестны)
- **Ответственность:**
  - Polling SFTP-сервера (каждые 15 минут)
  - Импорт CSV-файлов в свою БД
  - Отображение ставок операторам
  - Предупреждение при устаревших ставках (> 48 часов)
- **Связи:**
  - ← SFTP Server: "Polling и загрузка файлов"
  - ← Оператор партнёрского кол-центра: "Просматривает ставки"

---

## Детализированные потоки данных

### Поток 1: Обновление ставок в системе кол-центра банка (webhook + API)

```
1. Сотрудник бэк-офиса кредитов → Rate Management Service UI
   - Обновляет ставки

2. Rate Management Service → Rates DB
   - UPDATE deposit_rates

3. Rate Management Service → Redis Cache
   - INVALIDATE и SET новых ставок

4. Rate Management Service (API Integration Module) → Rates Integration Adapter
   - POST /webhooks/rates-updated
   - Body: {event: "rates.updated", timestamp: "...", url: "/api/v1/rates"}

5. Rates Integration Adapter получает webhook
   - Валидация подписи (HMAC)

6. Rates Integration Adapter → API Gateway → Rate Management Service
   - GET /api/v1/rates
   - Header: X-API-Key: <key>

7. Rate Management Service (API Integration Module) → Redis Cache
   - GET rates:* (если в кэше, TTL 15 мин)

8. Rate Management Service → Rates Integration Adapter
   - Response 200 OK
   - Body: [{"productId": "DEP001", "productName": "...", "rate": 8.5, ...}, ...]

9. Rates Integration Adapter → Call Center DB
   - DELETE FROM deposit_rates_cache WHERE 1=1
   - INSERT INTO deposit_rates_cache VALUES (...)
   - UPDATE last_sync_at = NOW()

10. Оператор кол-центра → Deposits UI for Operators
    - Открывает страницу "Депозиты"

11. Deposits UI → Call Center DB
    - SELECT * FROM deposit_rates_cache ORDER BY rate DESC

12. Deposits UI → Оператор
    - Отображает таблицу с актуальными ставками
    - Индикатор: Зелёный (обновлено 2 минуты назад)
```

**Fallback (если webhook не сработал):**
```
Scheduled Task в Rates Integration Adapter
  → Каждые 60 минут: вызов API Rate Management Service
  → Обновление deposit_rates_cache
```

---

### Поток 2: Передача ставок в партнёрский кол-центр (SFTP)

```
1. Сотрудник бэк-офиса кредитов → Rate Management Service UI
   - Обновляет ставки

2. Rate Management Service → Rates DB
   - UPDATE deposit_rates

3. Rate Management Service (событие "rates.updated") → File Export Module
   - Triggered Spring Batch Job

4. File Export Module → Rates DB
   - SELECT * FROM deposit_rates WHERE active = true

5. File Export Module генерирует CSV-файл
   - Файл: deposit_rates_20251124.csv
   - Содержимое:
     ProductID,ProductName,Rate,TermMonths,MinAmount,MaxAmount,Currency,UpdatedAt
     DEP001,Классический депозит,8.5,12,10000,5000000,RUB,2025-11-24T09:00:00Z

6. File Export Module → Export Logs DB
   - INSERT INTO export_logs (export_date, file_name, status='in_progress', destination='partner')

7. File Export Module (SFTP Client) → SFTP Server (партнёр)
   - Подключение: sftp.partner-callcenter.com:22
   - Аутентификация: SSH private key
   - Команда: PUT deposit_rates_20251124.csv /incoming/rates/

8a. Успех:
   - File Export Module → Export Logs DB
     - UPDATE export_logs SET status='success', completed_at=NOW()
   - File Export Module → Prometheus
     - Метрика: export_success_total{destination="partner"}++

8b. Неудача (SFTP недоступен):
   - File Export Module → Export Logs DB
     - UPDATE export_logs SET status='failed', error_message='Connection timeout', retry_count++
   - File Export Module ждёт 30 минут, повторяет попытку (до 5 раз)
   - После 5 неудач → Alert в Slack: "SFTP экспорт в партнёрский кол-центр failed"

9. Партнёрский кол-центр (Scheduled Task)
   - Каждые 15 минут: polling SFTP-сервера
   - Команда: LIST /incoming/rates/
   - Обнаруживает новый файл: deposit_rates_20251124.csv

10. Partner Call Center System → SFTP Server
    - GET /incoming/rates/deposit_rates_20251124.csv

11. Partner Call Center System импортирует CSV
    - Парсинг CSV
    - UPDATE/INSERT в свою БД

12. Оператор партнёрского кол-центра
    - Открывает интерфейс со ставками
    - Видит обновлённые ставки
```

---

## Технические детали

### API Integration Module - Endpoints:

**GET /api/v1/rates**
- **Auth:** API Key (header: `X-API-Key`)
- **Query params:**
  - `productType` (optional): deposit, savings
  - `term` (optional): фильтр по сроку в месяцах
  - `minAmount`, `maxAmount` (optional): фильтр по сумме
- **Response:**
```json
[
  {
    "productId": "DEP001",
    "productName": "Классический депозит",
    "rate": 8.5,
    "termMonths": 12,
    "minAmount": 10000,
    "maxAmount": 5000000,
    "currency": "RUB",
    "updatedAt": "2025-11-24T09:00:00Z"
  }
]
```
- **Caching:** Redis, TTL 15 минут
- **Rate Limiting:** 100 requests/minute per API Key

**POST /api/v1/webhooks/subscribe**
- **Auth:** API Key
- **Body:**
```json
{
  "url": "https://callcenter-system.bank.com/webhooks/rates-updated",
  "secret": "webhook-secret-key-for-hmac"
}
```
- **Response:** 201 Created

**Webhook Notification (POST to subscribed URL):**
```json
{
  "event": "rates.updated",
  "timestamp": "2025-11-24T09:00:00Z",
  "ratesUrl": "/api/v1/rates"
}
```
- **Header:** `X-Signature: HMAC-SHA256(body, secret)`

---

### File Export Module - CSV Format:

```csv
ProductID,ProductName,Rate,TermMonths,MinAmount,MaxAmount,Currency,UpdatedAt
DEP001,Классический депозит,8.5,12,10000,5000000,RUB,2025-11-24T09:00:00Z
DEP002,Накопительный счёт,7.0,0,1000,,RUB,2025-11-24T09:00:00Z
DEP003,Пенсионный депозит,9.0,24,50000,10000000,RUB,2025-11-24T09:00:00Z
```

**Файл:** `deposit_rates_YYYYMMDD.csv`
- **Кодировка:** UTF-8
- **Разделитель:** `,`
- **Decimal separator:** `.`
- **Пустые значения:** пустая строка (не NULL)

---

### SFTP Configuration:

**Rate Management Service (SFTP Client):**
```yaml
sftp:
  host: sftp.partner-callcenter.com
  port: 22
  username: bank-standart-user
  privateKey: ${SFTP_PRIVATE_KEY} # from Vault
  remoteDirectory: /incoming/rates/
  retryPolicy:
    maxAttempts: 5
    backoff: [30m, 1h, 2h, 4h, 8h]
```

**Partner SFTP Server (provided by partner):**
- IP whitelist: IP-адреса серверов Rate Management Service
- SSH Key fingerprint: проверка при первом подключении

---

### Мониторинг

**Prometheus Metrics:**

Rate Management Service:
- `rate_api_requests_total{endpoint, status}` - количество API-запросов
- `rate_api_latency_seconds{endpoint}` - latency API
- `rate_export_total{destination, status}` - количество экспортов (success/failed)
- `rate_export_duration_seconds{destination}` - длительность экспорта
- `rate_export_file_size_bytes{destination}` - размер файлов
- `rate_sftp_retry_count{destination}` - количество retry

Система кол-центра (Rates Integration Adapter):
- `rates_sync_total{status}` - количество синхронизаций (success/failed)
- `rates_sync_latency_seconds` - latency синхронизации
- `rates_last_sync_timestamp` - timestamp последней успешной синхронизации
- `rates_cache_age_seconds` - возраст кэша ставок

**Grafana Dashboards:**
- "Rates Distribution" - общая картина распределения ставок
- "Call Center Rates Integration" - метрики интеграции с кол-центрами

**Alerts (Prometheus Alertmanager → Slack):**
1. `RatesNotUpdated`: ставки не обновлены > 25 часов
2. `CallCenterAPIFailure`: 3+ неудачных API-вызова от системы кол-центра
3. `PartnerSFTPFailure`: 5 неудачных попыток SFTP
4. `PartnerFileNotFetched`: файл не забран партнёром > 48 часов
5. `RatesCacheStale`: кэш ставок устарел > 2 часов

---

## Безопасность

### API Integration Module:
- **API Key Management:**
  - Генерация ключей: admin UI в Rate Management Service
  - Хранение: Vault или Kubernetes Secrets
  - Ротация: каждые 90 дней
  - Отдельные ключи для каждого потребителя (система кол-центра, future systems)
- **Rate Limiting:** Kong API Gateway + Spring Boot rate limiter
- **Webhook Security:**
  - HMAC-SHA256 signature для валидации
  - HTTPS only
  - IP whitelist для webhook endpoints

### File Export Module:
- **SFTP Security:**
  - SSH private key authentication (not password)
  - Key хранится в Vault
  - IP whitelist на SFTP-сервере партнёра
  - Опционально: шифрование файлов (если партнёр поддерживает PGP)
- **Data Security:**
  - Файлы содержат только справочные данные (ставки, не персональные данные)
  - Логирование всех операций загрузки

---

## Примечания для визуализации

**Цветовая схема:**
- **Синий** - Новые компоненты в Rate Management Service (API Integration, File Export)
- **Голубой** - Новые компоненты в системе кол-центра (Rates Integration Adapter, UI)
- **Оранжевый** - Внешние компоненты партнёра (SFTP Server, Partner System)
- **Цилиндры** - Базы данных / таблицы

**Группировка:**
- "Rate Management Service" - сгруппировать API Integration Module, File Export Module
- "Система кол-центра" - сгруппировать Rates Integration Adapter, Deposits UI, БД
- "Партнёрский кол-центр" - сгруппировать SFTP Server, Partner System

**Потоки (стрелки):**
- Пунктирные стрелки для асинхронных/периодических операций (SFTP, scheduled tasks)
- Сплошные стрелки для синхронных API-вызовов
- Цвет стрелок:
  - Зелёный: успешные операции
  - Красный: retry/fallback пути

---

## Инструкция для draw.io:

1. Создать файл `C4_Container_CallCenter_Rates.drawio`
2. Взять за основу диаграмму из Task 3
3. Расширить Rate Management Service, добавить внутри:
   - API Integration Module (компонент, синий)
   - File Export Module (компонент, синий)
4. Добавить справа группу "Система кол-центра":
   - Rates Integration Adapter (контейнер, голубой)
   - Deposits UI for Operators (компонент, голубой)
   - Таблица deposit_rates_cache (БД, цилиндр)
5. Добавить справа внизу группу "Партнёрский кол-центр":
   - SFTP Server (контейнер, оранжевый)
   - Partner Call Center System (контейнер, оранжевый)
6. Соединить стрелками согласно потокам выше
7. Добавить легенду с протоколами и цветами
