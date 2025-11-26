# C4 Container Diagram: Онлайн подача заявок на кредит

## Описание

Эта диаграмма детализирует внутреннюю структуру "Системы онлайн кредитования", а также показывает изменения в системе скоринга и Кредитном конвейере.

---

## Контейнеры системы "Онлайн кредитование"

### 1. **Credit Application Service** [Container: Spring Boot Application] (НОВЫЙ)
- **Технологии:** Java 11, Spring Boot 2.7, PostgreSQL 14
- **Ответственность:**
  - Управление жизненным циклом кредитных заявок
  - Валидация заявок
  - Управление статусами
  - Интеграция с системой скоринга (REST API)
  - Публикация заявок в Kafka
  - REST API для получения статусов и предодобренных предложений
  - Идемпотентность операций
  - Аудит всех операций
- **База данных:** Credit Applications DB (PostgreSQL)
- **Порты:** 8083 (HTTP internal)
- **Связи:**
  - ← API Gateway (HTTP): "Принимает заявки"
  - → Credit Applications DB: "Сохраняет заявки и статусы"
  - → Kafka: "Публикует заявки в топик 'credit-applications'"
  - → Система скоринга (REST API): "Запрашивает instant scoring для новых клиентов"
  - → Система скоринга (REST API): "Запрашивает предодобренные предложения"
  - ← Кредитный конвейер Integration Adapter: "Получает обновления статусов"

### 2. **Credit Applications DB** [Container: PostgreSQL Database] (НОВАЯ)
- **Технологии:** PostgreSQL 14
- **Схема:**
  - `credit_applications` - заявки на кредиты
  - `application_status_history` - история изменений статусов
  - `audit_log` - аудит операций
- **Ответственность:**
  - Хранение заявок
  - Хранение истории статусов
  - Аудит
- **Связи:**
  - ← Credit Application Service: "Чтение/запись заявок"

---

## Детализация системы скоринга (изменения)

### 3. **Scoring Service** [Container: Flask Application] (ОБНОВЛЕНИЕ)
- **Технологии:** Python 3.9, Flask, PostgreSQL 14
- **Ответственность:**
  - **Существующее:**
    - Расчёт скоринга для заявок из Кредитного конвейера
  - **НОВОЕ:**
    - Instant scoring для новых клиентов (POST /api/v1/scoring/instant)
    - API для получения предодобренных предложений (GET /api/v1/scoring/pre-approved/{clientId})
- **Связи:**
  - ← Credit Application Service (REST API): "Запросы instant scoring и предодобренных предложений"
  - ← Кредитный конвейер (REST API): "Запросы скоринга для заявок"
  - → БКИ (REST API): "Запрашивает кредитные истории"
  - → Scoring DB: "Чтение/запись данных"
  - ← АБС: "Получает данные клиентов раз в сутки"

### 4. **Pre-Approved Offers Batch Job** [Container: Python Script/Scheduler] (НОВЫЙ)
- **Технологии:** Python 3.9, Cron
- **Ответственность:**
  - Batch-процесс предрасчёта предодобренных предложений
  - Запуск ежедневно в 02:00 (ночью)
  - Обработка всех клиентов банка
  - Расчёт скоринга на основе АБС + БКИ (кэш)
  - Сохранение предложений в pre_approved_offers
- **Scheduler:** Cron (crontab: 0 2 * * *)
- **Связи:**
  - → Scoring DB: "Читает данные клиентов (из АБС, синхронизация раз в сутки)"
  - → Scoring DB: "Читает кэш БКИ (< 30 дней)"
  - → Scoring DB: "Записывает предодобренные предложения в pre_approved_offers"
  - → Prometheus: "Отправляет метрики (количество обработанных, длительность)"

