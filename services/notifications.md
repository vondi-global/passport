# Notification Service - System Passport

> **Версия:** 1.0.0 | **Статус:** Production Ready | **Обновлено:** 2025-12-21

## Обзор

**Notification Service** — микросервис для управления уведомлениями пользователей на платформе Vondi.
Реализует омниканальную доставку уведомлений через Email (Zoho Mail), Telegram Bot, SMS, Push и In-App каналы.

### Технологический стек

- **Язык:** Go 1.23
- **Архитектура:** Domain-Driven Design (DDD)
- **База данных:** PostgreSQL (migrations-based)
- **Кеширование:** Redis
- **Протокол:** gRPC + HTTP
- **Логирование:** zerolog

### Директория проекта

```
/p/github.com/vondi-global/notifications/
```

### Каналы доставки

| Канал | Провайдер | Статус | Rate Limit |
|-------|-----------|--------|------------|
| **Email** | Zoho Mail API (OAuth2) | ✅ Готов | 100 emails/min |
| **Telegram** | Telegram Bot API (@VondiNotifyBot) | ✅ Готов | 30 msg/sec |
| **Push** | FCM | ⏳ Placeholder | - |
| **SMS** | SMS Gateway | ⏳ Placeholder | - |
| **In-App** | PostgreSQL | ✅ Готов | Unlimited |

---

## Архитектура (DDD)

```
notifications/
├── cmd/server/main.go              # Точка входа
├── internal/
│   ├── app/app.go                  # Bootstrap приложения
│   ├── config/config.go            # Конфигурация (VONDINOTIFY_*)
│   │
│   ├── domain/                     # БИЗНЕС-ЛОГИКА
│   │   ├── types/                  # Value Objects (Channel, Priority, Status, Locale)
│   │   ├── notification/           # Notification entity (aggregate root)
│   │   ├── events/                 # 50+ EventTypes (order.created, payment.completed...)
│   │   ├── settings/               # UserSettings, TelegramConnection
│   │   ├── template/               # Template entity, SimpleRenderer
│   │   └── delivery/               # DeliveryLog entity
│   │
│   ├── application/                # USE CASES
│   │   ├── send_notification.go    # Отправка уведомления
│   │   ├── get_notifications.go    # Получение списка
│   │   ├── update_settings.go      # Обновление настроек
│   │   └── connect_telegram.go     # Привязка Telegram
│   │
│   ├── infrastructure/             # РЕАЛИЗАЦИЯ
│   │   ├── persistence/postgres/   # PostgreSQL репозитории
│   │   ├── provider/
│   │   │   ├── email/              # Zoho Mail (OAuth + API клиент)
│   │   │   └── telegram/           # Bot client + Rate limiter
│   │   ├── queue/                  # Redis producer/consumer
│   │   └── i18n/                   # Локализация (sr/ru/en)
│   │
│   └── interfaces/                 # АДАПТЕРЫ
│       ├── grpc/                   # gRPC handlers
│       └── http/                   # HTTP (health, webhooks, metrics)
│
├── api/proto/notify/v1/            # Protobuf definitions
├── migrations/                     # PostgreSQL миграции (001-003)
└── docker-compose.yml              # PostgreSQL + Redis
```

---

## Сетевая конфигурация

### Порты

| Протокол | Порт | Назначение | Переменная |
|----------|------|-----------|------------|
| gRPC | **50054** | Основной API для микросервисов | `VONDINOTIFY_SERVER_GRPC_PORT` |
| HTTP | **8088** | Health check, webhooks, Prometheus metrics | `VONDINOTIFY_SERVER_HTTP_PORT` |
| PostgreSQL | **35436** | База данных (notifications_db) | `VONDINOTIFY_DB_PORT` |
| Redis | **36382** | Очереди, rate limiting | `VONDINOTIFY_REDIS_PORT` |

### Внешние зависимости

| Сервис | URL | Назначение |
|--------|-----|-----------|
| Zoho OAuth | accounts.zoho.eu | Токены для email |
| Zoho Mail | mail.zoho.eu | Отправка писем |
| Telegram API | api.telegram.org | Отправка сообщений |
| Main DB | localhost:5433 | Получение preferences (read-only) |
| Auth Service | localhost:28086 | Проверка прав |

---

## База данных

