# Users Domain - Паспорт домена

**Версия:** 1.1.0
**Статус:** Production Ready
**Обновлено:** 2025-12-21
**Владелец:** Auth Service + Monolith
**Репозитории:**
- `github.com/vondi-global/auth` (Auth Service)
- `github.com/vondi-global/vondi` (Monolith)

---

## 1. Обзор домена

Users Domain - центральный домен платформы Vondi, отвечающий за управление пользователями, аутентификацию, авторизацию и контроль доступа.

### 1.1 Архитектура

Users domain **распределен между двумя системами**:

**Auth Service** (микросервис `/p/github.com/vondi-global/auth`):
- PostgreSQL порт: `25432` → БД: `auth_db` (schema: `auth`)
- HTTP API: `http://localhost:28086`
- gRPC API: `localhost:20053`
- **Ответственность:** Аутентификация, JWT токены, роли, пользователи (core identity)

**Monolith** (`/p/github.com/vondi-global/vondi`):
- PostgreSQL порт: `5433` → БД: `vondi_db` (schema: `public`)
- HTTP API: `http://localhost:3000`
- **Ответственность:** Расширенный профиль, настройки, чат, витрины

### 1.2 Принципы взаимодействия

```
Browser → /api/v2/* (Next.js BFF) → Backend → Auth Service (Internal API)
         └─ httpOnly cookies       └─ REST/gRPC
```

**КРИТИЧЕСКИ ВАЖНО:**
- Auth Service - внутренний API микросервис
- Frontend НИКОГДА не обращается напрямую к Auth Service
- Вся аутентификация через Backend → Auth Service
- JWT токены в httpOnly cookies (BFF proxy)

### 1.3 Основные функции

- **Identity Management** - управление идентификацией пользователей
- **Authentication** - проверка подлинности (OAuth, Email/Password, Phone)
- **Authorization** - контроль доступа через RBAC (28 ролей, 67 разрешений)
- **Session Management** - управление сессиями и токенами
- **User Profiles** - профили пользователей и адреса доставки
- **Service Accounts** - аутентификация микросервисов (S2S)
- **Worker Authentication** - специальные токены для WMS/Delivery workers

### 1.4 Технологии

- **Язык:** Go 1.23
- **База данных:** PostgreSQL 15
  - Auth Service: `auth_db` (port 25432, schema: `auth`)
  - Monolith: `vondi_db` (port 5433, schema: `public`)
- **Кэш:** Redis 7 (port 26379)
- **Framework:** Fiber v2 (HTTP), gRPC (Protocol Buffers v3)
- **JWT:** RS256 (RSA-SHA256, 2048-bit keys)
- **OAuth:** Google, Facebook, Apple, Firebase Phone
- **Хеширование:** Bcrypt (cost 12)

---

## 2. Entities (Доменные сущности)

### 2.1 User (Auth Service)

**Файл:** `/p/github.com/vondi-global/auth/internal/domain/user.go`
**Таблица:** `auth.users` (90 пользователей, 416 KB)

**Структура:**

```go
type User struct {
    // Идентификация
    ID              int       `json:"id"`
    UUID            uuid.UUID `json:"uuid"`
    Email           string    `json:"email"`
    EmailNormalized string    `json:"-"`
    Name            string    `json:"name"`
    PasswordHash    *string   `json:"-"`

    // OAuth providers
    GoogleID   *string `json:"-"`
    FacebookID *string `json:"-"`
    AppleID    *string `json:"-"`

    // Профиль
    PictureURL    *string `json:"picture_url,omitempty"`
    Phone         *string `json:"phone,omitempty"`
    PhoneVerified bool    `json:"phone_verified"`
    Bio           *string `json:"bio,omitempty"`
    Timezone      string  `json:"timezone"`  // default: "UTC"
    City          *string `json:"city,omitempty"`
    Country       *string `json:"country,omitempty"`

    // Безопасность
    EmailVerified    bool     `json:"email_verified"`
    TwoFactorEnabled bool     `json:"two_factor_enabled"`
    TwoFactorSecret  *string  `json:"-"`
    BackupCodes      []string `json:"-"`

    // Статус
    Provider     UserProvider `json:"provider"` // local | google | facebook | apple | service
    IsActive     bool         `json:"is_active"`
    IsDeleted    bool         `json:"-"`
    DeletedAt    *time.Time   `json:"-"`
    TermAccepted bool         `json:"term_accepted"`

    // Security Metadata
    LastLoginAt         *time.Time `json:"last_login_at,omitempty"`
    LastLoginIP         *string    `json:"-"`
    LastLoginUserAgent  *string    `json:"-"`
    FailedLoginAttempts int        `json:"-"`
    LockedUntil         *time.Time `json:"-"`

    // Временные метки
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`

    // Роли (загружаются из связанной таблицы)
    Roles []string `json:"roles,omitempty"`
}
```

**UserProvider enum:**

```go
type UserProvider string

const (
    ProviderLocal    UserProvider = "local"    // Email/Password
    ProviderGoogle   UserProvider = "google"   // Google OAuth
    ProviderFacebook UserProvider = "facebook" // Facebook OAuth
    ProviderApple    UserProvider = "apple"    // Apple OAuth
    ProviderService  UserProvider = "service"  // Service Account
)
```

**Методы:**

```go
// Security
func (u *User) IsLocked() bool                      // Проверка блокировки
func (u *User) IncrementFailedAttempts()             // +1 attempt, auto-lock at 5
func (u *User) LockAccount(duration time.Duration)   // Блокировка на время
func (u *User) UnlockAccount()                       // Разблокировка + reset counter
func (u *User) RecordLogin(ip, userAgent string)     // Запись успешного входа

// Lifecycle
func (u *User) SoftDelete()                          // Мягкое удаление

// Authorization
func (u *User) HasRole(role string) bool             // Проверка роли
func (u *User) IsAdmin() bool                        // Проверка admin/super_admin
func (u *User) IsModerator() bool                    // Проверка moderator roles
func (u *User) IsSupport() bool                      // Проверка support roles

// Conversion
func (u *User) ToProfile() *entity.UserProfile       // → публичный профиль
```

**Индексы:**

```sql
-- Уникальные
CREATE UNIQUE INDEX idx_users_email ON auth.users(email_normalized);
CREATE UNIQUE INDEX idx_users_uuid ON auth.users(uuid);
CREATE UNIQUE INDEX idx_users_google_id ON auth.users(google_id) WHERE google_id IS NOT NULL;

