# C4 Container Diagram: Онлайн открытие депозитов (MVP)

## Описание диаграммы контейнеров

Эта диаграмма детализирует внутреннюю структуру "Системы онлайн открытия депозитов", а также показывает детали интернет-банка и АБС.

---

## Контейнеры системы "Онлайн открытие депозитов"

### 1. **API Gateway** [Container: Nginx + Kong]
- **Технологии:** Nginx, Kong API Gateway
- **Ответственность:**
  - Единая точка входа для всех внешних запросов
  - Авторизация и аутентификация
  - Rate limiting (защита от перегрузки)
  - Маршрутизация запросов к микросервисам
  - Логирование всех запросов
  - SSL/TLS терминация
- **Порты:** 443 (HTTPS)
- **Связи:**
  - ← Интернет-банк, Сайт, Система кол-центра (HTTPS)
  - → Deposit Application Service (HTTP)
  - → Rate Management Service (HTTP)

### 2. **Deposit Application Service** [Container: Spring Boot Application]
- **Технологии:** Java 11, Spring Boot 2.7, PostgreSQL
- **Ответственность:**
  - Управление жизненным циклом заявок на депозиты
  - Валидация заявок
  - Управление статусами
  - Публикация заявок в Kafka
  - REST API для получения статусов
  - Webhook-уведомления для системы кол-центра
  - Идемпотентность операций
  - Аудит всех операций
- **База данных:** Applications DB (PostgreSQL)
- **Порты:** 8080 (HTTP internal)
- **Связи:**
  - ← API Gateway (HTTP): "Принимает заявки"
  - → Applications DB: "Сохраняет заявки и статусы"
  - → Kafka: "Публикует заявки в топик 'deposit-applications'"
  - → Система кол-центра: "Webhook-уведомления о новых заявках с сайта"
  - ← АБС Integration Adapter: "Получает обновления статусов"

### 3. **Applications DB** [Container: PostgreSQL Database]
- **Технологии:** PostgreSQL 14
- **Схема:**
  - `deposit_applications` - заявки на депозиты
  - `application_status_history` - история изменений статусов
  - `audit_log` - аудит операций
- **Ответственность:**
  - Хранение заявок
  - Хранение истории статусов
  - Аудит
- **Связи:**
  - ← Deposit Application Service: "Чтение/запись заявок"

### 4. **Rate Management Service** [Container: Spring Boot Application]
- **Технологии:** Java 11, Spring Boot 2.7, PostgreSQL, Redis
- **Ответственность:**
  - Управление ставками депозитов
  - Валидация ставок (диапазоны, продукты)
  - REST API для публикации и получения ставок
  - Кэширование ставок в Redis
  - История изменений ставок
  - Веб-интерфейс для бэк-офиса кредитов
- **База данных:** Rates DB (PostgreSQL)
- **Кэш:** Redis
- **Порты:** 8081 (HTTP internal), 443 (HTTPS для UI)
- **Связи:**
  - ← API Gateway: "Запросы на получение ставок"
  - ← Сотрудник бэк-офиса кредитов (HTTPS): "Управление ставками через UI"
  - → Rates DB: "Сохраняет ставки"
  - → Redis Cache: "Кэширует актуальные ставки"

### 5. **Rates DB** [Container: PostgreSQL Database]
- **Технологии:** PostgreSQL 14
- **Схема:**
  - `deposit_rates` - актуальные ставки
  - `rate_history` - история изменений ставок
  - `rate_products` - связь ставок с продуктами
- **Ответственность:**
  - Хранение ставок
  - Хранение истории
- **Связи:**
  - ← Rate Management Service: "Чтение/запись ставок"

### 6. **Redis Cache** [Container: Redis]
- **Технологии:** Redis 7
- **Ответственность:**
  - Кэширование актуальных ставок
  - TTL: 15 минут (автообновление при изменении)
  - Ускорение загрузки справочных данных
- **Связи:**
  - ← Rate Management Service: "Запись ставок"
  - → API Gateway / микросервисы: "Чтение ставок"

### 7. **Kafka Cluster** [Container: Apache Kafka]
- **Технологии:** Apache Kafka 3.4
- **Топики:**
  - `deposit-applications` - заявки на депозиты из всех каналов
  - `deposit-application-status` - обновления статусов от АБС
- **Конфигурация:**
  - 3 брокера
  - Replication factor: 2
  - Партиции: 10 (для масштабирования)