### 5. **Scoring DB** [Container: PostgreSQL Database] (ОБНОВЛЕНИЕ)
- **Технологии:** PostgreSQL 14
- **Схема:**
  - **Существующие:**
    - `scoring_results` - результаты скоринга
    - `client_data_cache` - кэш данных клиентов из АБС
    - `bki_cache` - кэш данных БКИ (30 дней)
  - **НОВОЕ:**
    - `pre_approved_offers` - предодобренные кредитные предложения
      - Колонки: client_id, max_amount, rate, term_months, score, valid_until, created_at
- **Ответственность:**
  - Хранение результатов скоринга
  - Хранение предодобренных предложений
  - Кэширование данных из АБС и БКИ
- **Связи:**
  - ← Scoring Service: "Чтение/запись"
  - ← Pre-Approved Offers Batch Job: "Чтение/запись"

---

## Детализация Кредитного конвейера (изменения)

### 6. **Credit Conveyor BPM** [Container: Camunda BPM Application] (ОБНОВЛЕНИЕ)
- **Технологии:** Camunda BPM, Java, Oracle
- **Ответственность:**
  - **Существующее:**
    - Workflow обработки кредитных заявок
    - Интерфейс для сотрудников бэк-офиса кредитов
  - **НОВОЕ:**
    - Приоритизация заявок (pre_approved: true → высокий приоритет)
    - Индикатор приоритета в UI для операторов
    - Автоматический вызов скоринга для всех заявок (REST API)
- **Связи:**
  - ← Кредитный конвейер Integration Adapter: "Получает заявки"
  - → Scoring Service (REST API): "Запрашивает скоринг"
  - → Кредитный конвейер Integration Adapter: "Отправляет статусы заявок"
  - → АБС: "Зачисление средств при одобрении"
  - ← Сотрудник бэк-офиса кредитов: "Обрабатывает заявки"

### 7. **Credit Conveyor Integration Adapter** [Container: Spring Boot Application] (НОВЫЙ)
- **Технологии:** Java 11, Spring Boot, JDBC (Oracle), Kafka Client
- **Ответственность:**
  - Kafka Consumer: чтение заявок из топика 'credit-applications'
  - Запись заявок в БД Кредитного конвейера (Oracle)
  - Приоритизация: заявки с pre_approved: true → высокий приоритет
  - Kafka Producer: публикация статусов в топик 'credit-application-status'
  - Изоляция Кредитного конвейера от прямых вызовов
- **Развёртывание:** На серверах Кредитного конвейера
- **Порты:** 8084 (HTTP internal для healthcheck)
- **Связи:**
  - ← Kafka: "Читает заявки из топика 'credit-applications'"
  - → Credit Conveyor Oracle DB: "Записывает заявки"
  - ← Credit Conveyor BPM: "Читает статусы заявок (polling)"
  - → Kafka: "Публикует статусы в топик 'credit-application-status'"

### 8. **Credit Conveyor Oracle DB** [Container: Oracle Database] (ОБНОВЛЕНИЕ)
- **Технологии:** Oracle 19c
- **Схема:**
  - **Существующие:**
    - `credit_applications` - кредитные заявки
    - `scoring_results` - результаты скоринга
    - `credit_contracts` - кредитные договоры
  - **НОВОЕ:**
    - `credit_applications_online` - новая таблица для онлайн-заявок
      - Колонки: ..., pre_approved (boolean), priority (высокий/обычный), source (online/legacy)
- **Ответственность:**
  - Хранение кредитных заявок
  - Хранение договоров и платежей
- **Связи:**
  - ← Credit Conveyor Integration Adapter: "Запись заявок"
  - ← Credit Conveyor BPM: "Чтение/запись всех данных"

---

## Расширение Kafka Cluster

### 9. **Kafka Cluster** [Container: Apache Kafka] (ОБНОВЛЕНИЕ)
- **Технологии:** Apache Kafka 3.4
- **Топики:**
  - **Существующие (Task 3):**
    - `deposit-applications`
    - `deposit-application-status`
  - **НОВЫЕ:**
    - `credit-applications` - заявки на кредиты из всех каналов
    - `credit-application-status` - обновления статусов от Кредитного конвейера
