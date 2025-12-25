# Auth Service - Паспорт микросервиса

**Версия:** 1.21.2
**Статус:** Production Ready
**Обновлено:** 2025-12-21
**Репозиторий:** `github.com/vondi-global/auth`
**Локальная директория:** `/p/github.com/vondi-global/auth`

---

## 1. Обзор

Auth Service - внутренний микросервис аутентификации и авторизации для платформы Vondi. Предоставляет централизованное управление пользователями, JWT токенами, ролями и разрешениями.

### Архитектура

**DDD (Domain-Driven Design)** с чистой архитектурой:
- `internal/domain/` - доменные сущности (User, Role, Token, Session)
- `internal/application/` - use cases и бизнес-логика
- `internal/infrastructure/` - репозитории, внешние сервисы
- `internal/grpc/` - gRPC сервер и handlers
- `internal/http/` - HTTP/REST API (Fiber framework)
- `pkg/` - публичная библиотека для интеграции

### Технологии

- **Язык:** Go 1.23
- **База данных:** PostgreSQL 15 (auth_db, порт 25432)
- **Кэш:** Redis 7 (порт 26379)
- **Web Framework:** Fiber v2
- **gRPC:** Protocol Buffers v3
- **JWT:** RS256 (RSA-SHA256)
- **OAuth:** Google, Firebase Phone Authentication

### Порты

| Сервис | Порт | Описание |
|--------|------|----------|
| HTTP API | 28086 | REST API endpoints |
| gRPC | 20053 | gRPC service |
| Metrics | 29090 | Prometheus metrics |
| PostgreSQL | 25432 | База данных |
| Redis | 26379 | Кэш и сессии |

### Ключевые особенности

1. **Внутренний сервис** - НЕ доступен напрямую из интернета
2. **Dual Protocol** - HTTP и gRPC для разных сценариев
3. **Публичная библиотека** - `pkg/` для упрощённой интеграции
4. **RBAC** - 28 ролей, 67 разрешений
5. **Service-to-Service Auth** - API keys для микросервисов
6. **Worker Authentication** - специальные токены для warehouse/delivery workers
7. **OAuth 2.0** - Google, Firebase Phone
8. **JWT RS256** - асимметричное шифрование токенов

---

## 2. Конфигурация

Все переменные окружения используют префикс `VONDIAUTH_`.

### 2.1 Service

```bash
VONDIAUTH_SERVICE_NAME=auth-service
VONDIAUTH_SERVICE_VERSION=1.21.2      # Автоматически из internal/version
VONDIAUTH_SERVICE_ENV=development     # development | production | preprod
VONDIAUTH_SERVICE_LOG_LEVEL=debug     # debug | info | warn | error
VONDIAUTH_SERVICE_LOG_FORMAT=json     # json | text
```

### 2.2 Server

```bash
VONDIAUTH_SERVER_HTTP_PORT=28086
VONDIAUTH_SERVER_GRPC_PORT=20053
VONDIAUTH_SERVER_METRICS_PORT=29090
VONDIAUTH_SERVER_READ_TIMEOUT=30s
VONDIAUTH_SERVER_WRITE_TIMEOUT=30s
VONDIAUTH_SERVER_IDLE_TIMEOUT=120s
```

### 2.3 Database

```bash
VONDIAUTH_DB_HOST=localhost
VONDIAUTH_DB_PORT=25432
VONDIAUTH_DB_NAME=auth_db
VONDIAUTH_DB_USER=auth_user
VONDIAUTH_DB_PASSWORD=AuthP@ssw0rd2025!
VONDIAUTH_DB_SSL_MODE=disable
VONDIAUTH_DB_MAX_CONNECTIONS=100
VONDIAUTH_DB_MAX_IDLE_CONNECTIONS=10
VONDIAUTH_DB_CONNECTION_MAX_LIFETIME=1h
VONDIAUTH_DB_MIGRATION_PATH=migrations
```

**DSN:** `host=localhost port=25432 user=auth_user password=*** dbname=auth_db sslmode=disable`

### 2.4 Redis

```bash
VONDIAUTH_REDIS_HOST=localhost
VONDIAUTH_REDIS_PORT=26379
VONDIAUTH_REDIS_PASSWORD=RedisP@ssw0rd2025!
VONDIAUTH_REDIS_DB=0
VONDIAUTH_REDIS_MAX_RETRIES=3
VONDIAUTH_REDIS_POOL_SIZE=100
VONDIAUTH_REDIS_DIAL_TIMEOUT=5s
```