-- Обычные
CREATE INDEX idx_users_provider ON auth.users(provider);
CREATE INDEX idx_users_created_at ON auth.users(created_at DESC);
CREATE INDEX idx_users_last_login ON auth.users(last_login_at DESC);
```

**Особенности:**

- **Account Locking:** 5 неудачных попыток → блокировка 15 минут
- **Soft Delete:** IsDeleted flag вместо физического удаления
- **OAuth Users:** password_hash = NULL, email_verified = true
- **Admin Detection:** Автоматическая проверка в `admin_users` table при первом входе

### 2.2 RefreshToken (Auth Service)

**Файл:** `/p/github.com/vondi-global/auth/internal/domain/token.go`
**Таблица:** `auth.refresh_tokens` (193 токена, 360 KB)

**Структура:**

```go
type RefreshToken struct {
    ID        int       `json:"-"`
    JTI       uuid.UUID `json:"jti"`          // Unique token ID
    UserID    int       `json:"user_id"`
    TokenHash string    `json:"-"`            // SHA256 hash

    // Token Family (Refresh Token Rotation)
    FamilyID      uuid.UUID `json:"family_id"`      // Семья токенов
    ParentTokenID *int64    `json:"-"`              // Родительский токен

    // Device Tracking
    DeviceID   *string `json:"device_id,omitempty"`
    DeviceName *string `json:"device_name,omitempty"`  // "iPhone 13", "Chrome Desktop"
    IPAddress  *string `json:"ip_address,omitempty"`
    UserAgent  *string `json:"user_agent,omitempty"`

    // Expiration & Revocation
    ExpiresAt     time.Time  `json:"expires_at"`       // +30 дней
    IsRevoked     bool       `json:"is_revoked"`
    RevokedAt     *time.Time `json:"revoked_at,omitempty"`
    RevokedReason *string    `json:"revoked_reason,omitempty"`

    // Usage Tracking
    UsedAt   *time.Time `json:"used_at,omitempty"`
    UseCount int        `json:"use_count"`

    CreatedAt time.Time `json:"created_at"`
}
```

**Методы:**

```go
func (t *RefreshToken) IsExpired() bool      // Проверка истечения
func (t *RefreshToken) IsValid() bool        // !revoked && !expired
func (t *RefreshToken) Revoke(reason string) // Отзыв токена
func (t *RefreshToken) RecordUse()           // Запись использования
```

**Особенности:**

- **Token Rotation:** Каждый refresh создаёт новый токен, старый отзывается
- **Family Tracking:** Все токены одной цепочки имеют общий FamilyID
- **Reuse Detection:** Попытка использовать отозванный токен → отзыв всей семьи
- **Device Binding:** Токен привязан к устройству (опционально)

### 2.3 UserSession (Auth Service)

**Файл:** `/p/github.com/vondi-global/auth/internal/domain/session.go`
**Таблица:** `auth.user_sessions` (0 записей - функционал отключён)

**Структура:**

```go
type UserSession struct {
    ID             int       `json:"id"`
    SessionID      uuid.UUID `json:"session_id"`
    UserID         int       `json:"user_id"`
    RefreshTokenID *int64    `json:"-"`

    // Device Info
    DeviceID        *string `json:"device_id,omitempty"`
    DeviceName      *string `json:"device_name,omitempty"`
    DeviceType      *string `json:"device_type,omitempty"`    // mobile | desktop | tablet
    Browser         *string `json:"browser,omitempty"`        // Chrome, Safari, Firefox
    OS              *string `json:"os,omitempty"`             // iOS, Android, Windows
    IPAddress       *string `json:"ip_address,omitempty"`
    LocationCountry *string `json:"location_country,omitempty"`
    LocationCity    *string `json:"location_city,omitempty"`

    // Activity
    LastActivityAt time.Time `json:"last_activity_at"`
    ActivityCount  int       `json:"activity_count"`

    // Status
    IsActive          bool       `json:"is_active"`
    TerminatedAt      *time.Time `json:"terminated_at,omitempty"`
    TerminationReason *string    `json:"termination_reason,omitempty"`

    // Timestamps
    CreatedAt time.Time `json:"created_at"`
    ExpiresAt time.Time `json:"expires_at"`
}
```

**Статус:** Функционал НЕ используется. Отслеживание сессий через RefreshToken.

### 2.4 Address (Auth Service)

**Файл:** `/p/github.com/vondi-global/auth/internal/domain/address.go`
**Таблица:** `auth.user_addresses` (3 адреса, 152 KB)

**Структура:**

```go
type Address struct {
    ID     int64 `json:"id"`
    UserID int64 `json:"user_id"`

    // Метка адреса
    Label string `json:"label,omitempty"` // "Home", "Work", "Dacha"

    // Получатель
    RecipientName  string `json:"recipient_name"`
    RecipientPhone string `json:"recipient_phone"`

    // Адрес
    CountryCode string `json:"country_code"` // ISO 3166-1 (RS, RU, US)
    PostalCode  string `json:"postal_code"`

    // Многоязычные поля (sr_latin обязательно для Post Express)
    StreetSrLatin   string  `json:"street_sr_latin"`    // Обязательно
    CitySrLatin     string  `json:"city_sr_latin"`      // Обязательно
    DistrictSrLatin *string `json:"district_sr_latin,omitempty"`

    // English локализация
    StreetEn   *string `json:"street_en,omitempty"`
    CityEn     *string `json:"city_en,omitempty"`
    DistrictEn *string `json:"district_en,omitempty"`

    // User locale (оригинальный ввод пользователя)
    StreetUserLocale   *string `json:"street_user_locale,omitempty"`
    CityUserLocale     *string `json:"city_user_locale,omitempty"`
    DistrictUserLocale *string `json:"district_user_locale,omitempty"`
    UserLocale         *string `json:"user_locale,omitempty"` // sr_cyrillic, en, ru

    // Форматированные адреса
    FormattedSrLatin    *string `json:"formatted_sr_latin,omitempty"`
    FormattedEn         *string `json:"formatted_en,omitempty"`
    FormattedUserLocale *string `json:"formatted_user_locale,omitempty"`

    // Детали здания
    Building  *string `json:"building,omitempty"`
    Apartment *string `json:"apartment,omitempty"`
    Entrance  *string `json:"entrance,omitempty"`
    Floor     *string `json:"floor,omitempty"`
    Intercom  *string `json:"intercom,omitempty"`
    Notes     *string `json:"notes,omitempty"`

    // Геолокация (от Mapbox/geocoding)
    Latitude  *float64 `json:"latitude,omitempty"`
    Longitude *float64 `json:"longitude,omitempty"`

    // Валидация (от geocoding service)
    IsValidated      bool       `json:"is_validated"`
    ValidationSource *string    `json:"validation_source,omitempty"` // "mapbox", "post_express"
    ConfidenceScore  *float64   `json:"confidence_score,omitempty"`  // 0.0 - 1.0
    ValidatedAt      *time.Time `json:"validated_at,omitempty"`

    // Post Express интеграция (Serbian delivery)
    PostExpressSettlementID *int `json:"post_express_settlement_id,omitempty"`
    PostExpressStreetID     *int `json:"post_express_street_id,omitempty"`

    // Флаги
    IsDefault  bool `json:"is_default"`  // Адрес по умолчанию
    IsLastUsed bool `json:"is_last_used"` // Последний использованный

    // Soft Delete
    IsDeleted  bool       `json:"-"`
    DeletedAt  *time.Time `json:"-"`
    CreatedAt  time.Time  `json:"created_at"`
    UpdatedAt  time.Time  `json:"updated_at"`
    LastUsedAt *time.Time `json:"last_used_at,omitempty"`
}
```

**Методы:**

```go
func (a *Address) SoftDelete()  // Мягкое удаление
func (a *Address) MarkAsUsed()  // Отметить как последний использованный
```

**Особенности:**

- **Multilingual:** Поддержка 3 языков (sr_latin, en, user_locale)
- **Post Express Ready:** Интеграция с сербской почтой
- **Geocoding:** Автоматическая валидация через Mapbox
- **Smart Defaults:** Автоматическое определение default адреса

### 2.5 ServiceAccount (Auth Service)

**Файл:** `/p/github.com/vondi-global/auth/internal/domain/service_account.go`
**Таблица:** `auth.service_accounts` (4 аккаунта, 112 KB)

**Структура:**

```go
type ServiceAccount struct {
    ID              int        `json:"id"`
    UUID            uuid.UUID  `json:"uuid"`
    ServiceName     string     `json:"service_name"`     // Уникальное название
    Description     *string    `json:"description,omitempty"`
    APIKey          string     `json:"-"`                // Bcrypt hash
    IsActive        bool       `json:"is_active"`
    ExpiresAt       *time.Time `json:"expires_at,omitempty"` // NULL = бессрочно
    LastUsedAt      *time.Time `json:"last_used_at,omitempty"`
    CreatedAt       time.Time  `json:"created_at"`
    UpdatedAt       time.Time  `json:"updated_at"`
}
```

**Назначение:** API keys для service-to-service аутентификации микросервисов.

**Существующие Service Accounts:**

- `tasktracker` - TaskTracker service
- `payment` - Payment service
- `warehouse` - WMS service
- `delivery` - Delivery service

**Workflow:**

1. Микросервис обменивает API key на JWT: `POST /api/v1/service/exchange-api-key`
2. JWT токен содержит `service_name` в claims
3. Токен используется для S2S вызовов
4. Автоматическое обновление через `ServiceTokenManager` (pkg library)

---

## 3. Value Objects

### 3.1 Role (Роль)

**Файл:** `/p/github.com/vondi-global/auth/pkg/entity/roles.go`

**28 системных ролей:**

#### Административные (2)
```go
RoleSuperAdmin Role = "super_admin"  // Все разрешения
RoleAdmin      Role = "admin"        // Большинство разрешений
```

#### Модерация (5)
```go
RoleModerator        Role = "moderator"
RoleContentModerator Role = "content_moderator"
RoleReviewModerator  Role = "review_moderator"
RoleChatModerator    Role = "chat_moderator"
RoleDisputeManager   Role = "dispute_manager"
```

#### Бизнес (4)
```go
RoleVendorManager    Role = "vendor_manager"
RoleCategoryManager  Role = "category_manager"
RoleMarketingManager Role = "marketing_manager"
RoleFinancialManager Role = "financial_manager"
```

#### Операции (5)
```go
RoleWarehouseManager Role = "warehouse_manager"
RoleWarehouseWorker  Role = "warehouse_worker"
RolePickupManager    Role = "pickup_manager"
RolePickupWorker     Role = "pickup_worker"
RoleCourier          Role = "courier"
```

#### Поддержка (3)
```go
RoleSupportL1 Role = "support_l1"
RoleSupportL2 Role = "support_l2"
RoleSupportL3 Role = "support_l3"
```

#### Комплаенс (2)
```go
RoleLegalAdvisor      Role = "legal_advisor"
RoleComplianceOfficer Role = "compliance_officer"
```

#### Продавцы (4)
```go
RoleProfessionalVendor Role = "professional_vendor"
RoleVendor             Role = "vendor"
RoleIndividualSeller   Role = "individual_seller"
RoleStorefrontStaff    Role = "storefront_staff"
```

#### Покупатели (3)
```go
RoleVerifiedBuyer Role = "verified_buyer"
RoleVIPCustomer   Role = "vip_customer"
RoleUser          Role = "user"  // Базовая роль по умолчанию
```

#### Аналитика (2)
```go
RoleDataAnalyst     Role = "data_analyst"
RoleBusinessAnalyst Role = "business_analyst"
```

#### Служебные (1)
```go
RoleService Role = "service"  // Service accounts
```

### 3.2 Permission (Разрешение)

**67 системных разрешений:**

#### User Management (8)
```go
PermUsersView, PermUsersList, PermUsersEdit, PermUsersDelete
PermUsersBlock, PermUsersVerify, PermUsersAssignRole, PermUsersExport
```

#### Admin Panel (7)
```go
PermAdminAccess, PermAdminUsers, PermAdminCategories, PermAdminAttributes
PermAdminTranslations, PermAdminReports, PermAdminSettings
```

#### Listings (10)
```go
PermListingsCreate, PermListingsEditOwn, PermListingsEditAny
PermListingsDeleteOwn, PermListingsDeleteAny, PermListingsModerate
PermListingsViewAll, PermListingsApprove, PermListingsReject, PermListingsFeature
```

#### Orders (7), Payments (5), Messaging (3), Reviews (3), Storefronts (5)
#### Warehouse (4), Pickup (3), Delivery (3), Categories (4), Marketing (4)
#### System (6), Analytics (3), Support (3), Financial (4), Compliance (3)

**Полный список:** См. `/p/github.com/vondi-global/auth/pkg/entity/roles.go`

### 3.3 Priority (Приоритет роли)

```go
type Priority int