- **Конфигурация новых топиков:**
  - Партиции: 10 (для параллельной обработки)
  - Replication factor: 2
  - Retention: 7 дней (credit-applications), 30 дней (credit-application-status)
  - Compression: snappy
- **Связи:**
  - ← Credit Application Service: "Публикация заявок"
  - → Credit Conveyor Integration Adapter: "Чтение заявок"
  - ← Credit Conveyor Integration Adapter: "Публикация статусов"
  - → Credit Application Service: "Чтение статусов"

---

## Доработки в интернет-банке

### 10. **Internet Banking Web App** [Container: ASP.NET MVC Application] (ОБНОВЛЕНИЕ)
- **Существующее:** (см. Task 3)
- **НОВЫЕ функции:**
  - Раздел "Кредиты":
    - Отображение предодобренных кредитных предложений
    - Список доступных кредитных продуктов
  - Формы подачи заявок на кредит:
    - Для предодобренных предложений (упрощённая)
    - Для обычных кредитов (полная)
  - Раздел "Мои заявки на кредиты" с отображением статусов
  - Polling статусов заявок (каждые 30 сек)
- **Новые связи:**
  - → API Gateway → Credit Application Service:
    - GET /api/v1/pre-approved-offers/{clientId}
    - POST /api/v1/credit-applications
    - GET /api/v1/credit-applications/{id}
    - GET /api/v1/credit-applications (список заявок клиента)

---

## Доработки на сайте

### 11. **Website** [Container: PHP + React.js Application] (ОБНОВЛЕНИЕ)
- **Существующее:** (см. Task 3)
- **НОВЫЕ функции:**
  - Страница "Кредиты":
    - Каталог кредитных продуктов
  - Форма подачи заявки для новых клиентов:
    - ФИО, телефон, сумма, срок
    - Опционально: паспортные данные (серия, номер, дата выдачи)
  - Мгновенное решение (если паспортные данные указаны):
    - Вызов instant scoring через Credit Application Service
    - Отображение результата: "Одобрено до X рублей" или "Не одобрено"
  - Страница с результатом и СМС-уведомление
- **Новые связи:**
  - → API Gateway → Credit Application Service:
    - POST /api/v1/credit-applications (с passport_data)

---

## Ключевые потоки данных (детализированные)

### Поток 1: Предрасчёт предодобренных предложений (batch, ночью)

```
02:00 - Cron запускает Pre-Approved Offers Batch Job

Pre-Approved Offers Batch Job → Scoring DB
  - SELECT * FROM client_data_cache (данные из АБС, синхронизация раз в сутки)
  - SELECT * FROM bki_cache WHERE updated_at > NOW() - INTERVAL '30 days'

FOR EACH клиент:
  Pre-Approved Offers Batch Job:
    - Рассчитывает скоринг (Python ML model)
    - IF скоринг > порог:
      - Рассчитывает max_amount, rate
      - INSERT/UPDATE pre_approved_offers (client_id, max_amount, rate, valid_until = NOW() + 30 days)
    - ELSE:
      - DELETE FROM pre_approved_offers WHERE client_id = ?

Pre-Approved Offers Batch Job → Prometheus:
  - Метрики: processed_clients_total, approved_offers_total, batch_duration_seconds

06:00 - Batch Job завершён (или alert если не завершён)
```

---

### Поток 2: Просмотр и подача заявки по предодобренному предложению (интернет-банк)