### 2.5 JWT

```bash
VONDIAUTH_JWT_ALGORITHM=RS256
VONDIAUTH_JWT_ACCESS_TOKEN_DURATION=49h        # 2 дня (default 15m)
VONDIAUTH_JWT_REFRESH_TOKEN_DURATION=720h      # 30 дней
VONDIAUTH_JWT_ISSUER=https://auth.vondi.rs
VONDIAUTH_JWT_AUDIENCE=https://vondi.rs
VONDIAUTH_JWT_PRIVATE_KEY_PATH=keys/private.pem
VONDIAUTH_JWT_PUBLIC_KEY_PATH=keys/public.pem
```

**Генерация ключей:**
```bash
cd /p/github.com/vondi-global/auth
make keys
```

### 2.6 OAuth

```bash
# Google OAuth 2.0
VONDIAUTH_OAUTH_GOOGLE_CLIENT_ID=your_google_client_id
VONDIAUTH_OAUTH_GOOGLE_CLIENT_SECRET=your_google_client_secret

# Firebase Phone Authentication
VONDIAUTH_FIREBASE_PROJECT_ID=vondi-global
VONDIAUTH_FIREBASE_SERVICE_ACCOUNT_KEY_PATH=keys/firebase-service-account.json
```

### 2.7 Security

```bash
VONDIAUTH_BCRYPT_COST=12                        # Стоимость хеширования паролей
VONDIAUTH_RATE_LIMIT_ENABLED=true
VONDIAUTH_RATE_LIMIT_LOGIN=5/15m                # 5 попыток за 15 минут
VONDIAUTH_RATE_LIMIT_REGISTER=3/1h              # 3 регистрации за час
VONDIAUTH_RATE_LIMIT_TOKEN_REFRESH=30/1m        # 30 refresh за минуту
VONDIAUTH_ALLOWED_ORIGINS=http://localhost:3000,https://vondi.rs,https://dev.vondi.rs
VONDIAUTH_ENABLE_CSRF=true
VONDIAUTH_CSRF_TOKEN_DURATION=24h
```

### 2.8 Fixtures

```bash
VONDIAUTH_FIXTURES_ENABLED=true                 # Загружать тестовые данные
VONDIAUTH_FIXTURES_PATH=fixtures
```

### 2.9 Monitoring

```bash
VONDIAUTH_METRICS_ENABLED=true
VONDIAUTH_TRACING_ENABLED=false
VONDIAUTH_TRACING_ENDPOINT=http://jaeger:14268/api/traces
```

---

## 3. Доменная модель

### 3.1 User (Пользователь)

**Файл:** `internal/domain/user.go`

**Структура:**
```go
type User struct {
    ID              int
    UUID            uuid.UUID
    Email           string
    EmailNormalized string
    Name            string
    PasswordHash    *string

    // OAuth providers
    GoogleID   *string
    FacebookID *string
    AppleID    *string

    // Profile
    PictureURL    *string
    Phone         *string
    PhoneVerified bool
    Bio           *string
    Timezone      string
    City          *string
    Country       *string

    // Security
    EmailVerified    bool
    TwoFactorEnabled bool
    TwoFactorSecret  *string
    BackupCodes      []string

    // Status
    Provider            UserProvider  // local | google | facebook | apple | service
    IsActive            bool
    IsDeleted           bool
    DeletedAt           *time.Time
    TermAccepted        bool
    FailedLoginAttempts int
    LockedUntil         *time.Time

    // Metadata
    LastLoginAt        *time.Time
    LastLoginIP        *string
    LastLoginUserAgent *string

    // Timestamps
    CreatedAt time.Time
    UpdatedAt time.Time

    // Roles
    Roles []string
}
```

**Методы:**
- `IsLocked() bool` - проверка блокировки аккаунта
- `IncrementFailedAttempts()` - увеличение счётчика неудачных попыток
- `LockAccount(duration)` - блокировка на время
- `UnlockAccount()` - разблокировка
- `RecordLogin(ip, userAgent)` - запись успешного входа
- `SoftDelete()` - мягкое удаление
- `HasRole(role) bool` - проверка наличия роли
- `IsAdmin() bool` - проверка админских прав