const (
    PrioritySuperAdmin       Priority = 1
    PriorityAdmin            Priority = 10
    PriorityModerator        Priority = 20
    PriorityManager          Priority = 30
    PrioritySupport          Priority = 40
    PriorityVendor           Priority = 50
    PriorityVerifiedCustomer Priority = 60
    PriorityUser             Priority = 100
)
```

**Использование:** Меньше число = выше приоритет

---

## 4. Use Cases

### 4.1 Registration (Регистрация)

**HTTP:** `POST /api/v1/auth/register` (Auth Service)
**gRPC:** `Register(RegisterRequest)`

**Request:**

```go
type RegisterRequest struct {
    Email         string `json:"email" validate:"required,email"`
    Password      string `json:"password" validate:"required,min=8"`
    Name          string `json:"name" validate:"required,min=2,max=100"`
    TermsAccepted bool   `json:"terms_accepted" validate:"required"`
}
```

**Flow:**

```
1. Validate email format, password strength, terms acceptance
2. Check if email already exists (email_normalized lowercase)
3. Hash password (bcrypt, cost 12)
4. Create User record (provider: local, IsActive: true)
5. Assign default role "user"
6. Generate JWT access token (49h) + refresh token (30d)
7. Store RefreshToken in DB
8. Return tokens + user profile
```

**Validations:**

- Email: RFC 5322, unique (normalized lowercase)
- Password: min 8 chars
- Name: min 2 chars, max 100 chars
- Terms: must be true

**Response:**

```json
{
  "access_token": "eyJhbGc...",
  "refresh_token": "uuid...",
  "token_type": "Bearer",
  "expires_in": 176400,
  "user": {
    "id": 123,
    "email": "user@example.com",
    "name": "John Doe",
    "roles": ["user"],
    "email_verified": false
  }
}
```

### 4.2 Login (Вход)

**HTTP:** `POST /api/v1/auth/login` (Auth Service)
**gRPC:** `Login(LoginRequest)`

**Request:**

```go
type LoginRequest struct {
    Email      string  `json:"email" validate:"required,email"`
    Password   string  `json:"password" validate:"required"`
    DeviceID   *string `json:"device_id,omitempty"`
    DeviceName *string `json:"device_name,omitempty"`
}
```

**Flow:**

```
1. Find user by email (normalized lowercase)
2. Check if user.IsLocked() (LockedUntil > now)
3. Verify password (bcrypt.CompareHashAndPassword)
4. On failure:
   - IncrementFailedAttempts()
   - If attempts >= 5: LockAccount(15 minutes)
   - Return 401 Unauthorized