```
1. Клиент → Internet Banking Web App (HTTPS)
   - Открывает раздел "Кредиты"

2. Internet Banking Web App → API Gateway (HTTPS)
   - GET /api/v1/pre-approved-offers/{clientId}
   - Header: Authorization: Bearer {JWT}

3. API Gateway → Credit Application Service (HTTP)
   - Маршрутизация, JWT-валидация

4. Credit Application Service → Scoring Service (REST API)
   - GET /api/v1/scoring/pre-approved/{clientId}

5. Scoring Service → Scoring DB
   - SELECT * FROM pre_approved_offers WHERE client_id = ? AND valid_until > NOW()

6. Scoring Service → Credit Application Service
   - Response: {max_amount: 500000, rate: 12.5, term_months: 36}

7. Credit Application Service → Internet Banking Web App
   - Response: предодобренное предложение

8. Клиент видит баннер: "Одобрено до 500,000 руб под 12.5% годовых"

--- Клиент заполняет форму заявки ---

9. Internet Banking Web App → API Gateway → Credit Application Service
   - POST /api/v1/credit-applications
   - Body: {clientId, productId, amount: 300000, term: 24, accountId, pre_approved: true}

10. Credit Application Service → Credit Applications DB
    - INSERT INTO credit_applications (status: 'CREATED', pre_approved: true, source: 'INTERNET_BANKING')

11. Credit Application Service → Kafka
    - Публикация в топик 'credit-applications'
    - Message: {applicationId, clientId, amount, term, pre_approved: true, ...}

12. Kafka → Credit Conveyor Integration Adapter
    - Чтение сообщения (consumer group: credit-conveyor-consumers)

13. Credit Conveyor Integration Adapter → Credit Conveyor Oracle DB
    - INSERT INTO credit_applications_online (priority: 'HIGH' т.к. pre_approved: true)

14. Credit Conveyor BPM (workflow engine)
    - Автоматически создаёт задачу для сотрудника (высокий приоритет)

15. Сотрудник бэк-офиса кредитов → Credit Conveyor BPM
    - Видит заявку с индикатором "Предодобренная"
    - Открывает заявку

16. Credit Conveyor BPM → Scoring Service (REST API)
    - POST /api/v1/scoring/calculate
    - Body: {clientId, amount, term}
    - (Подтверждение актуальности скоринга)

17. Scoring Service → Credit Conveyor BPM
    - Response: {score: 750, decision: 'APPROVED'}

18. Сотрудник проверяет данные и одобряет заявку

19. Credit Conveyor BPM → Credit Conveyor Oracle DB
    - UPDATE credit_applications_online SET status = 'APPROVED'

20. Credit Conveyor Integration Adapter (polling или trigger)
    - Обнаруживает изменение статуса

21. Credit Conveyor Integration Adapter → Kafka
    - Публикация в топик 'credit-application-status'
    - Message: {applicationId, status: 'APPROVED', timestamp}

22. Kafka → Credit Application Service
    - Чтение обновления статуса

23. Credit Application Service → Credit Applications DB
    - UPDATE credit_applications SET status = 'APPROVED'

24. Credit Conveyor BPM → АБС (интеграция)
    - Команда на зачисление кредитных средств на счёт клиента

25. АБС → СМС-шлюз
    - Отправка СМС: "Кредит одобрен, средства зачислены"

26. СМС-шлюз → Телеком-оператор → Клиент (СМС)

27. Internet Banking Web App (polling)
    - GET /api/v1/credit-applications/{applicationId} (каждые 30 сек)
    - Response: {status: 'APPROVED'}

28. Клиент видит обновлённый статус: "Одобрено"
```

---

### Поток 3: Новый клиент с сайта - мгновенное решение