**Таблица:** `auth.users` (90 пользователей, 416 KB)

### 3.2 RefreshToken

**Файл:** `internal/domain/token.go`

**Структура:**
```go
type RefreshToken struct {
    ID          int
    UserID      int
    TokenHash   string
    JTI         string
    FamilyID    string
    DeviceID    *string
    DeviceName  *string
    IPAddress   *string
    UserAgent   *string
    ExpiresAt   time.Time
    IsRevoked   bool
    RevokedAt   *time.Time
    CreatedAt   time.Time
}
```

**Таблица:** `auth.refresh_tokens` (193 токена, 360 KB)

**Особенности:**
- Token Rotation - каждый refresh создаёт новый токен
- Family ID - отслеживание цепочки refresh токенов
- Device tracking - привязка к устройству

### 3.3 Session

**Файл:** `internal/domain/session.go`

**Структура:**
```go
type UserSession struct {
    ID             int
    UserID         int
    SessionID      string
    DeviceID       *string
    DeviceName     *string
    DeviceType     *string
    IPAddress      *string
    UserAgent      *string
    Location       *string
    IsActive       bool
    LastActivityAt time.Time
    CreatedAt      time.Time
    ExpiresAt      time.Time
}
```

**Таблица:** `auth.user_sessions` (0 записей - функционал отключён)

### 3.4 ServiceAccount

**Файл:** `internal/domain/service_account.go`

**Структура:**
```go
type ServiceAccount struct {
    ID              int
    UUID            uuid.UUID
    ServiceName     string
    Description     *string
    APIKey          string
    IsActive        bool
    ExpiresAt       *time.Time
    LastUsedAt      *time.Time
    CreatedAt       time.Time
    UpdatedAt       time.Time
}
```

**Таблица:** `auth.service_accounts` (4 аккаунта, 112 KB)

**Назначение:** API keys для микросервисов (tasktracker, payment, warehouse, delivery)

### 3.5 Role (Роль)

**Файл:** `pkg/entity/roles.go`

**28 системных ролей:**

#### Административные
- `super_admin` - суперадминистратор
- `admin` - администратор

#### Модерация
- `moderator`, `content_moderator`, `review_moderator`, `chat_moderator`, `dispute_manager`

#### Бизнес
- `vendor_manager`, `category_manager`, `marketing_manager`, `financial_manager`

#### Операции
- `warehouse_manager`, `warehouse_worker`, `pickup_manager`, `pickup_worker`, `courier`

#### Поддержка
- `support_l1`, `support_l2`, `support_l3`

#### Комплаенс
- `legal_advisor`, `compliance_officer`

#### Продавцы
- `professional_vendor`, `vendor`, `individual_seller`, `storefront_staff`

#### Покупатели
- `verified_buyer`, `vip_customer`, `user`

#### Аналитика
- `data_analyst`, `business_analyst`

#### Служебные
- `service`

### 3.6 Permission (Разрешение)

**67 системных разрешений** (см. `pkg/entity/roles.go`):

**Категории:**
- User Management (8): `users.view`, `users.edit`, `users.delete`, ...
- Admin Panel (7): `admin.access`, `admin.users`, `admin.categories`, ...
- Listings (10): `listings.create`, `listings.edit_own`, `listings.moderate`, ...
- Orders (7): `orders.view_all`, `orders.process`, `orders.refund`, ...
- Payments (5): `payments.view`, `payments.process`, `payments.withdraw`, ...
- Messaging (3): `messaging.view`, `messaging.moderate`, ...
- Reviews (3): `reviews.moderate`, `reviews.delete`, `reviews.respond`
- Storefronts (5): `storefront.create`, `storefront.verify`, ...
- Warehouse (4): `warehouse.view_stock`, `warehouse.ship_orders`, ...
- Pickup (3): `pickup.manage_points`, `pickup.receive_items`, ...
- Delivery (3): `delivery.assign_routes`, `delivery.update_status`, ...
- Categories (4): `categories.create`, `categories.edit`, ...
- Marketing (4): `marketing.campaigns`, `marketing.promotions`, ...
- System (6): `system.view_logs`, `system.manage_settings`, ...
- Analytics (3): `analytics.view_reports`, `analytics.export_data`, ...
- Support (3): `support.view_tickets`, `support.resolve_tickets`, ...
- Financial (4): `financial.view_reports`, `financial.manage_payouts`, ...
- Compliance (3): `compliance.view_reports`, `compliance.manage_kyc`, ...