### PostgreSQL (notifications_db)

**Порт:** 35436
**Пользователь:** notify_user
**Пароль:** notify_secret
**DSN:** `postgres://notify_user:notify_secret@localhost:35436/notifications_db?sslmode=disable`

### Таблицы

| Таблица | Записей | Назначение |
|---------|---------|-----------|
| `notifications` | - | In-app уведомления |
| `user_notification_preferences` | - | Глобальные настройки пользователя |
| `notification_type_settings` | - | Настройки per event type |
| `user_telegram_connections` | - | Привязка Telegram аккаунтов |
| `delivery_log` | - | Логи доставки с retry tracking |

### Индексы

```sql
-- notifications
idx_notifications_user_id         -- Поиск по user_id
idx_notifications_user_unread     -- Непрочитанные (WHERE is_read = FALSE)
idx_notifications_created_at      -- Сортировка по дате
idx_notifications_type            -- Фильтр по типу события

-- delivery_log
idx_delivery_log_notification     -- По notification_id
idx_delivery_log_user             -- По user_id
idx_delivery_log_status           -- По статусу (pending)
idx_delivery_log_retry            -- Для retry механизма

-- telegram
idx_telegram_chat_id              -- Reverse lookup по chat_id
```

---

## Переменные окружения

**Prefix:** `VONDINOTIFY_`

### Service

```bash
VONDINOTIFY_SERVICE_NAME=notification-service
VONDINOTIFY_SERVICE_ENV=development
VONDINOTIFY_SERVICE_VERSION=1.0.0
```

### Server

```bash
VONDINOTIFY_SERVER_GRPC_PORT=50054
VONDINOTIFY_SERVER_HTTP_PORT=8088
```

### Database

```bash
VONDINOTIFY_DB_HOST=localhost
VONDINOTIFY_DB_PORT=35436
VONDINOTIFY_DB_NAME=notifications_db
VONDINOTIFY_DB_USER=notify_user
VONDINOTIFY_DB_PASSWORD=notify_secret
VONDINOTIFY_DB_SSLMODE=disable
VONDINOTIFY_DB_MAX_OPEN_CONNS=25
VONDINOTIFY_DB_MAX_IDLE_CONNS=10
VONDINOTIFY_DB_CONN_MAX_LIFETIME=5m
```

### Main Database (Read-Only)

Подключение к основной БД Vondi для получения языковых настроек пользователей.

```bash
VONDINOTIFY_MAINDB_ENABLED=true
VONDINOTIFY_MAINDB_HOST=localhost
VONDINOTIFY_MAINDB_PORT=5433
VONDINOTIFY_MAINDB_NAME=vondi_db
VONDINOTIFY_MAINDB_USER=postgres
VONDINOTIFY_MAINDB_PASSWORD=mX3g1XGhMRUZEX3l
VONDINOTIFY_MAINDB_SSLMODE=disable
VONDINOTIFY_MAINDB_AUTH_SERVICE_URL=http://localhost:28086
```

### Redis

```bash
VONDINOTIFY_REDIS_HOST=localhost
VONDINOTIFY_REDIS_PORT=36382
VONDINOTIFY_REDIS_PASSWORD=
VONDINOTIFY_REDIS_DB=0
```

### Zoho Mail

```bash
VONDINOTIFY_ZOHO_CLIENT_ID=1000.WYUN6SHOC65CCO5VOL6CP307TG3JWL
VONDINOTIFY_ZOHO_CLIENT_SECRET=***        # REQUIRED
VONDINOTIFY_ZOHO_REFRESH_TOKEN=***        # REQUIRED
VONDINOTIFY_ZOHO_ACCOUNT_ID=7772578000000002002
VONDINOTIFY_ZOHO_FROM_EMAIL=mail@vondi.rs
VONDINOTIFY_ZOHO_FROM_NAME=Vondi
VONDINOTIFY_ZOHO_ACCOUNTS_DOMAIN=accounts.zoho.eu
VONDINOTIFY_ZOHO_MAIL_DOMAIN=mail.zoho.eu
```

### Telegram

```bash
VONDINOTIFY_TELEGRAM_BOT_TOKEN=8264184887:AAG7HolQ5u-Ea8kfEwHF2RNSardP90CKeig
VONDINOTIFY_TELEGRAM_WEBHOOK_URL=https://notifications.vondi.rs/webhook/telegram
```