5. On success:
   - RecordLogin(ip, userAgent) - reset failed attempts, update last_login
   - Load user roles
   - Generate JWT tokens (access + refresh)
   - Store RefreshToken with device tracking
   - Return tokens + user profile
```

**Rate Limiting:**

- 5 attempts / 15 minutes per IP
- Account lock after 5 failed attempts (15 minutes)

**Response:** Same as Registration

### 4.3 OAuth Login (Google, Facebook, Apple)

**HTTP:**

- `POST /api/v1/auth/oauth/google` - Generate OAuth URL
- `POST /api/v1/auth/oauth/google/callback` - Handle callback

**gRPC:**

- `GenerateOAuthURL(provider, redirect_uri, state)`
- `ExchangeOAuthCode(provider, code, redirect_uri)`

**Flow (Google OAuth):**

```
1. Frontend → Backend: Request OAuth URL
2. Backend → Auth Service: GenerateOAuthURL(google)
3. Auth Service → Backend: Returns Google OAuth URL with state
4. Backend → Frontend: Return OAuth URL
5. Frontend: Redirect user to Google
6. User: Authenticate with Google
7. Google → Frontend: Redirect with code
8. Frontend → Backend: Send code
9. Backend → Google: Exchange code for access token
10. Backend → Google: Fetch user info (email, name, picture, google_id)
11. Backend → Auth Service: Find or create user
    - If NOT exists: Create (provider: google, EmailVerified: true)
    - If exists: Update GoogleID if missing
12. Auth Service: Generate JWT tokens
13. Auth Service → Backend: Return tokens + user
14. Backend → Frontend: Return tokens (httpOnly cookies)
```

**User Creation (First OAuth Login):**

- Email from OAuth → Check if user exists
- If NOT exists:
  - Create new User (provider: google, EmailVerified: true)
  - Assign default role "user"
- If exists:
  - Update GoogleID if NULL
  - Update PictureURL, Name if empty
- Generate JWT tokens
- Return tokens + user profile

### 4.4 Token Refresh

**HTTP:** `POST /api/v1/auth/refresh` (Auth Service)
**gRPC:** `RefreshToken(RefreshTokenRequest)`

**Request:**

```go
type RefreshTokenRequest struct {
    RefreshToken string `json:"refresh_token" validate:"required"`
}
```

**Flow:**

```
1. Parse refresh token JWT
2. Extract JTI, UserID from claims
3. Find RefreshToken in DB by JTI
4. Check if IsRevoked or IsExpired
5. Check if already used (Reuse Detection)
   - If reused (IsRevoked=true but used) → Revoke entire family
   - Return 401 Token Reuse Detected
6. Create new RefreshToken (Token Rotation)
   - Same FamilyID as parent
   - ParentTokenID = old token ID
   - New JTI, new expiration (+30 days)