---

## 4. gRPC API

**Файл:** `api/proto/auth/v1/auth.proto`
**Package:** `vondi.auth.v1`

### 4.1 Методы (25 штук)

**Аутентификация (6):**
1. `Register(RegisterRequest) → RegisterResponse`
2. `Login(LoginRequest) → LoginResponse`
3. `RefreshToken(RefreshTokenRequest) → RefreshTokenResponse`
4. `ValidateToken(ValidateTokenRequest) → ValidateTokenResponse`
5. `Logout(LogoutRequest) → LogoutResponse`
6. `LogoutAll(LogoutAllRequest) → LogoutAllResponse`

**OAuth (2):**
7. `GenerateOAuthURL(GenerateOAuthURLRequest) → GenerateOAuthURLResponse`
8. `ExchangeOAuthCode(ExchangeOAuthCodeRequest) → ExchangeOAuthCodeResponse`

**Пользователи (7):**
9. `GetAllUsers(GetAllUsersRequest) → GetAllUsersResponse`
10. `GetUser(GetUserRequest) → GetUserResponse`
11. `GetUserByEmail(GetUserByEmailRequest) → GetUserByEmailResponse`
12. `UpdateUserProfile(UpdateUserProfileRequest) → UpdateUserProfileResponse`
13. `UpdateUserStatus(UpdateUserStatusRequest) → UpdateUserStatusResponse`
14. `DeleteUser(DeleteUserRequest) → DeleteUserResponse`
15. `IsUserAdmin(IsUserAdminRequest) → IsUserAdminResponse`

**Роли (5):**
16. `GetUserRoles(GetUserRolesRequest) → GetUserRolesResponse`
17. `AddUserRole(AddUserRoleRequest) → AddUserRoleResponse`
18. `RemoveUserRole(RemoveUserRoleRequest) → RemoveUserRoleResponse`
19. `GetAllRoles(GetAllRolesRequest) → GetAllRolesResponse`
20. `GetUsersByRole(GetUsersByRoleRequest) → GetUsersByRoleResponse`

**Служебные (3):**
21. `ExchangeAPIKey(ExchangeAPIKeyRequest) → ExchangeAPIKeyResponse` - S2S auth
22. `CreateWorkerToken(CreateWorkerTokenRequest) → CreateWorkerTokenResponse` - WMS workers
23. `GetPublicKey(GetPublicKeyRequest) → GetPublicKeyResponse` - JWT public key

### 4.2 Примеры

**ValidateToken:**
```protobuf
message ValidateTokenRequest {
  string access_token = 1;
}

message ValidateTokenResponse {
  bool valid = 1;
  optional string error = 2;
  optional int32 user_id = 3;
  optional string email = 4;
  repeated string roles = 5;
  optional int64 expires_at = 6;
  map<string, string> claims = 7;
}
```

**CreateWorkerToken:**
```protobuf
message CreateWorkerTokenRequest {
  int64 worker_id = 1;
  string worker_code = 2;
  string worker_name = 3;
  int64 warehouse_id = 4;
  string worker_role = 5;  // picker | packer | supervisor | manager
}
```

---

## 5. HTTP API

**Базовый путь:** `/api/v1`
**Порт:** 28086

### 5.1 Endpoints

**Health & Info:**
```
GET  /health               - Health check
GET  /api/v1/version       - Версия сервиса
GET  /api/v1/public-key    - Публичный RSA ключ
```

**Аутентификация:**
```
POST /api/v1/auth/register           - Регистрация
POST /api/v1/auth/login              - Вход
POST /api/v1/auth/refresh            - Обновление токенов
POST /api/v1/auth/logout             - Выход
POST /api/v1/auth/logout-all         - Выход со всех устройств
GET  /api/v1/auth/validate           - Валидация токена
GET  /api/v1/auth/me                 - Текущий пользователь
```

**OAuth:**
```
GET  /api/v1/auth/oauth/google           - OAuth URL для Google
GET  /api/v1/auth/oauth/google/callback  - Callback для Google
```