### i18n

```bash
VONDINOTIFY_I18N_DEFAULT_LOCALE=sr
VONDINOTIFY_I18N_SUPPORTED_LOCALES=sr,ru,en
VONDINOTIFY_I18N_FALLBACK_LOCALE=en
VONDINOTIFY_I18N_MESSAGES_PATH=./internal/infrastructure/i18n/messages
```

### Queue & Retry

```bash
VONDINOTIFY_QUEUE_WORKERS=5
VONDINOTIFY_QUEUE_BATCH_SIZE=10
VONDINOTIFY_RETRY_MAX_ATTEMPTS=5
VONDINOTIFY_RETRY_INITIAL_DELAY=5s
VONDINOTIFY_RETRY_MAX_DELAY=1h
```

---

## Domain Layer

### Event Types (54 типа)

Полный список определённых типов событий, сгруппированных по категориям.

| Категория | Количество | События |
|-----------|------------|---------|
| **Orders** | 8 | order.created, order.confirmed, order.accepted, order.processing, order.shipped, order.delivered, order.cancelled, order.refunded |
| **Payments** | 5 | payment.initiated, payment.completed, payment.failed, payment.refunded, payment.cod_confirmed |
| **Delivery** | 7 | delivery.created, delivery.confirmed, delivery.in_transit, delivery.out_for_delivery, delivery.delivered, delivery.failed, delivery.returned |
| **Users** | 7 | user.registered, user.email_verified, user.phone_verified, user.password_changed, user.password_reset, user.role_changed, user.banned |
| **Chat** | 3 | chat.created, chat.message_received, chat.unread_reminder |
| **Reviews** | 3 | review.posted, review.liked, review.seller_replied |
| **Listings** | 7 | listing.created, listing.approved, listing.rejected, listing.status_changed, listing.price_changed, listing.favorited, listing.sold_out |
| **Storefront** | 4 | storefront.created, storefront.product_added, storefront.sale_started, storefront.sale_ended |
| **Warehouse** | 4 | warehouse.invitation_created, warehouse.invitation_accepted, warehouse.staff_removed, warehouse.staff_role_changed |
| **Franchise** | 7 | franchise.application_created, franchise.application_approved, franchise.application_rejected, franchise.info_requested, franchise.info_provided, franchise.training_scheduled, franchise.contract_signed |
| **Direct** | 1 | direct.email (bypass event type validation) |
| **System** | 2 | system.maintenance, system.announcement |

**Итого:** 54 типа событий

**Файл:** `/p/github.com/vondi-global/notifications/internal/domain/events/types.go`

### Domain Entities

#### 1. Notification (In-App Notification)

**Файл:** `/p/github.com/vondi-global/notifications/internal/domain/notification/entity.go`

Агрегат для хранения внутриплатформенных уведомлений.

```go
type Notification struct {
    ID        int64                  // Уникальный ID
    UserID    int64                  // ID пользователя
    Type      events.EventType       // Тип события (order.shipped, payment.completed...)
    Title     string                 // Заголовок
    Body      string                 // Текст уведомления
    Data      map[string]interface{} // Дополнительные данные (JSONB)
    IsRead    bool                   // Прочитано ли
    CreatedAt time.Time              // Дата создания
    ReadAt    *time.Time             // Дата прочтения (nullable)
}
```

**Методы:**
- `NewNotification(userID, eventType, title, body)` - создание
- `MarkAsRead()` - пометить как прочитанное
- `SetData(key, value)` - добавить метаданные
- `Validate()` - валидация
- `IsNew()` - проверка, что новое
- `Age()` - возраст уведомления

#### 2. UserSettings (Настройки уведомлений)

**Файл:** `/p/github.com/vondi-global/notifications/internal/domain/settings/entity.go`

Настройки глобальных и типоспецифичных каналов доставки.

```go
type UserSettings struct {
    UserID          int64
    PreferredLocale Locale               // sr/en/ru
    EmailEnabled    bool                 // Email глобально
    TelegramEnabled bool                 // Telegram глобально
    SMSEnabled      bool                 // SMS глобально
    PushEnabled     bool                 // Push глобально
    DNDStart        *time.Time           // Do Not Disturb с (время дня)
    DNDEnd          *time.Time           // Do Not Disturb по
    TypeSettings    map[EventType]*TypeSetting  // Per event type
    CreatedAt       time.Time
    UpdatedAt       time.Time
}

type TypeSetting struct {
    EventType       events.EventType
    EmailEnabled    bool
    TelegramEnabled bool
    SMSEnabled      bool
    PushEnabled     bool
}
```