- **Ответственность:**
  - Буферизация заявок для АБС
  - Гарантированная доставка
  - Защита АБС от перегрузки
- **Связи:**
  - ← Deposit Application Service: "Публикация заявок"
  - → АБС Integration Adapter: "Чтение заявок"
  - ← АБС Integration Adapter: "Публикация статусов"
  - → Deposit Application Service: "Чтение статусов"

---

## Детализация Интернет-банка (для контекста интеграции)

### 8. **Internet Banking Web App** [Container: ASP.NET MVC Application]
- **Технологии:** ASP.NET MVC 4.5, .NET Framework 4.5, C#
- **Ответственность:**
  - Авторизация клиентов
  - Отображение депозитов с персонализированными ставками
  - Форма подачи заявки на депозит
  - Генерация и валидация СМС-кодов (существующий функционал)
  - Отображение статусов заявок
  - Отображение открытых депозитов
- **Связи:**
  - ← Клиент банка (HTTPS): "Использует интернет-банк"
  - → API Gateway: "Получает ставки, подаёт заявки, получает статусы"
  - → Internet Banking DB: "Работает с данными клиентов"
  - → СМС-шлюз: "Отправляет СМС-коды"

### 9. **Internet Banking DB** [Container: MS SQL Database]
- **Технологии:** MS SQL Server
- **Ответственность:**
  - Хранение сессий клиентов
  - Кэш данных из АБС
  - Временные данные
- **Связи:**
  - ← Internet Banking Web App: "Чтение/запись"
  - ↔ АБС Oracle DB: "Репликация/синхронизация данных"

---

## Детализация АБС (для контекста интеграции)

### 10. **АБС Desktop Client** [Container: Delphi Application]
- **Технологии:** Delphi
- **Ответственность:**
  - Интерфейс для сотрудников отделений
  - Интерфейс для бэк-офиса депозитов
  - Интерфейс для бэк-офиса кредитов
  - Обработка заявок на депозиты
  - Открытие депозитов
  - Управление клиентами
- **Связи:**
  - ← Сотрудники (Desktop): "Работают в АБС"
  - → АБС Oracle DB: "Все операции через БД"

### 11. **АБС Business Logic** [Container: PL-SQL Procedures]
- **Технологии:** Oracle PL-SQL
- **Ответственность:**
  - Основная бизнес-логика банка
  - Учёт операций по счетам
  - Создание депозитов
  - Валидация операций
  - Процедуры обработки заявок
- **Расположение:** В Oracle DB (stored procedures)
- **Связи:**
  - ← АБС Desktop Client: "Вызовы процедур"
  - ← АБС Integration Adapter: "Вызовы процедур"
  - → АБС Oracle DB: "Работа с данными"

### 12. **АБС Integration Adapter** [Container: Java Application]
- **Технологии:** Java 11, JDBC, Kafka Client
- **Ответственность:**
  - Чтение заявок из Kafka
  - Запись заявок в таблицы АБС
  - Вызов PL-SQL процедур для обработки
  - Чтение ставок из Rate Management Service
  - Публикация обновлений статусов в Kafka
  - Изоляция АБС от прямых вызовов
- **Развёртывание:** На серверах АБС (Java-процесс)
- **Связи:**
  - ← Kafka: "Читает заявки из топика 'deposit-applications'"
  - → АБС Oracle DB: "Записывает заявки в таблицы"
  - → АБС Business Logic: "Вызывает процедуры обработки"
  - → Kafka: "Публикует статусы в топик 'deposit-application-status'"
  - → Rate Management Service: "Получает актуальные ставки"

### 13. **АБС Oracle DB** [Container: Oracle Database]
- **Технологии:** Oracle 19c
- **Схема:**
  - `clients` - клиенты банка
  - `accounts` - счета клиентов
  - `deposits` - депозиты
  - `deposit_applications_external` - новая таблица для онлайн-заявок
  - `products` - депозитные продукты
  - 500+ других таблиц
- **Ответственность:**
  - Основное хранилище всех банковских данных
  - Бизнес-логика (PL-SQL процедуры)
- **Масштабирование:** Только вертикальное
- **Связи:**
  - ← АБС Desktop Client, АБС Business Logic, АБС Integration Adapter

---

## Детализация других существующих систем (краткая)

### 14. **Сайт** [Container: PHP + React.js Application]
- **Технологии:** PHP 8.1, React.js 18, Nginx
- **Ответственность:**
  - Отображение маркетинговой информации
  - Страница депозитов с актуальными ставками
  - Форма подачи заявки