7. Revoke old RefreshToken (IsRevoked=true, reason="token_rotation")
8. Generate new JWT access token
9. Return new tokens
```

**Token Rotation:** Каждый refresh создаёт новую пару токенов, старый refresh отзывается.

**Reuse Detection:** Попытка использовать отозванный токен → отзыв всей семьи (security breach).

**Response:**

```json
{
  "access_token": "new_token...",
  "refresh_token": "new_refresh...",
  "token_type": "Bearer",
  "expires_in": 176400,
  "user": {
    "id": 123,
    "email": "user@example.com"
  }
}
```

### 4.5 Logout

**HTTP:** `POST /api/v1/auth/logout` (Auth Service)
**gRPC:** `Logout(LogoutRequest)`

**Flow:**

```
1. Extract access token from Authorization header
2. Parse access token, get JTI
3. Find related RefreshToken by JTI or UserID
4. Revoke RefreshToken (IsRevoked=true, reason="user_logout")
5. Add access token JTI to revoked_access_tokens (TTL до expiration)
6. Return success message
```

**Logout All Devices:**

**HTTP:** `POST /api/v1/auth/logout-all`
**gRPC:** `LogoutAll(LogoutAllRequest)`

**Flow:**

```
1. Find all active RefreshTokens for UserID
2. Revoke all tokens (IsRevoked=true, reason="logout_all")
3. Return count of revoked tokens
```

### 4.6 Token Validation

**HTTP:** `GET /api/v1/auth/validate` (Auth Service)
**gRPC:** `ValidateToken(ValidateTokenRequest)`

**Flow:**

```
1. Parse JWT token
2. Verify signature (RSA public key)
3. Check expiration (exp claim)
4. Check if token JTI in revoked_access_tokens (Redis cache preferred)
5. Extract claims (user_id, email, roles)
6. Return validation result + claims
```

**Response:**

```json
{
  "valid": true,
  "user_id": 123,
  "email": "user@example.com",
  "roles": ["user", "vendor"],
  "expires_at": 1735123456,
  "claims": {
    "name": "John Doe",
    "provider": "google",
    "email_verified": "true"
  }
}
```

### 4.7 Worker Authentication (WMS/Delivery)

**gRPC:** `CreateWorkerToken(CreateWorkerTokenRequest)`

**Request:**

```protobuf
message CreateWorkerTokenRequest {
  int64 worker_id = 1;        // Worker ID from WMS
  string worker_code = 2;     // Worker code (e.g., "WRK-001")
  string worker_name = 3;     // Worker display name
  int64 warehouse_id = 4;     // Assigned warehouse ID
  string worker_role = 5;     // Role: picker, packer, supervisor, manager
  optional string device_id = 6;
  optional string device_name = 7;
}
```

**Flow:**

```
1. Warehouse/Delivery service calls Auth Service
2. Auth Service creates special JWT token with worker claims:
   - worker_id, worker_code, worker_name
   - warehouse_id, worker_role (picker, packer, supervisor)
3. Token has shorter TTL (8 hours)
4. Return access + refresh tokens
```

**Worker JWT Claims:**

```json
{
  "worker_id": 456,
  "worker_code": "WRK-001",
  "worker_name": "Ivan Petrović",
  "warehouse_id": 10,
  "worker_role": "picker",
  "exp": 1735123456
}
```

**Использование:** WMS mobile app для warehouse workers (picking, packing).

### 4.8 Service-to-Service Authentication

**HTTP:** `POST /api/v1/service/exchange-api-key` (Auth Service)
**gRPC:** `ExchangeAPIKey(ExchangeAPIKeyRequest)`

**Request:**

```go
type ExchangeAPIKeyRequest struct {
    APIKey string `json:"api_key" validate:"required"`
}
```

**Flow:**

```
1. Microservice sends API key
2. Auth Service finds ServiceAccount by hashed API key
3. Verify API key (bcrypt.CompareHashAndPassword)
4. Check if IsActive and NOT expired
5. Update LastUsedAt
6. Generate JWT token with service_name claim (TTL: 1 hour)
7. Return access token
```

**Service JWT Claims:**

```json
{
  "service_name": "warehouse",
  "iss": "https://auth.vondi.rs",
  "exp": 1735123456
}
```

**Автоматическое управление:** `pkg/service.ServiceTokenManager` автоматически обновляет токены за 5 минут до истечения.

---

## 5. API Endpoints

### 5.1 HTTP REST API (Auth Service)

**Base URL:** `http://localhost:28086/api/v1`

#### Health & Info

```
GET  /health               - Health check
GET  /api/v1/version       - Service version
GET  /api/v1/public-key    - RSA public key (for JWT validation)
```

#### Authentication (Public)

```
POST /api/v1/auth/register           - Register new user
POST /api/v1/auth/login              - Login with email/password
POST /api/v1/auth/refresh            - Refresh access token
POST /api/v1/auth/logout             - Logout current device
POST /api/v1/auth/logout-all         - Logout all devices
GET  /api/v1/auth/validate           - Validate JWT token
GET  /api/v1/auth/me                 - Get current user profile
```

#### OAuth (Public)

```
GET  /api/v1/auth/oauth/google           - Generate Google OAuth URL
GET  /api/v1/auth/oauth/google/callback  - Handle Google callback
```

#### User Management (Authenticated, Admin)

```
GET    /api/v1/users                  - List all users
GET    /api/v1/users/:id              - Get user by ID
GET    /api/v1/users/email/:email     - Get user by email
PUT    /api/v1/users/:id              - Update user profile
PATCH  /api/v1/users/:id/status       - Update user status
DELETE /api/v1/users/:id              - Delete user (soft)
GET    /api/v1/users/:id/is-admin     - Check if user is admin
```

#### Role Management (Authenticated, Admin)

```
GET    /api/v1/users/:id/roles        - Get user roles
POST   /api/v1/users/:id/roles        - Add role to user
DELETE /api/v1/users/:id/roles/:role  - Remove role from user
GET    /api/v1/roles                  - List all roles
GET    /api/v1/roles/:role/users      - Get users with role
```

#### Service Accounts (Internal)

```
POST /api/v1/service/exchange-api-key  - Exchange API key for JWT
```

### 5.2 gRPC API (Auth Service)

**Package:** `vondi.auth.v1`
**Proto:** `/p/github.com/vondi-global/auth/api/proto/auth/v1/auth.proto`

**25 методов:**

#### Authentication (6)

- `Register(RegisterRequest) → RegisterResponse`
- `Login(LoginRequest) → LoginResponse`
- `RefreshToken(RefreshTokenRequest) → RefreshTokenResponse`
- `ValidateToken(ValidateTokenRequest) → ValidateTokenResponse`
- `Logout(LogoutRequest) → LogoutResponse`
- `LogoutAll(LogoutAllRequest) → LogoutAllResponse`

#### OAuth (2)

- `GenerateOAuthURL(GenerateOAuthURLRequest) → GenerateOAuthURLResponse`
- `ExchangeOAuthCode(ExchangeOAuthCodeRequest) → ExchangeOAuthCodeResponse`