**Методы:**
- `NewUserSettings(userID)` - создание с defaults (email=true, telegram=true, sms=false, push=true, locale=sr)
- `IsChannelEnabled(eventType, channel) bool` - проверка, включен ли канал
- `GetEnabledChannels(eventType) []Channel` - список включенных каналов
- `IsInDND() bool` - режим "Не беспокоить"
- `SetTypeSetting(*TypeSetting)` - установить настройки для типа события

#### 3. TelegramConnection

```go
type TelegramConnection struct {
    UserID           int64
    TelegramChatID   int64     // Telegram chat_id
    TelegramUsername string    // @username
    ConnectedAt      time.Time
}
```

#### 4. Template (Шаблон уведомления)

**Файл:** `/p/github.com/vondi-global/notifications/internal/domain/template/entity.go`

```go
type Template struct {
    EventType events.EventType
    Locale    Locale
    Channel   Channel
    Subject   string  // Только для email
    Body      string  // Текст с переменными {{var}}
    IsActive  bool
}

type TemplateKey struct {
    EventType events.EventType
    Locale    Locale
    Channel   Channel
}
```

**Примеры переменных:**
- `{{user_name}}` - Имя пользователя
- `{{order_id}}` - ID заказа
- `{{tracking_number}}` - Трек-номер
- `{{price}}` - Цена
- `{{link}}` - Ссылка

#### 5. DeliveryLog (Журнал доставки)

**Файл:** `/p/github.com/vondi-global/notifications/internal/domain/delivery/entity.go`

```go
type DeliveryLog struct {
    ID                int64
    NotificationID    *int64              // Nullable (email-only без in-app)
    UserID            int64
    Channel           Channel
    Status            DeliveryStatus
    ProviderMessageID string              // ID от Zoho/Telegram
    ErrorMessage      string
    RetryCount        int
    SentAt            *time.Time
    DeliveredAt       *time.Time
    CreatedAt         time.Time
}
```

**Методы:**
- `NewDeliveryLog(notificationID, userID, channel)`
- `MarkQueued()`, `MarkSent(msgID)`, `MarkDelivered()`, `MarkFailed(err)`, `MarkSkipped(reason)`
- `CanRetry(maxAttempts) bool` - можно ли повторить
- `IsFinal() bool` - финальный статус

### Value Objects

| Тип | Значения | Файл |
|-----|----------|------|
| **Channel** | email, telegram, sms, push, in_app | `domain/types/channel.go` |
| **Priority** | low (0), medium (1), high (2), critical (3) | `domain/types/priority.go` |
| **Locale** | sr (primary), en, ru | `domain/types/locale.go` |
| **DeliveryStatus** | pending, queued, sent, delivered, failed, skipped | `domain/types/status.go` |

### Delivery Status Flow

```
pending → queued → sent → delivered ✓
                      ↓
                   failed → (retry до 5 раз) → delivered ✓
                                            → dead_letter_queue
```

---

## Infrastructure

### Zoho Mail Provider

**OAuth Token Manager:**
- Автоматический refresh токена
- Thread-safe (RWMutex)
- Safety margin 5 минут перед истечением

**ZohoClient:**
- POST `/accounts/{id}/messages`
- HTML и plain text поддержка
- Обработка 401 → InvalidateToken()

### Telegram Provider

**BotClient:**
- Поддержка HTML/Markdown ParseMode
- Rate limiter: 30 msg/sec (token bucket)
- Per-chat rate limiting
- Exponential backoff retry

**RateLimiter:**
- Token bucket алгоритм
- Global + per-chat лимиты
- Non-retryable errors: Forbidden, blocked, deactivated, kicked

### i18n System

**Локализация:**
- JSON файлы: sr.json, ru.json, en.json
- Fallback на английский
- Подстановка переменных: `{variable_name}`

**Template Rendering:**
- Channel-specific: email (subject+body), telegram (telegram→body)
- HTML templates для email (embedded)