- **Связи:**
  - ← Новый клиент (HTTPS): "Просматривает депозиты"
  - → API Gateway: "Получает ставки, отправляет заявки"

### 15. **Система кол-центра** [Container: Microservices]
- **Технологии:** React.js, Java Spring Boot, PostgreSQL
- **Ответственность:**
  - Управление обращениями
  - Интерфейс для операторов
  - Обработка заявок с сайта
  - Просмотр актуальных ставок
- **Связи:**
  - ← Сотрудник кол-центра (HTTPS): "Работает с заявками"
  - → API Gateway: "Получает заявки, ставки, обновляет статусы"
  - ← Deposit Application Service (Webhook): "Уведомления о новых заявках"

### 16. **СМС-шлюз** [Container: Application]
- **Технологии:** Внутренняя разработка
- **Ответственность:**
  - Генерация СМС-кодов
  - Отправка СМС через телеком-оператора
  - Валидация СМС-кодов
- **Связи:**
  - ← АБС, Internet Banking: "Команды на отправку СМС"
  - → Телеком-оператор (API): "Отправка СМС"

---

## Ключевые потоки данных (детализированные)

### Поток 1: Подача заявки из интернет-банка
```
1. Клиент банка → Internet Banking Web App (HTTPS)
   - Заполняет форму заявки

2. Internet Banking Web App → API Gateway (HTTPS)
   - POST /api/v1/deposit-applications
   - Body: {clientId, productId, accountId, amount}

3. API Gateway → Deposit Application Service (HTTP)
   - Маршрутизация, авторизация, rate limiting

4. Deposit Application Service → Applications DB
   - Сохранение заявки (status: 'CREATED')

5. Deposit Application Service → Kafka
   - Публикация в топик 'deposit-applications'
   - Message: {applicationId, clientId, productId, accountId, amount, timestamp}

6. Kafka → АБС Integration Adapter
   - Чтение сообщения (consumer group)

7. АБС Integration Adapter → АБС Oracle DB
   - INSERT в таблицу deposit_applications_external

8. АБС Integration Adapter → АБС Business Logic
   - Вызов процедуры validate_deposit_application()

9. Сотрудник бэк-офиса депозитов → АБС Desktop Client
   - Видит заявку, обрабатывает

10. АБС Desktop Client → АБС Business Logic
    - Вызов процедуры approve_deposit_application()

11. АБС Business Logic → АБС Integration Adapter (через триггер)
    - Уведомление об изменении статуса

12. АБС Integration Adapter → Kafka
    - Публикация в топик 'deposit-application-status'
    - Message: {applicationId, status: 'APPROVED', timestamp}

13. Kafka → Deposit Application Service
    - Чтение обновления статуса

14. Deposit Application Service → Applications DB
    - UPDATE status = 'APPROVED'

15. Internet Banking Web App → API Gateway → Deposit Application Service
    - GET /api/v1/deposit-applications/{id} (polling или WebSocket)
    - Клиент видит обновлённый статус

16. АБС Business Logic → СМС-шлюз
    - Отправка СМС о подтверждении ставки

17. СМС-шлюз → Телеком-оператор → Клиент
    - Доставка СМС
```

### Поток 2: Получение актуальных ставок
```
1. Сотрудник бэк-офиса кредитов → Rate Management Service UI (HTTPS)
   - Вводит новые ставки

2. Rate Management Service → Rates DB
   - INSERT в таблицу deposit_rates

3. Rate Management Service → Redis Cache
   - SET rates:{product_id} = {rate_data}, TTL: 900s

4. Клиент/Новый клиент → [Интернет-банк / Сайт] (HTTPS)
   - Открывает страницу депозитов

5. [Интернет-банк / Сайт] → API Gateway (HTTPS)
   - GET /api/v1/rates

6. API Gateway → Rate Management Service (HTTP)
   - Маршрутизация запроса

7. Rate Management Service → Redis Cache
   - GET rates:* (если есть в кэше)

8. Rate Management Service → Rates DB
   - SELECT (если нет в кэше)

9. Rate Management Service → API Gateway → [Интернет-банк / Сайт]
   - Response: JSON с актуальными ставками

10. Клиент видит ставки (< 500 мс)
```

---

## Технические детали