```
1. Клиент → Website (HTTPS)
   - Открывает страницу "Кредиты"
   - Заполняет форму: ФИО, телефон, сумма: 200000, срок: 12, паспорт: 4512 123456

2. Website → API Gateway (HTTPS)
   - POST /api/v1/credit-applications
   - Body: {fullName, phone, amount, term, passport: {series, number, issueDate}, client_type: 'NEW'}

3. API Gateway → Credit Application Service (HTTP)
   - Маршрутизация

4. Credit Application Service:
   - Валидация паспортных данных (формат, корректность)
   - Создание заявки в БД (status: 'CREATED', source: 'WEBSITE', client_type: 'NEW')

5. Credit Application Service → Scoring Service (REST API)
   - POST /api/v1/scoring/instant
   - Body: {passport, amount, term}

6. Scoring Service → БКИ (REST API, внешний)
   - POST /api/credit-history
   - Body: {passport_series, passport_number}

7. БКИ → Scoring Service
   - Response: кредитная история клиента (займы, платежи, просрочки)

8. Scoring Service:
   - Рассчитывает скоринг на основе БКИ (т.к. клиент новый, нет данных в АБС)
   - IF скоринг > порог:
     - decision: 'APPROVED', max_amount: 250000
   - ELSE:
     - decision: 'REJECTED'

9. Scoring Service → Credit Application Service
   - Response: {decision: 'APPROVED', max_amount: 250000, score: 720}

10. Credit Application Service → Credit Applications DB
    - UPDATE credit_applications SET status = 'APPROVED', max_amount = 250000

11. Credit Application Service → Website
    - Response: {status: 'APPROVED', max_amount: 250000}

12. Website → Клиент
    - Отображает: "Поздравляем! Кредит одобрен до 250,000 руб"
    - "Приходите в отделение для получения кредита с паспортом"

13. Credit Application Service → СМС-шлюз
    - Отправка СМС с адресом ближайшего отделения

14. Клиент приходит в отделение

15. Менеджер отделения → АБС (Desktop Client)
    - Поиск заявки: по ФИО + телефону

16. АБС → Credit Applications DB (через интеграцию)
    - SELECT * FROM credit_applications WHERE full_name = ? AND phone = ?

17. Менеджер находит заявку, идентифицирует клиента (проверка паспорта)

18. Менеджер → АБС
    - Продолжает работу с заявкой (status: 'IN_PROGRESS')
    - Открывает счёт для нового клиента
    - Создаёт кредитный договор

19. АБС → Зачисление средств на счёт

20. АБС → СМС-шлюз → Клиент (СМС)
    - "Кредит выдан, средства на счёте"
```

---

## Технические детали

### Credit Application Service - Endpoints:

**GET /api/v1/pre-approved-offers/{clientId}**
- **Auth:** JWT (интернет-банк)
- **Response:**
```json
{
  "clientId": "12345",
  "maxAmount": 500000,
  "rate": 12.5,
  "termMonths": 36,
  "validUntil": "2025-12-24T00:00:00Z"
}
```

**POST /api/v1/credit-applications**
- **Auth:** JWT (интернет-банк) или API Key (сайт)
- **Body:**
```json
{
  "clientId": "12345", // если существующий клиент
  "fullName": "Иванов Иван Иванович", // если новый клиент
  "phone": "+79991234567",
  "productId": "CR001",
  "amount": 300000,
  "termMonths": 24,
  "accountId": "40817...", // для зачисления (если есть)
  "passport": { // опционально, для новых клиентов
    "series": "4512",
    "number": "123456",
    "issueDate": "2015-05-10"
  },
  "preApproved": true // флаг предодобренной заявки
}
```
- **Response:**
```json
{
  "applicationId": "APP-2025-001234",
  "status": "CREATED", // или "APPROVED"/"REJECTED" для instant scoring
  "message": "Заявка принята. Решение в течение рабочего дня",
  "maxAmount": 250000 // если instant scoring
}
```

**GET /api/v1/credit-applications/{id}**
- **Auth:** JWT
- **Response:**
```json
{
  "applicationId": "APP-2025-001234",
  "status": "APPROVED",
  "amount": 300000,
  "termMonths": 24,
  "rate": 12.5,
  "monthlyPayment": 14123,
  "createdAt": "2025-11-24T10:00:00Z",
  "updatedAt": "2025-11-24T14:30:00Z"
}
```

---

### Scoring Service - New Endpoints:

**POST /api/v1/scoring/instant**
- **Auth:** API Key (Credit Application Service)
- **Body:**
```json
{
  "passport": {"series": "4512", "number": "123456"},
  "amount": 200000,
  "termMonths": 12
}
```
- **Response:**
```json
{
  "decision": "APPROVED",
  "score": 720,
  "maxAmount": 250000,
  "rate": 14.5
}
```