---

## API (gRPC)

### NotificationService (Управление уведомлениями)

**Файл:** `/p/github.com/vondi-global/notifications/internal/interfaces/grpc/notification_handler.go`

| Метод | Описание | Request | Response |
|-------|----------|---------|----------|
| `SendNotification` | Отправить уведомление пользователю | user_id/email, event_type, variables, channels, priority | notification_id, channel_results, success |
| `GetNotifications` | Получить список уведомлений | user_id, limit, offset, unread_only | notifications[], total, unread_count |
| `GetUnreadCount` | Счётчик непрочитанных | user_id | count |
| `MarkAsRead` | Пометить как прочитанные | user_id, notification_ids[] | Empty |
| `MarkAllAsRead` | Пометить все как прочитанные | user_id | Empty |
| `DeleteNotification` | Удалить уведомление | user_id, notification_id | Empty |

**Пример SendNotification:**

```protobuf
message SendNotificationRequest {
  int64 user_id = 1;                    // ID пользователя (обязательно ИЛИ email)
  string email = 2;                     // Email (альтернатива user_id)
  string event_type = 3;                // "order.shipped", "payment.completed"...
  Locale locale = 4;                    // sr/en/ru (опционально)
  map<string, string> variables = 5;    // {"order_id": "ORD-123", "tracking_number": "TRK-456"}
  repeated Channel channels = 6;        // [EMAIL, TELEGRAM] (опционально, по умолчанию все)
  Priority priority = 7;                // LOW/MEDIUM/HIGH/CRITICAL
  map<string, string> metadata = 8;     // Дополнительные данные
}

message SendNotificationResponse {
  int64 notification_id = 1;            // ID созданного уведомления
  repeated ChannelDeliveryResult channel_results = 2;
  bool success = 3;
}

message ChannelDeliveryResult {
  Channel channel = 1;                  // EMAIL/TELEGRAM/SMS/PUSH/IN_APP
  DeliveryStatus status = 2;            // PENDING/SENT/DELIVERED/FAILED/SKIPPED
  string message_id = 3;                // ID от провайдера (Zoho/Telegram)
  string error = 4;                     // Текст ошибки (если есть)
}
```

### SettingsService (Управление настройками)

**Файл:** `/p/github.com/vondi-global/notifications/internal/interfaces/grpc/settings_handler.go`

| Метод | Описание | Request | Response |
|-------|----------|---------|----------|
| `GetSettings` | Получить настройки пользователя | user_id | UserSettings |
| `UpdateSettings` | Обновить глобальные настройки | user_id, optional fields | UserSettings |
| `GetTypeSetting` | Получить настройки для типа события | user_id, event_type | TypeSetting |
| `UpdateTypeSetting` | Обновить настройки для типа | user_id, event_type, channels | TypeSetting |
| `DeleteTypeSetting` | Удалить настройки типа (вернуться к глобальным) | user_id, event_type | Empty |

**UserSettings структура:**

```protobuf
message UserSettings {
  int64 user_id = 1;
  Locale preferred_locale = 2;          // sr/en/ru
  bool email_enabled = 3;               // Email глобально включен
  bool telegram_enabled = 4;            // Telegram глобально включен
  bool sms_enabled = 5;                 // SMS глобально включен
  bool push_enabled = 6;                // Push глобально включен
  google.protobuf.Timestamp dnd_start = 7;   // Do Not Disturb с (время дня)
  google.protobuf.Timestamp dnd_end = 8;     // Do Not Disturb по
  google.protobuf.Timestamp created_at = 9;
  google.protobuf.Timestamp updated_at = 10;
}

message TypeSetting {
  string event_type = 1;                // "order.shipped"
  bool email_enabled = 2;
  bool telegram_enabled = 3;
  bool sms_enabled = 4;
  bool push_enabled = 5;
}
```

### TelegramService (Подключение Telegram)

**Файл:** `/p/github.com/vondi-global/notifications/internal/interfaces/grpc/telegram_handler.go`

| Метод | Описание | Request | Response |
|-------|----------|---------|----------|
| `ConnectTelegram` | Начать процесс подключения Telegram | user_id | connect_url, bot_username, deep_link, token, expires_at |
| `VerifyTelegramConnection` | Проверить и завершить подключение (вызывается ботом) | token, chat_id, username | Empty |
| `DisconnectTelegram` | Отключить Telegram | user_id | Empty |
| `GetTelegramConnection` | Получить информацию о подключении | user_id | TelegramConnection |