#### User Management (7)

- `GetAllUsers(GetAllUsersRequest) → GetAllUsersResponse`
- `GetUser(GetUserRequest) → GetUserResponse`
- `GetUserByEmail(GetUserByEmailRequest) → GetUserByEmailResponse`
- `UpdateUserProfile(UpdateUserProfileRequest) → UpdateUserProfileResponse`
- `UpdateUserStatus(UpdateUserStatusRequest) → UpdateUserStatusResponse`
- `DeleteUser(DeleteUserRequest) → DeleteUserResponse`
- `IsUserAdmin(IsUserAdminRequest) → IsUserAdminResponse`

#### Role Management (5)

- `GetUserRoles(GetUserRolesRequest) → GetUserRolesResponse`
- `AddUserRole(AddUserRoleRequest) → AddUserRoleResponse`
- `RemoveUserRole(RemoveUserRoleRequest) → RemoveUserRoleResponse`
- `GetAllRoles(GetAllRolesRequest) → GetAllRolesResponse`
- `GetUsersByRole(GetUsersByRoleRequest) → GetUsersByRoleResponse`

#### Service/Worker Auth (3)

- `ExchangeAPIKey(ExchangeAPIKeyRequest) → ExchangeAPIKeyResponse`
- `CreateWorkerToken(CreateWorkerTokenRequest) → CreateWorkerTokenResponse`
- `GetPublicKey(GetPublicKeyRequest) → GetPublicKeyResponse`

---

## 6. Sequence Diagrams

### 6.1 User Registration Flow

```
┌─────────┐       ┌─────────┐       ┌──────────────┐       ┌──────────┐
│ Browser │       │ Backend │       │ Auth Service │       │ Database │
└────┬────┘       └────┬────┘       └──────┬───────┘       └────┬─────┘
     │                 │                    │                    │
     │ POST /register  │                    │                    │
     ├────────────────>│                    │                    │
     │                 │ Register(email,    │                    │
     │                 │  password, name)   │                    │
     │                 ├───────────────────>│                    │
     │                 │                    │ Validate email,    │
     │                 │                    │ password strength  │
     │                 │                    │                    │
     │                 │                    │ Hash password      │
     │                 │                    │ (bcrypt cost 12)   │
     │                 │                    │                    │
     │                 │                    │ INSERT users       │
     │                 │                    ├───────────────────>│
     │                 │                    │ User ID            │
     │                 │                    │<───────────────────┤
     │                 │                    │                    │
     │                 │                    │ Assign role "user" │
     │                 │                    │                    │
     │                 │                    │ Generate JWT       │
     │                 │                    │ (RS256, 49h TTL)   │
     │                 │                    │                    │
     │                 │                    │ INSERT refresh_    │
     │                 │                    │ tokens             │
     │                 │                    ├───────────────────>│
     │                 │                    │                    │
     │                 │ access_token,      │                    │
     │                 │ refresh_token,     │                    │
     │                 │ user_profile       │                    │
     │                 │<───────────────────┤                    │
     │                 │                    │                    │
     │ Set httpOnly    │                    │                    │
     │ cookies, return │                    │                    │
     │ user profile    │                    │                    │
     │<────────────────┤                    │                    │
```

### 6.2 Login Flow with Account Lock

```
┌─────────┐       ┌─────────┐       ┌──────────────┐       ┌──────────┐
│ Browser │       │ Backend │       │ Auth Service │       │ Database │
└────┬────┘       └────┬────┘       └──────┬───────┘       └────┬─────┘
     │                 │                    │                    │
     │ POST /login     │                    │                    │
     │ (email, pass)   │                    │                    │
     ├────────────────>│                    │                    │
     │                 │ Login(email, pass, │                    │
     │                 │  ip, userAgent)    │                    │
     │                 ├───────────────────>│                    │
     │                 │                    │                    │
     │                 │                    │ SELECT users       │
     │                 │                    │ WHERE email=...    │
     │                 │                    ├───────────────────>│
     │                 │                    │ User found         │
     │                 │                    │<───────────────────┤
     │                 │                    │                    │
     │                 │                    │ Check IsLocked()   │
     │                 │                    │ IF locked:         │
     │                 │                    │   Return 423       │
     │                 │                    │                    │
     │                 │                    │ bcrypt.Compare     │
     │                 │                    │ (password, hash)   │
     │                 │                    │                    │
     │                 │                    │ IF wrong password: │
     │                 │                    │   INCREMENT        │
     │                 │                    │   failed_attempts  │
     │                 │                    │                    │
     │                 │                    │ UPDATE users SET   │
     │                 │                    │ failed_login_      │
     │                 │                    │ attempts = +1      │
     │                 │                    ├───────────────────>│
     │                 │                    │                    │
     │                 │                    │ IF attempts >= 5:  │
     │                 │                    │   LockAccount(15m) │
     │                 │                    │                    │
     │                 │                    │ UPDATE users SET   │
     │                 │                    │ locked_until =     │
     │                 │                    │ NOW() + 15min      │
     │                 │                    ├───────────────────>│
     │                 │                    │                    │
     │                 │ Error 401          │                    │
     │                 │<───────────────────┤                    │
     │                 │                    │                    │
     │ Error: Invalid  │                    │                    │
     │ credentials     │                    │                    │
     │<────────────────┤                    │                    │
     │                 │                    │                    │
     │                 │                    │ IF password OK:    │
     │                 │                    │   RecordLogin()    │
     │                 │                    │                    │
     │                 │                    │ UPDATE users SET   │
     │                 │                    │  last_login_at,    │
     │                 │                    │  last_login_ip,    │
     │                 │                    │  failed_attempts=0,│
     │                 │                    │  locked_until=NULL │
     │                 │                    ├───────────────────>│
     │                 │                    │                    │
     │                 │                    │ Generate JWT       │
     │                 │                    │ INSERT refresh_    │
     │                 │                    │ tokens             │
     │                 │                    ├───────────────────>│
     │                 │                    │                    │
     │                 │ Tokens + user      │                    │
     │                 │<───────────────────┤                    │
     │ Success         │                    │                    │
     │<────────────────┤                    │                    │
```

### 6.3 Token Refresh with Rotation