**GET /api/v1/scoring/pre-approved/{clientId}**
- **Auth:** API Key
- **Response:**
```json
{
  "clientId": "12345",
  "maxAmount": 500000,
  "rate": 12.5,
  "termMonths": 36,
  "score": 780,
  "validUntil": "2025-12-24T00:00:00Z"
}
```

---

## Мониторинг и Метрики

### Prometheus Metrics:

**Credit Application Service:**
- `credit_applications_total{source, pre_approved, status}` - количество заявок
- `credit_application_latency_seconds` - latency API
- `credit_instant_scoring_total{status}` - количество instant scoring (success/failed)
- `credit_instant_scoring_duration_seconds` - длительность instant scoring

**Scoring Service (новые метрики):**
- `scoring_instant_requests_total{status}` - instant scoring запросы
- `scoring_instant_latency_seconds` - latency instant scoring
- `scoring_bki_requests_total{status}` - запросы к БКИ
- `scoring_bki_latency_seconds` - latency БКИ
- `scoring_batch_duration_seconds` - длительность batch-процесса
- `scoring_batch_processed_clients_total` - количество обработанных клиентов
- `scoring_batch_approved_offers_total` - количество предодобренных предложений

**Credit Conveyor Integration Adapter:**
- `credit_conveyor_kafka_messages_total{topic, status}` - сообщения Kafka
- `credit_conveyor_priority_applications_total{priority}` - заявки по приоритетам

---

## Безопасность

### Credit Application Service:
- **JWT Authentication:** для запросов из интернет-банка
- **API Key Authentication:** для запросов с сайта
- **Rate Limiting:** 100 req/min per IP (сайт), 1000 req/min (интернет-банк)
- **Шифрование паспортных данных в БД:** AES-256
- **Валидация паспортных данных:** формат, контрольная сумма

### Scoring Service (дополнительно):
- **API Key для Credit Application Service:** отдельный ключ для instant scoring
- **Rate Limiting для instant scoring:** 50 req/min (защита от перегрузки)
- **Кэширование данных БКИ:** шифрование в БД

### БКИ интеграция:
- **HTTPS/TLS:** обязательно
- **API Key от БКИ:** хранение в Vault
- **IP Whitelist:** IP-адреса серверов Scoring Service

---

## Масштабирование

### Горизонтальное:
- **Credit Application Service:** 2-6 инстансов (за Load Balancer)
- **Scoring Service:** 2-4 инстанса (уже поддерживается)
- **Credit Conveyor Integration Adapter:** 2-3 инстанса (Kafka consumer group)

### Вертикальное:
- **Credit Conveyor Oracle DB:** по необходимости (как АБС)

### Kafka:
- **Партиции:** 10 (параллельная обработка)
- **Consumer Groups:** credit-conveyor-consumers (3 consumers)

---

## Инструкция для draw.io:

1. Создать файл `C4_Container_Credits.drawio`
2. Взять за основу диаграмму контейнеров из Task 3
3. Добавить справа группу "Система онлайн кредитования":
   - Credit Application Service (контейнер, синий)
   - Credit Applications DB (БД, цилиндр)
4. Расширить группу "Система скоринга":
   - Scoring Service (контейнер, серый, обновлённый)
   - Pre-Approved Offers Batch Job (компонент, серый)
   - Scoring DB (БД, цилиндр, обновлённая)
5. Добавить группу "Кредитный конвейер":
   - Credit Conveyor BPM (контейнер, серый)
   - Credit Conveyor Integration Adapter (контейнер, синий - новый)
   - Credit Conveyor Oracle DB (БД, цилиндр)
6. Обновить Kafka Cluster (добавить новые топики)
7. Показать связи согласно потокам выше
8. Добавить легенду с технологиями и цветами
9. Опционально: добавить timing annotations (например, "batch: 02:00-06:00", "instant: < 5 sec")