**ConnectTelegram Response:**

```protobuf
message ConnectTelegramResponse {
  string connect_url = 1;               // URL для подключения
  string bot_username = 2;              // @VondiNotifyBot
  string deep_link = 3;                 // Deep link для мобильных
  string token = 4;                     // Одноразовый токен
  google.protobuf.Timestamp expires_at = 5;  // Срок действия токена
}

message TelegramConnection {
  int64 user_id = 1;
  int64 chat_id = 2;                    // Telegram chat_id
  string username = 3;                  // @username
  google.protobuf.Timestamp connected_at = 4;
}
```

---

## Команды

### Makefile

```bash
# Сборка
make build              # → ./bin/notification-service
make run                # Build + run
make dev                # go run (development)

# Тесты
make test               # go test -v -race -cover
make test-coverage      # Coverage report
make lint               # golangci-lint
make format             # gofumpt + goimports
make check              # format + lint + test

# Миграции
make migrate-up         # Применить
make migrate-down       # Откатить все
make migrate-down-1     # Откатить одну
make migrate-create     # Создать новую

# Docker
make docker-build       # Build image
make docker-up          # docker-compose up -d
make docker-down        # docker-compose down
make docker-logs        # Логи
```

### Быстрый старт

```bash
cd /p/github.com/vondi-global/notifications
cp .env.example .env
# Заполнить VONDINOTIFY_ZOHO_CLIENT_SECRET и VONDINOTIFY_ZOHO_REFRESH_TOKEN

make docker-up
make migrate-up
make dev

# Проверка
curl http://localhost:8088/health
```

---

## Health Check

```bash
# HTTP
curl http://localhost:8088/health
# {"status":"healthy","version":"1.0.0"}

# gRPC
grpcurl -plaintext localhost:50054 grpc.health.v1.Health/Check
```

---

## Prometheus Metrics

**Endpoint:** `http://localhost:8088/metrics`

| Метрика | Тип | Описание |
|---------|-----|----------|
| `vondi_notifications_sent_total` | Counter | Отправленные уведомления (channel, status) |
| `vondi_notifications_delivery_duration_seconds` | Histogram | Время доставки |
| `vondi_notifications_queue_size` | Gauge | Размер очереди |
| `vondi_notifications_retry_count` | Counter | Количество retry |
| `vondi_notifications_errors_total` | Counter | Ошибки (type, channel) |

---

## Зависимости (go.mod)

| Пакет | Версия | Назначение |
|-------|--------|-----------|
| google.golang.org/grpc | v1.69.2 | gRPC сервер |
| google.golang.org/protobuf | v1.36.8 | Protobuf |
| github.com/gofiber/fiber/v2 | v2.52.5 | HTTP фреймворк |
| github.com/jackc/pgx/v5 | v5.7.6 | PostgreSQL драйвер |
| github.com/redis/go-redis/v9 | v9.17.2 | Redis клиент |
| github.com/go-telegram-bot-api/telegram-bot-api/v5 | v5.5.1 | Telegram Bot |
| github.com/rs/zerolog | v1.34.0 | Логирование |
| github.com/kelseyhightower/envconfig | v1.4.0 | Config из env |
| github.com/prometheus/client_golang | v1.23.2 | Prometheus metrics |

---

## Известные ограничения

1. **Telegram требует привязки** — без `telegram_chat_id` уведомления не отправляются
2. **Rate limiting Telegram** — 30 msg/sec глобально
3. **SMS и Push** — placeholders, требуют интеграции с провайдерами
4. **Zoho OAuth** — refresh token может истечь (требует периодического обновления)

---

## Миграции базы данных

### 001_notifications.up.sql

**Таблица:** `notifications` - In-app уведомления пользователей

```sql
CREATE TABLE IF NOT EXISTS notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    type VARCHAR(100) NOT NULL,        -- event_type (order.shipped...)
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    data JSONB DEFAULT '{}',           -- Дополнительные метаданные
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    read_at TIMESTAMP WITH TIME ZONE
);

-- Индексы
CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_user_unread ON notifications(user_id) WHERE is_read = FALSE;
CREATE INDEX idx_notifications_created_at ON notifications(created_at DESC);
CREATE INDEX idx_notifications_type ON notifications(type);
```