```
┌─────────┐       ┌─────────┐       ┌──────────────┐       ┌──────────┐
│ Browser │       │ Backend │       │ Auth Service │       │ Database │
└────┬────┘       └────┬────┘       └──────┬───────┘       └────┬─────┘
     │                 │                    │                    │
     │ POST /refresh   │                    │                    │
     │ (refresh_token) │                    │                    │
     ├────────────────>│                    │                    │
     │                 │ RefreshToken(token)│                    │
     │                 ├───────────────────>│                    │
     │                 │                    │                    │
     │                 │                    │ Parse JWT,         │
     │                 │                    │ extract JTI        │
     │                 │                    │                    │
     │                 │                    │ SELECT refresh_    │
     │                 │                    │ tokens WHERE       │
     │                 │                    │ jti = ...          │
     │                 │                    ├───────────────────>│
     │                 │                    │ RefreshToken row   │
     │                 │                    │<───────────────────┤
     │                 │                    │                    │
     │                 │                    │ Check IsRevoked    │
     │                 │                    │                    │
     │                 │                    │ IF revoked:        │
     │                 │                    │   SECURITY BREACH! │
     │                 │                    │   Revoke entire    │
     │                 │                    │   family           │
     │                 │                    │ UPDATE refresh_    │
     │                 │                    │ tokens SET         │
     │                 │                    │ is_revoked=true    │
     │                 │                    │ WHERE family_id=...│
     │                 │                    ├───────────────────>│
     │                 │                    │ Error 401          │
     │                 │                    │ (Token reuse)      │
     │                 │                    │                    │
     │                 │                    │ IF valid:          │
     │                 │                    │ INSERT new         │
     │                 │                    │ refresh_token      │
     │                 │                    │ (same family_id)   │
     │                 │                    ├───────────────────>│
     │                 │                    │ New token ID       │
     │                 │                    │<───────────────────┤
     │                 │                    │                    │
     │                 │                    │ UPDATE old token   │
     │                 │                    │ SET is_revoked=true│
     │                 │                    ├───────────────────>│
     │                 │                    │                    │
     │                 │                    │ Generate new JWT   │
     │                 │                    │ access_token       │
     │                 │                    │                    │
     │                 │ New tokens         │                    │
     │                 │<───────────────────┤                    │
     │ New tokens      │                    │                    │
     │<────────────────┤                    │                    │
```

---

## 7. Security

### 7.1 Password Security

**Hashing:** bcrypt with cost factor 12 (~250ms per hash)
**Minimum Strength:** 8 characters
**Salt:** Automatic per bcrypt (unique per password)
**Storage:** `users.password_hash` (NULL for OAuth users)

### 7.2 JWT Security

**Algorithm:** RS256 (RSA-SHA256)
**Key Size:** 2048-bit RSA keypair
**Location:** `keys/private.pem`, `keys/public.pem`

**Access Token:**

- TTL: 49 hours (configurable)
- Storage: Browser memory (NOT localStorage)
- Transmission: Authorization: Bearer header

**Refresh Token:**

- TTL: 30 days
- Storage: HttpOnly cookie (recommended)
- One-time use (Token Rotation)

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
  "provider": "google",
  "email_verified": true,
  "term_accepted": true,
  "exp": 1735123456,
  "iat": 1735000000,
  "jti": "unique-id"
}
```

### 7.3 Account Security

**Account Locking:**

- 5 failed login attempts → 15 minutes lock
- Stored in `users.locked_until`
- Auto-unlock after duration or successful login

**Session Management:**

- RefreshToken tracking (device, IP, user-agent)
- Token rotation on each refresh
- Reuse detection → revoke family
- Logout all devices support

**Rate Limiting:**

- Login: 5 attempts / 15 minutes per IP
- Register: 3 registrations / 1 hour per IP
- Token Refresh: 30 refreshes / 1 minute per user
- Implementation: Redis with sliding window

### 7.4 RBAC (Role-Based Access Control)

**Authorization Flow:**

```
1. JWT token contains roles array
2. Middleware extracts roles from token
3. Check if user has required role/permission
4. Allow or deny access
```

**Example:**

```go
// Require admin role
app.Use(authmiddleware.RequireAuth(entity.RoleAdmin))

// Check permission in handler
if !user.HasPermission(entity.PermListingsEditAny) {
    return fiber.ErrForbidden
}
```

---

## 8. Integration & Public Library

### 8.1 Публичная библиотека (pkg)

Auth Service предоставляет **публичную библиотеку `pkg`** для упрощения интеграции.

**Структура:**

```
pkg/
├── service/              # Protocol-agnostic (главная точка входа)
│   ├── auth.go          - AuthService (HTTP + gRPC auto-select)
│   ├── user.go          - UserService
│   ├── oauth.go         - OAuthService
│   └── service_token_manager.go - Auto S2S token refresh
│
├── entity/              - Общие модели (protocol-agnostic)
│   ├── auth.go
│   ├── user.go
│   └── roles.go         - Роли и разрешения
│
├── http/                - HTTP-specific
│   ├── client/          - OpenAPI client
│   └── fiber/           - Fiber middleware
│       └── middleware/
│           └── auth.go  - RequireAuth, GetUserID, GetEmail
│
├── grpc/                - gRPC-specific
│   ├── client/
│   ├── interceptor/
│   └── proto/
│
└── proto/               - Protobuf sources
```

### 8.2 Использование библиотеки

```go
import (
    "github.com/vondi-global/auth/pkg/service"
    "github.com/vondi-global/auth/pkg/entity"
    authmiddleware "github.com/vondi-global/auth/pkg/http/fiber/middleware"
)

// Инициализация
authSvc, _ := service.NewAuthService(&service.Config{
    HTTPURL: "http://auth-service:28086",  // Optional
    GRPCURL: "auth-service:20053",         // Optional (priority)
    Timeout: 30 * time.Second,
})

// Login
resp, err := authSvc.Login(ctx, entity.UserLoginRequest{
    Email: "user@example.com",
    Password: "password",
})

// Middleware
app.Use(authmiddleware.RequireAuth())        // Any authenticated
app.Use(authmiddleware.RequireAuth("admin")) // Admin only

// In handler
userID, _ := authmiddleware.GetUserID(c)
email, _ := authmiddleware.GetEmail(c)
roles, _ := authmiddleware.GetRoles(c)
```

### 8.3 ServiceTokenManager (S2S автоматическое обновление)

```go
import "github.com/vondi-global/auth/pkg/service"