**Пользователи (требуется авторизация):**
```
GET    /api/v1/users                  - Список всех
GET    /api/v1/users/:id              - Получить по ID
GET    /api/v1/users/email/:email     - Получить по email
PUT    /api/v1/users/:id              - Обновить профиль
PATCH  /api/v1/users/:id/status       - Обновить статус
DELETE /api/v1/users/:id              - Удалить
GET    /api/v1/users/:id/is-admin     - Проверка admin
```

**Роли:**
```
GET    /api/v1/users/:id/roles        - Роли пользователя
POST   /api/v1/users/:id/roles        - Добавить роль
DELETE /api/v1/users/:id/roles/:role  - Удалить роль
GET    /api/v1/roles                  - Список ролей
GET    /api/v1/roles/:role/users      - Пользователи с ролью
```

**Service Accounts:**
```
POST /api/v1/service/exchange-api-key  - Обмен API key на JWT
```

### 5.2 Примеры

**Login:**
```bash
curl -X POST http://localhost:28086/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "SecurePass123!"}'
```

**Validate:**
```bash
curl -X GET http://localhost:28086/api/v1/auth/validate \
  -H "Authorization: Bearer <token>"
```

---

## 6. Аутентификация

### 6.1 JWT Параметры

**Алгоритм:** RS256 (RSA-SHA256)
**Ключи:** 2048-bit RSA keypair

**Access Token:**
- Срок жизни: 49 часов (2 дня)
- Формат: `Bearer <token>`
- Содержит: user_id, email, roles, permissions

**Refresh Token:**
- Срок жизни: 720 часов (30 дней)
- Token Rotation - каждый refresh создаёт новый
- Family ID для отслеживания цепочки

**Claims:**
```json
{
  "iss": "https://auth.vondi.rs",
  "aud": "https://vondi.rs",
  "sub": "user@example.com",
  "user_id": 123,
  "email": "user@example.com",
  "name": "John Doe",
  "roles": ["user", "vendor"],
  "permissions": ["listings.create"],
  "provider": "google",
  "email_verified": true,
  "exp": 1735123456,
  "iat": 1735000000,
  "jti": "unique-id"
}
```

### 6.2 OAuth Flow

**Google OAuth:**
1. GET `/api/v1/auth/oauth/google?redirect_uri=...&state=...`
2. Redirect на Google
3. Google callback с `code`
4. GET `/api/v1/auth/oauth/google/callback?code=...&state=...`
5. Ответ с access_token, refresh_token, user

**Firebase Phone:**
- Верификация через Firebase Admin SDK
- Требует service account key

### 6.3 Rate Limiting

- **Login:** 5 попыток / 15 минут
- **Register:** 3 регистрации / 1 час
- **Token Refresh:** 30 обновлений / 1 минуту
- **Account Lock:** 5 неудачных попыток → блокировка 15 минут

---

## 7. Авторизация (RBAC)

### 7.1 Middleware

**Пример использования:**
```go
import authmiddleware "github.com/vondi-global/auth/pkg/http/fiber"

// Требуется авторизация
app.Use(authmiddleware.RequireAuth())

// Требуется роль admin
app.Use(authmiddleware.RequireAuth(entity.RoleAdmin))

// В хендлере
userID, _ := authmiddleware.GetUserID(c)
roles, _ := authmiddleware.GetRoles(c)
```

### 7.2 Проверка разрешений

```go
if !user.HasPermission(entity.PermListingsEditAny) {
    return fiber.ErrForbidden
}
```

### 7.3 Role Hierarchy

**Приоритеты:**
1. `super_admin` (100) - все разрешения
2. `admin` (90) - большинство разрешений
3. `moderator` (80)
4. `warehouse_manager` (60)
5. `vendor` (40)
6. `user` (10) - базовая роль

---

## 8. Миграции

**Директория:** `migrations/`
**Всего:** 9 миграций

### 8.1 Список

1. `0001_functions` - SQL функции
2. `0002_users` - Таблица users (33 колонки)
3. `0003_refresh_tokens` - Таблица refresh_tokens
4. `0004_user_sessions` - Таблица user_sessions
5. `0005_revoked_access_tokens` - Таблица revoked_access_tokens
6. `0006_user_profile_extended` - Расширение профиля
7. `0007_create_service_accounts` - Таблица service_accounts
8. `0008_user_addresses` - Таблица user_addresses
9. `0009_user_checkout_profiles` - Таблица user_checkout_profiles