### 002_settings.up.sql

**Три таблицы:**

1. **user_notification_preferences** - Глобальные настройки пользователя
2. **notification_type_settings** - Настройки per event type
3. **user_telegram_connections** - Привязка Telegram аккаунтов

```sql
-- Глобальные настройки
CREATE TABLE IF NOT EXISTS user_notification_preferences (
    user_id BIGINT PRIMARY KEY,
    preferred_locale VARCHAR(5) DEFAULT 'sr',
    email_enabled BOOLEAN DEFAULT TRUE,
    telegram_enabled BOOLEAN DEFAULT TRUE,
    sms_enabled BOOLEAN DEFAULT FALSE,
    push_enabled BOOLEAN DEFAULT TRUE,
    dnd_start TIME,                    -- Do Not Disturb начало (время дня)
    dnd_end TIME,                      -- Do Not Disturb конец
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Per-event-type настройки
CREATE TABLE IF NOT EXISTS notification_type_settings (
    user_id BIGINT NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    email_enabled BOOLEAN DEFAULT TRUE,
    telegram_enabled BOOLEAN DEFAULT TRUE,
    sms_enabled BOOLEAN DEFAULT FALSE,
    push_enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (user_id, event_type)
);

-- Telegram connections
CREATE TABLE IF NOT EXISTS user_telegram_connections (
    user_id BIGINT PRIMARY KEY,
    telegram_chat_id BIGINT NOT NULL UNIQUE,
    telegram_username VARCHAR(255),
    connected_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_telegram_chat_id ON user_telegram_connections(telegram_chat_id);
```

### 003_delivery_log.up.sql

**Таблица:** `delivery_log` - Журнал попыток доставки

```sql
CREATE TABLE IF NOT EXISTS delivery_log (
    id BIGSERIAL PRIMARY KEY,
    notification_id BIGINT REFERENCES notifications(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL,
    channel VARCHAR(20) NOT NULL,      -- email/telegram/sms/push/in_app
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    provider_message_id VARCHAR(255),  -- ID от Zoho/Telegram
    error_message TEXT,
    retry_count INT DEFAULT 0,
    sent_at TIMESTAMP WITH TIME ZONE,
    delivered_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Индексы для эффективных запросов
CREATE INDEX idx_delivery_log_notification ON delivery_log(notification_id);
CREATE INDEX idx_delivery_log_user ON delivery_log(user_id);
CREATE INDEX idx_delivery_log_status ON delivery_log(status) WHERE status = 'pending';
CREATE INDEX idx_delivery_log_retry ON delivery_log(status, retry_count) WHERE status = 'failed';
CREATE INDEX idx_delivery_log_created ON delivery_log(created_at DESC);
```

---

## Roadmap

- [ ] FCM Push notifications
- [ ] SMS провайдер (Infobip/Twilio)
- [ ] WebSocket для real-time in-app updates
- [ ] A/B testing для шаблонов
- [ ] Analytics dashboard
- [ ] Email open/click tracking
- [ ] Scheduled notifications (cron-based)

---

## Связанные документы

- [TZ_NOTIFICATION_MICROSERVICE.md](/p/github.com/vondi-global/docs/TZ_NOTIFICATION_MICROSERVICE.md) - Техническое задание
- [SYSTEM_PASSPORT.md](/p/github.com/vondi-global/SYSTEM_PASSPORT.md) - Архитектура всей системы
- [ZOHO_MAIL_API_INTEGRATION_SPEC.md](/p/github.com/vondi-global/docs/ZOHO_MAIL_API_INTEGRATION_SPEC.md) - Интеграция Zoho Mail
- [TELEGRAM_BOT_INTEGRATION_SPEC.md](/p/github.com/vondi-global/docs/TELEGRAM_BOT_INTEGRATION_SPEC.md) - Интеграция Telegram Bot
- [TRANSLATION_SYSTEM_SPEC.md](/p/github.com/vondi-global/docs/TRANSLATION_SYSTEM_SPEC.md) - Система переводов

---

**Версия паспорта:** 1.0.0
**Дата обновления:** 2025-12-21
**Авторы:** Notification Service Team