### API Gateway (Kong) - основные плагины:
- **Rate Limiting:** 100 req/sec per IP
- **JWT Auth:** Валидация токенов для интернет-банка
- **CORS:** Для веб-клиентов
- **Logging:** В ELK Stack
- **Prometheus:** Метрики

### Deposit Application Service - endpoints:
- `POST /api/v1/deposit-applications` - создание заявки
- `GET /api/v1/deposit-applications/{id}` - получение статуса
- `GET /api/v1/deposit-applications` - список заявок (для бэк-офиса)
- `PATCH /api/v1/deposit-applications/{id}/status` - обновление статуса

### Rate Management Service - endpoints:
- `GET /api/v1/rates` - получение всех актуальных ставок
- `GET /api/v1/rates/{productId}` - получение ставки по продукту
- `POST /api/v1/rates` - создание/обновление ставок (только для бэк-офиса)
- `GET /api/v1/rates/history` - история изменений

### Kafka - конфигурация топиков:

**deposit-applications:**
- Партиции: 10
- Replication factor: 2
- Retention: 7 дней
- Compression: snappy

**deposit-application-status:**
- Партиции: 10
- Replication factor: 2
- Retention: 30 дней
- Compression: snappy

---

## Безопасность

### Уровень API Gateway:
- SSL/TLS 1.3
- JWT токены для авторизации
- Rate limiting: 100 req/sec per IP, 1000 req/sec global
- IP whitelist для внутренних систем

### Уровень микросервисов:
- Внутренняя сеть (недоступна извне)
- Mutual TLS между сервисами (опционально для MVP)
- Аутентификация через JWT
- Валидация всех входных данных

### Уровень данных:
- Encryption at rest (PostgreSQL)
- Шифрование чувствительных полей (ФИО, телефоны)
- Разграничение доступа (роли БД)

---

## Масштабирование

### Горизонтальное масштабирование:
- **API Gateway:** 2-4 инстанса за Load Balancer
- **Deposit Application Service:** 2-6 инстансов
- **Rate Management Service:** 2-4 инстанса
- **Kafka:** 3 брокера (возможно расширение до 5)
- **Redis:** Master-Replica (1+2)

### Вертикальное масштабирование:
- **АБС Oracle DB:** 32 CPU, 256 GB RAM (текущее)
- **PostgreSQL БД:** 16 CPU, 64 GB RAM (начальное)

### Load Balancing:
- Nginx Load Balancer перед API Gateway
- Round-robin для микросервисов
- Sticky sessions для интернет-банка (существующее)

---

## Мониторинг и наблюдаемость

### Metrics (Prometheus + Grafana):
- Request rate, latency, errors (RED method)
- Kafka lag, throughput
- Database connections, query time
- JVM metrics (heap, GC)

### Logging (ELK Stack):
- Централизованное логирование всех компонентов
- Correlation ID для трассировки запросов
- Уровни: ERROR, WARN, INFO, DEBUG

### Tracing (Jaeger - опционально):
- Распределённая трассировка запросов
- Визуализация задержек

### Alerting (Prometheus Alertmanager):
- Доступность < 99.9%
- Latency > 1 sec (p95)
- Kafka lag > 1000 messages
- Error rate > 1%

---

## Примечания для визуализации

**Цветовая схема:**
- **Синий** - Новые контейнеры системы онлайн депозитов
- **Светло-синий** - Вспомогательные компоненты (Kafka, Redis)
- **Серый** - Существующие контейнеры
- **Цилиндры** - Базы данных

**Группировка:**
- Сгруппировать контейнеры по системам (пунктирные границы)
- "Система онлайн депозитов" - отдельная группа
- "Интернет-банк" - отдельная группа
- "АБС" - отдельная группа

**Стиль C4:**
- Прямоугольники для приложений
- Цилиндры для БД
- Указать технологии в заголовках
- Стрелки с протоколами и краткими описаниями

---

## Инструкция для создания диаграммы в draw.io:

1. Создать файл `C4_Container_Deposits_MVP.drawio`
2. Использовать библиотеку C4 Model
3. Создать 3 группы контейнеров:
   - "Система онлайн депозитов" (синий фон) - в центре
   - "Интернет-банк" (серый фон) - слева
   - "АБС" (серый фон) - справа
4. Разместить контейнеры согласно списку выше
5. Соединить стрелками с указанием протоколов
6. Добавить легенду с технологиями и цветами
7. Опционально: добавить deployment diagram с ЦОД