### 8.2 Применение

```bash
cd /p/github.com/vondi-global/auth
make migrate-up           # Применить
make migrate-down         # Откатить
make migrate-reset        # Пересоздать
```

### 8.3 Статус

| Таблица | Записей | Размер | Статус |
|---------|---------|--------|--------|
| `users` | 90 | 416 KB | ✅ Активна |
| `refresh_tokens` | 193 | 360 KB | ✅ Активна |
| `service_accounts` | 4 | 112 KB | ✅ Активна |
| `user_addresses` | 3 | 152 KB | ✅ Используется |
| `revoked_access_tokens` | 2 | 64 KB | ✅ Используется |
| `user_sessions` | 0 | 64 KB | ⚠️ Не используется |
| `user_checkout_profiles` | 0 | 56 KB | ⚠️ Не используется |

---

## 9. События

Auth Service **НЕ публикует события** в очереди.

**Причина:** Внутренний сервис, события обрабатываются вызывающим сервисом.

**Паттерн:**
1. Backend → auth.Register() → Success
2. Backend → events.Publish("user.registered")
3. Backend → email.SendWelcome()

---

## 10. Публичная библиотека (pkg)

**Структура:**
```
pkg/
├── service/      # Protocol-agnostic
├── entity/       # Модели
├── http/         # HTTP client + Fiber middleware
├── grpc/         # gRPC client + interceptors
└── proto/        # Protobuf
```

**Использование:**
```go
import "github.com/vondi-global/auth/pkg/service"

authService, _ := service.NewAuthService(&service.Config{
    HTTPURL: "http://localhost:28086",
    GRPCURL: "localhost:20053",
})

resp, _ := authService.Login(ctx, entity.UserLoginRequest{
    Email:    "user@example.com",
    Password: "SecurePass123!",
})
```

---

## 11. Запуск

**Локально:**
```bash
cd /p/github.com/vondi-global/auth
docker compose up -d postgres redis
make migrate-up
screen -dmS auth-service bash -c './bin/auth-service'
```

**Проверка:**
```bash
curl http://localhost:28086/health
curl http://localhost:28086/api/v1/version
```

**Сборка:**
```bash
make build          # Сборка
make test           # Тесты
make lint           # Линтинг
make check-total    # Полная проверка
```

---

## 12. Интеграция

### 12.1 Vondi Монолит
- Через HTTP API
- BFF proxy `/api/v2/auth/*`

### 12.2 WMS
- gRPC + Worker Tokens
- `CreateWorkerToken()` для работников

### 12.3 Микросервисы
- gRPC + Service Accounts
- Автоматическое управление токенами через `ServiceTokenManager`

---

## 13. Известные проблемы

1. **Пустые таблицы:** user_sessions, user_checkout_profiles, audit_log
2. **Дублирующиеся индексы:** 17 штук (~240 KB)
3. **NULL колонки:** 26 колонок всегда NULL

**Рекомендации:**
- Удалить дублирующиеся индексы
- Реализовать или удалить пустые таблицы
- Начать использовать NULL колонки или удалить

---

## 14. Безопасность

**Реализовано:**
- ✅ JWT RS256
- ✅ Bcrypt (cost 12)
- ✅ Rate limiting
- ✅ CSRF защита
- ✅ Account locking
- ✅ Token rotation

**Критично:**
- ❌ НЕ публиковать `keys/private.pem`
- ❌ НЕ логировать пароли/токены
- ❌ НЕ делать сервис публичным

---

## 15. Дополнительно

**Документация:**
- `README.md` - Быстрый старт
- `CLAUDE.md` - Руководство разработки
- `docs/ARCHITECTURE.md` - Архитектура
- `pkg/README.md` - Использование библиотеки
- `AUTH_DB_AUDIT_REPORT.md` - Аудит БД

**Примеры:**
- `examples/tasktracker/` - Пример приложения
- `examples/payment-service/` - S2S auth

**Git:**
- Org: `github.com/vondi-global`
- Repo: `github.com/vondi-global/auth`
- Workflow: Только через Pull Requests

---

**Владелец:** VONDI GLOBAL D.O.O., Beograd, Srbija
**Генеральный директор:** Malden Simić