// Auto-refresh service tokens
tokenManager := service.NewServiceTokenManager(
    "warehouse",                      // Service name
    "api-key-from-env",               // API key
    "http://auth-service:28086",      // Auth service URL
)

// Get token (auto-refreshed 5 min before expiry)
token, _ := tokenManager.GetToken(ctx)

// Use token for S2S calls
md := metadata.Pairs("authorization", "Bearer "+token)
ctx := metadata.NewOutgoingContext(context.Background(), md)
```

---

## 9. Database Schema

### 9.1 Auth Service Tables

**auth.users** (90 rows, 416 KB)

- Primary table for user accounts
- 33 columns (ID, UUID, Email, PasswordHash, Profile, Security)
- Indexes: email_normalized (unique), uuid (unique), oauth IDs

**auth.refresh_tokens** (193 rows, 360 KB)

- Refresh token storage with rotation
- Columns: JTI, UserID, TokenHash, FamilyID, DeviceTracking
- Indexes: JTI (unique), UserID, FamilyID

**auth.user_sessions** (0 rows, 64 KB)

- Session tracking (NOT used, via RefreshToken instead)

**auth.service_accounts** (4 rows, 112 KB)

- Service-to-service authentication
- Columns: ServiceName, APIKey, Status, Expiration

**auth.user_addresses** (3 rows, 152 KB)

- User delivery addresses (multilingual, Post Express ready)

**auth.revoked_access_tokens** (2 rows, 64 KB)

- Revoked JWT access token JTIs
- TTL-based cleanup (Redis preferred for production)

**auth.user_checkout_profiles** (0 rows, 56 KB)

- Checkout profiles (NOT used)

### 9.2 Migrations

**Location:** `/p/github.com/vondi-global/auth/migrations/`
**Total:** 9 migrations

```
0001_functions.sql          - SQL helper functions
0002_users.sql              - Users table (33 columns)
0003_refresh_tokens.sql     - Refresh tokens table
0004_user_sessions.sql      - User sessions table
0005_revoked_access_tokens.sql - Revoked tokens
0006_user_profile_extended.sql - Extended profile fields
0007_create_service_accounts.sql - Service accounts
0008_user_addresses.sql     - User addresses
0009_user_checkout_profiles.sql - Checkout profiles
```

**Apply:**

```bash
cd /p/github.com/vondi-global/auth
make migrate-up
```

---

## 10. Known Issues & Recommendations

### Issues

1. **Пустые таблицы:** user_sessions, user_checkout_profiles
2. **Дублирующиеся индексы:** 17 штук (~240 KB overhead)
3. **NULL колонки:** 26 колонок всегда NULL (никогда не заполнялись)

### Рекомендации

**Краткосрочные:**

- Удалить дублирующиеся индексы
- Добавить индексы на user_addresses (user_id, is_default)
- Включить Redis для revoked_access_tokens (вместо PostgreSQL)

**Среднесрочные:**

- Реализовать user_sessions или удалить таблицу
- Реализовать user_checkout_profiles или удалить
- Создать audit_log для compliance

**Долгосрочные:**

- Добавить 2FA (TOTP, SMS)
- Реализовать passwordless auth (magic links)
- Добавить biometric authentication support

---

## 11. Testing

### 11.1 JWT для тестов

**Локально:**

```bash
cd /home/dim/jwtgen && ./jwtgen > /tmp/token
TOKEN=$(cat /tmp/token)
curl -H "Authorization: Bearer $TOKEN" http://localhost:3000/api/v1/auth/me | jq
```

**Для vondi.rs:**

```bash
ssh vondi "cat /opt/vondi-dev/keys/private.pem" > /tmp/vondi_private.pem
KEY_PATH=/tmp/vondi_private.pem ./jwtgen > /tmp/token_vondi
```

### 11.2 Integration Tests

**Auth Service:**

```bash
cd /p/github.com/vondi-global/auth
make test-integration
```

### 11.3 Example Application

**TaskTracker:** `/p/github.com/vondi-global/auth/examples/tasktracker/`

- Demonstrates full OAuth flow
- User authentication
- Service-to-service auth (TaskTracker → Payment)

**Run:**

```bash
cd /p/github.com/vondi-global/auth
make example-app-docker-up-build-all
# Visit: http://localhost:20033
```

---

## 12. Deployment

### 12.1 Development (Local)

```bash
cd /p/github.com/vondi-global/auth

# Start dependencies
docker compose up -d postgres redis

# Apply migrations
make migrate-up

# Run service
screen -dmS auth-service bash -c './bin/auth-service'

# Check health
curl http://localhost:28086/health
```

### 12.2 Production (dev.vondi.rs)

**Ports:**

- HTTP: 28086
- gRPC: 20053
- Metrics: 29090

**Environment:**

- `VONDIAUTH_SERVICE_ENV=production`
- `VONDIAUTH_JWT_ISSUER=https://auth.vondi.rs`
- JWT keys mounted from `/opt/vondi-dev/keys/`

---

## 13. Monitoring

### 13.1 Metrics (Prometheus)

**Endpoint:** `http://localhost:29090/metrics`

**Custom Metrics:**

- `auth_login_total` (labels: status, provider)
- `auth_registration_total` (labels: provider)
- `auth_token_refresh_total`
- `auth_token_validation_total`
- `auth_account_lock_total`

### 13.2 Logging

```bash
# Auth Service (local)
tail -f /p/github.com/vondi-global/auth/logs/auth.log

# Auth Service (server)
ssh vondi "docker logs -f vondi-dev-auth"
```

---

## 14. References

**Документация:**

- `/p/github.com/vondi-global/auth/README.md` - Quick start
- `/p/github.com/vondi-global/auth/CLAUDE.md` - Development guide
- `/p/github.com/vondi-global/auth/pkg/README.md` - Library guide
- `/p/github.com/vondi-global/.passport/services/auth.md` - Service passport

**API Specifications:**

- `/p/github.com/vondi-global/auth/api/proto/auth/v1/auth.proto` - gRPC API

**Repository:**

- **GitHub:** `github.com/vondi-global/auth`
- **Local:** `/p/github.com/vondi-global/auth`
- **Workflow:** Only via Pull Requests (NO direct push to main)

---

**Владелец:** VONDI GLOBAL D.O.O., Beograd, Srbija
**Генеральный директор:** Malden Simić
**Версия документа:** 1.1.0
**Дата создания:** 2025-12-21
