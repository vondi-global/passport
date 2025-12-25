# Security: Authentication & Authorization - Vondi Platform

**Версия:** 1.0.0
**Обновлено:** 2025-12-21
**Статус:** Production Ready ✅
**Владелец:** Auth Service Team

---

## Содержание

1. [Обзор архитектуры](#1-обзор-архитектуры)
2. [JWT RS256 Token System](#2-jwt-rs256-token-system)
3. [Token Rotation Strategy](#3-token-rotation-strategy)
4. [Google OAuth 2.0](#4-google-oauth-20)
5. [Firebase Phone Authentication](#5-firebase-phone-authentication)
6. [Session Management](#6-session-management)
7. [Account Lockout & Rate Limiting](#7-account-lockout--rate-limiting)
8. [Service-to-Service Authentication](#8-service-to-service-authentication)
9. [Security Headers & CSRF Protection](#9-security-headers--csrf-protection)
10. [Password Security](#10-password-security)
11. [Role-Based Access Control (RBAC)](#11-role-based-access-control-rbac)
12. [Security Monitoring](#12-security-monitoring)

---

## 1. Обзор архитектуры

### 1.1 Компоненты безопасности

```
┌─────────────────────────────────────────────────────────────────┐
│                         BROWSER                                  │
│  - httpOnly cookies (access_token, refresh_token)               │
│  - No localStorage/sessionStorage tokens                        │
│  - XSS protection via httpOnly                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ HTTPS only (TLS 1.3)
                         │ Security Headers (HSTS, CSP, etc.)
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   BFF PROXY (Next.js API Routes)                 │
│  - Cookie management (set httpOnly cookies)                      │
│  - CSRF token validation (state parameter)                       │
│  - Token extraction and forwarding                               │
│  - Rate limiting per IP                                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ Authorization: Bearer <JWT>
                         │ Internal network only
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                 BACKEND MONOLITH (Fiber/Go)                      │
│  - JWT validation with public key (RS256)                        │
│  - Role-based access control middleware                          │
│  - Token blacklist check (Redis)                                 │
│  - User context extraction                                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ gRPC/HTTP (internal)
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   AUTH SERVICE (Internal API)                    │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ Security Features:                                      │    │
│  │ - JWT RS256 generation & validation                    │    │
│  │ - Bcrypt password hashing (cost 12)                    │    │
│  │ - Refresh token rotation & family tracking             │    │
│  │ - Account lockout (5 failed attempts → 15min)          │    │
│  │ - Rate limiting (Redis)                                │    │
│  │ - OAuth 2.0 integration (Google)                       │    │
│  │ - Firebase Phone Auth                                  │    │
│  │ - Service-to-service API keys                          │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │
                         ▼
┌──────────────────┐   ┌──────────────────┐   ┌────────────────┐
│   PostgreSQL     │   │      Redis       │   │  Google OAuth  │
│   - Users        │   │  - Tokens        │   │  - User info   │
│   - Sessions     │   │  - Blacklist     │   │  - Verification│
│   - Roles        │   │  - Rate limits   │   └────────────────┘
└──────────────────┘   └──────────────────┘
```

### 1.2 Принципы безопасности

**Многоуровневая защита (Defense in Depth):**
1. **Transport Layer:** HTTPS/TLS 1.3 only
2. **Application Layer:** JWT validation, RBAC, CSRF tokens
3. **Session Layer:** httpOnly cookies, token rotation
4. **Storage Layer:** Encrypted passwords (bcrypt), encrypted tokens in DB
5. **Monitoring Layer:** Audit logs, rate limiting, anomaly detection

**Zero Trust Architecture:**
- ❌ НЕ доверяем токенам из клиента (всегда валидируем)
- ❌ НЕ доверяем IP адресам (могут быть подменены)
- ✅ Валидируем каждый запрос независимо
- ✅ Используем короткие TTL для access токенов (15m)
- ✅ Ротация refresh токенов при каждом обновлении

---

## 2. JWT RS256 Token System

### 2.1 Почему RS256 (не HS256)

**RS256 (RSA-SHA256) - асимметричное шифрование:**
- ✅ Приватный ключ ТОЛЬКО на Auth Service
- ✅ Публичный ключ доступен всем микросервисам
- ✅ Микросервисы могут валидировать токены БЕЗ доступа к Auth Service
- ✅ Невозможно подделать токен без приватного ключа
- ✅ Легко ротировать ключи без изменения кода микросервисов

**HS256 (HMAC-SHA256) - симметричное шифрование:**
- ❌ Один секретный ключ для подписи И валидации
- ❌ Все микросервисы должны знать секрет → риск утечки
- ❌ Любой микросервис может подделать токены
- ❌ Сложно ротировать ключ (нужно обновить везде одновременно)

### 2.2 Генерация ключей

**Создание RSA keypair (2048-bit):**
```bash
cd /p/github.com/vondi-global/auth
make keys
```

**Команды:**
```bash
# Приватный ключ (Auth Service ONLY)
openssl genrsa -out keys/private.pem 2048

# Публичный ключ (доступен всем)
openssl rsa -in keys/private.pem -pubout -out keys/public.pem

# Проверка
openssl rsa -in keys/private.pem -check
```

**Безопасность ключей:**
- ✅ `keys/private.pem` - chmod 600, НИКОГДА не коммитить в Git
- ✅ `keys/public.pem` - можно коммитить, доступен по HTTP `/api/v1/public-key`
- ✅ Автоматическая ротация ключей раз в 90 дней (TODO)
- ✅ Поддержка нескольких публичных ключей одновременно (graceful rotation)

### 2.3 Access Token

**Параметры:**
- **TTL:** 49 часов (2 дня) - настраивается через `VONDIAUTH_JWT_ACCESS_TOKEN_DURATION`
- **Алгоритм:** RS256
- **Размер:** ~800-1200 bytes (зависит от ролей/разрешений)
- **Формат:** `Bearer <token>`

**Структура (Claims):**
```json
{
  "iss": "https://auth.vondi.rs",
  "aud": "https://vondi.rs",
  "sub": "user@example.com",
  "exp": 1735123456,
  "iat": 1735000000,
  "jti": "550e8400-e29b-41d4-a716",
  "user_id": 123,
  "user_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "name": "John Doe",
  "roles": ["user", "vendor", "warehouse_worker"],
  "provider": "google",
  "term_accepted": true,
  "email_verified": true,
  "two_factor_enabled": false,
  "service_name": "tasktracker",
  "worker_id": 456,
  "worker_code": "W12345",
  "warehouse_id": 789,
  "worker_role": "picker"
}
```

### 2.4 Refresh Token

**Параметры:**
- **TTL:** 720 часов (30 дней)
- **Алгоритм:** RS256 (JWT) + SHA256 hash в БД
- **Rotation:** Каждый refresh создаёт НОВЫЙ токен
- **Family Tracking:** Все токены в цепочке имеют один `family_id`

**Структура (Claims):**
```json
{
  "iss": "https://auth.vondi.rs",
  "exp": 1737378600,
  "iat": 1734786600,
  "jti": "rt_abc123xyz",
  "user_id": 123,
  "family_id": "550e8400-e29b-41d4-a716",
  "device_id": "device_12345"
}
```

---

## 3. Token Rotation Strategy

### 3.1 Refresh Token Rotation

**Почему ротация необходима:**
- ✅ Защита от кражи токенов (replay attack)
- ✅ Ограничение времени жизни скомпрометированного токена
- ✅ Обнаружение повторного использования старого токена
- ✅ Автоматическая блокировка всей цепочки токенов при атаке

**Процесс ротации:**
1. Клиент отправляет старый refresh token
2. Auth Service валидирует токен
3. Генерируется НОВЫЙ access token
4. Генерируется НОВЫЙ refresh token (с тем же family_id)
5. Старый refresh token отзывается (is_revoked = TRUE)
6. Если старый токен уже отозван → **SECURITY BREACH** → отзыв ВСЕЙ семьи

### 3.2 Token Family Tracking

**Family ID - UUID для отслеживания всех токенов одного логина:**

```
Login (2025-12-01 10:00)
└─ Token 1 (family_id: abc-123)
   └─ Refresh (2025-12-01 10:15)
      └─ Token 2 (family_id: abc-123, parent: Token 1)
         └─ Refresh (2025-12-01 10:30)
            └─ Token 3 (family_id: abc-123, parent: Token 2)

❌ Reuse Token 1 → Revoke ALL (Token 1, 2, 3)
```

**Преимущества:**
- Мгновенная блокировка всех токенов при компрометации
- История ротации для аудита
- Можно отследить откуда произошла утечка

---

## 4. Google OAuth 2.0

### 4.1 Конфигурация

**OAuth 2.0 Client (Google Cloud Console):**
- **Client ID:** `<your-google-client-id>.apps.googleusercontent.com`
- **Client Secret:** `<your-google-client-secret>`
- **Authorized Redirect URIs:**
  - `https://dev.vondi.rs/api/auth/google/callback`
  - `https://vondi.rs/api/auth/google/callback`

**OAuth Scopes:**
```
openid                  // Required for OIDC
email                   // User email address
profile                 // User name and picture
```

### 4.2 OAuth Flow (Authorization Code)

**Последовательность:**
1. User clicks "Sign in with Google"
2. BFF generates random state (CSRF token) and saves in cookie
3. Redirect to Google authorization URL
4. User approves permissions
5. Google redirects to callback with code + state
6. BFF validates state (CSRF protection)
7. BFF exchanges code for access_token via Backend → Auth Service
8. Auth Service requests user info from Google
9. Create/update user in database
10. Generate JWT tokens (access + refresh)
11. Set httpOnly cookies and redirect to dashboard

### 4.3 CSRF Protection (State Parameter)

**Генерация state:**
```typescript
const state = crypto.randomUUID();
cookies().set('oauth_state', state, {
  httpOnly: true,
  secure: true,
  sameSite: 'lax',
  maxAge: 600
});
```

**Валидация state:**
```typescript
const receivedState = searchParams.get('state');
const savedState = cookies().get('oauth_state')?.value;

if (receivedState !== savedState) {
  throw new Error('CSRF attack detected');
}
```

---

## 5. Firebase Phone Authentication

### 5.1 Конфигурация

**Firebase Project:**
- **Project ID:** `vondi-global`
- **Service Account:** `keys/firebase-service-account.json`
- **Supported Countries:** Serbia (+381), Russia (+7), все международные

**Environment Variables:**
```bash
VONDIAUTH_FIREBASE_PROJECT_ID=vondi-global
VONDIAUTH_FIREBASE_SERVICE_ACCOUNT_KEY_PATH=keys/firebase-service-account.json
```

### 5.2 Phone Auth Flow

1. User enters phone number (+381641234567)
2. Auth Service validates format (E.164)
3. Check rate limit (max 5 SMS/day per phone)
4. Send verification code via Firebase
5. User enters OTP code
6. Auth Service verifies code with Firebase
7. Create/update user with phone_verified=true
8. Generate JWT tokens
9. Set httpOnly cookies and login

### 5.3 Phone Number Format (E.164)

**Примеры:**
```
✅ +381641234567    (Serbia mobile)
✅ +79991234567     (Russia mobile)
✅ +442071234567    (UK landline)

❌ 381641234567     (missing +)
❌ +381 64 123 4567 (spaces not allowed)
❌ +381-64-123-4567 (dashes not allowed)
```

### 5.4 Rate Limiting для SMS

**Защита от злоупотребления:**
- 5 SMS максимум на номер в день
- Минимум 60 секунд между SMS на один номер
- Redis для отслеживания лимитов

---

## 6. Session Management

### 6.1 Cookie-Based Sessions

**httpOnly Cookies (BFF устанавливает):**
```typescript
// Access token cookie
cookies().set('access_token', accessToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  path: '/',
  maxAge: 900  // 15 минут
});

// Refresh token cookie
cookies().set('refresh_token', refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  path: '/',
  maxAge: 2592000  // 30 дней
});
```

**Почему НЕ localStorage/sessionStorage:**
- ❌ Доступны из JavaScript → уязвимы к XSS атакам
- ❌ Отправляются во ВСЕХ запросах (даже к внешним API)
- ❌ Сложно реализовать автоматический refresh

**Преимущества httpOnly cookies:**
- ✅ Недоступны для JavaScript → защита от XSS
- ✅ Автоматически отправляются браузером
- ✅ SameSite защита от CSRF
- ✅ Автоматическое удаление при истечении

### 6.2 Session Tracking (PostgreSQL)

**Таблица user_sessions:**
- Отслеживание всех активных сессий пользователя
- Device information (ID, name, type)
- IP address и geolocation
- Last activity timestamp
- Возможность отзыва всех сессий (Logout All)

---

## 7. Account Lockout & Rate Limiting

### 7.1 Failed Login Attempts Tracking

**Механизм блокировки:**
- **1-4 попытки:** Warning (осталось N попыток)
- **5 попыток:** Блокировка на 15 минут
- **10 попыток за час:** Блокировка на 1 час
- **20 попыток за день:** Блокировка на 24 часа + алерт безопасности

**Реализация (Redis):**
- Счётчик попыток для email: `login_attempts:email:{email}`
- Счётчик попыток для IP: `login_attempts:ip:{ip}`
- Автоматическое истечение через TTL
- Сброс счётчика при успешном логине

### 7.2 Rate Limiting по Endpoints

| Endpoint | Лимит | Окно | Ключ |
|----------|-------|------|------|
| `/auth/login` | 5 requests | 15 min | IP + Email |
| `/auth/register` | 3 requests | 1 hour | IP |
| `/auth/refresh` | 30 requests | 1 min | User ID |
| `/auth/phone/send-code` | 5 SMS | 24 hours | Phone |
| `/auth/password/reset` | 3 requests | 1 hour | Email |

---

## 8. Service-to-Service Authentication

### 8.1 API Keys для микросервисов

**Таблица service_accounts:**
- Уникальный API key для каждого микросервиса
- SHA256 hash хранится в БД (не plaintext)
- Prefix для идентификации (первые 8 символов)
- Expiration date (опционально)
- Usage tracking (last_used_at, total_requests)

**Формат API Key:**
```
vondi_live_k8s7h3j9g2f4d6a1c5b9e7w2q4t8y1u3
│     │    └────────────────────────────┘
│     │           32-char random
│     └─ Environment (live | test | dev)
└─ Prefix (vondi)
```

### 8.2 Service Token Exchange

**Обмен API Key на JWT:**
1. Микросервис отправляет API key
2. Auth Service валидирует (SHA256 hash)
3. Проверка активности и expiration
4. Генерация JWT с service claims
5. Return JWT (TTL: 1 час)

### 8.3 Dual Authentication (Service + User)

**Сценарий:** TaskTracker вызывает Payment Service от имени пользователя

```
TaskTracker → Payment Service
Headers:
  Authorization: Bearer {service_token}
  x-user-token: {user_token}
```

Payment Service валидирует ОБА токена для полной авторизации.

---

## 9. Security Headers & CSRF Protection

### 9.1 HTTP Security Headers

**Обязательные заголовки:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

### 9.2 CSRF Protection

**State Parameter (OAuth):**
- Генерация случайного UUID
- Хранение в httpOnly cookie
- Валидация при callback
- Автоматическое удаление после использования

**SameSite Cookie Attribute:**
- `strict` - для access/refresh токенов
- `lax` - для OAuth state
- `none` - НИКОГДА для auth токенов

---

## 10. Password Security

### 10.1 Bcrypt Hashing

**Параметры:**
- **Algorithm:** bcrypt
- **Cost Factor:** 12
- **Salt:** Автоматически генерируется (16 байт)

**Формат hash:**
```
$2a$12$KIXxLVQZ9k7X8h3j2L9fU.Z8h3j2L9fU8h3j2L9fU8h3j2L9fU
└┬┘ └┬┘ └──────────┬──────────┘ └────────────┬────────────┘
 │   │            │                           │
 │   │            │                           └─ Hash (31 bytes)
 │   │            └─ Salt (22 chars base64)
 │   └─ Cost factor (12)
 └─ Algorithm (2a = bcrypt)
```

### 10.2 Password Requirements

**Валидация:**
- Минимум 8 символов
- Максимум 128 символов
- Обязательно: uppercase, lowercase, digit
- Опционально: special character
- Проверка на частые пароли (blacklist)

### 10.3 Password Reset Flow

**Безопасность:**
- Reset token - UUID (случайный, не предсказуемый)
- Token TTL - 1 час
- Хранение в Redis (автоматическое истечение)
- Одноразовый токен (удаляется после использования)
- НЕ раскрывает существование email (всегда 200 OK)
- Автоматический logout со всех устройств после сброса

---

## 11. Role-Based Access Control (RBAC)

### 11.1 Роли и разрешения

**28 системных ролей:**

#### Административные (2)
- `super_admin`, `admin`

#### Модерация (5)
- `moderator`, `content_moderator`, `review_moderator`, `chat_moderator`, `dispute_manager`

#### Бизнес (4)
- `vendor_manager`, `category_manager`, `marketing_manager`, `financial_manager`

#### Операции (5)
- `warehouse_manager`, `warehouse_worker`, `pickup_manager`, `pickup_worker`, `courier`

#### Поддержка (3)
- `support_l1`, `support_l2`, `support_l3`

#### Комплаенс (2)
- `legal_advisor`, `compliance_officer`

#### Продавцы (4)
- `professional_vendor`, `vendor`, `individual_seller`, `storefront_staff`

#### Покупатели (3)
- `verified_buyer`, `vip_customer`, `user`

#### Аналитика (2)
- `data_analyst`, `business_analyst`

#### Служебные (1)
- `service`

**67 разрешений** распределены по категориям:
- User Management (8)
- Admin Panel (7)
- Listings (10)
- Orders (7)
- Payments (5)
- Messaging (3)
- Reviews (3)
- Storefronts (5)
- Warehouse (4)
- Pickup (3)
- Delivery (3)
- Categories (4)
- Marketing (4)
- System (6)
- Analytics (3)
- Support (3)
- Financial (4)
- Compliance (3)

### 11.2 Проверка разрешений

**Middleware для ролей:**
```go
import authmiddleware "github.com/vondi-global/auth/pkg/http/fiber"

// Требуется роль admin
app.Use(authmiddleware.RequireRole("admin"))

// Требуется ЛЮБАЯ из ролей
app.Use(authmiddleware.RequireRole("admin", "moderator", "super_admin"))
```

### 11.3 Hierarchical Roles

**Приоритеты:**
- `super_admin` (100) → ВСЕ разрешения
- `admin` (90) → Большинство разрешений
- `moderator` (80)
- `warehouse_manager` (60)
- `vendor` (40)
- `user` (10) → Базовая роль

---

## 12. Security Monitoring

### 12.1 Audit Logging

**События для логирования:**

| Событие | Уровень | Данные |
|---------|---------|--------|
| User Login | INFO | user_id, email, ip, user_agent |
| Failed Login | WARN | email, ip, attempts_count |
| Account Locked | ERROR | user_id, email, ip, reason |
| Token Reuse Detected | CRITICAL | family_id, user_id, ip |
| Password Changed | INFO | user_id, ip |
| Role Added | WARN | user_id, role, admin_id |
| OAuth Login | INFO | user_id, provider, ip |
| Service Auth | INFO | service_name, endpoint |
| Rate Limit Hit | WARN | endpoint, ip, limit |

### 12.2 Metrics (Prometheus)

**Ключевые метрики:**
```prometheus
# Логины
auth_login_total{status="success"}
auth_login_total{status="failed"}
auth_login_total{status="locked"}

# Токены
auth_token_issued_total{type="access"}
auth_token_issued_total{type="refresh"}
auth_token_validated_total{result="valid"}
auth_token_rotated_total

# Security events
auth_account_lockout_total
auth_token_reuse_detected_total
auth_rate_limit_exceeded_total{endpoint}
```

### 12.3 Alerting

**Критичные алерты:**

1. **BruteForceAttack** - высокая частота неудачных логинов
2. **TokenReuseDetected** - обнаружена попытка повторного использования токена
3. **MassAccountLockout** - массовая блокировка аккаунтов
4. **SlowTokenValidation** - медленная валидация токенов
5. **HighInvalidTokenRate** - высокий процент невалидных токенов

### 12.4 Incident Response

**При обнаружении token reuse:**

1. **Автоматические действия:**
   - Отзыв всех токенов в семействе (family_id)
   - Блокировка аккаунта на 1 час
   - Отправка email уведомления пользователю
   - Запись в audit log (CRITICAL level)

2. **Ручные действия (Security Team):**
   - Анализ логов (IP, user_agent, геолокация)
   - Проверка других аккаунтов с того же IP
   - Расследование возможной утечки токена
   - Уведомление пользователя о подозрительной активности

3. **Пользовательские действия:**
   - Смена пароля (обязательно)
   - Проверка активных сессий
   - Включение 2FA (рекомендуется)

---

## Заключение

**Статус безопасности платформы Vondi: ВЫСОКИЙ ✅**

**Реализовано:**
- ✅ JWT RS256 с короткими TTL (49h access, 30d refresh)
- ✅ Refresh token rotation с family tracking
- ✅ Google OAuth 2.0 с CSRF защитой
- ✅ Firebase Phone Authentication
- ✅ httpOnly cookies (защита от XSS)
- ✅ Bcrypt password hashing (cost 12)
- ✅ Account lockout (5 попыток → 15min)
- ✅ Rate limiting на критичных endpoints
- ✅ Service-to-service authentication
- ✅ Security headers (HSTS, CSP, X-Frame-Options)
- ✅ RBAC (28 ролей, 67 разрешений)
- ✅ Audit logging и monitoring

**Рекомендации для улучшения:**
1. Включить 2FA (TOTP) для всех администраторов
2. Автоматическая ротация RSA ключей (каждые 90 дней)
3. Реализовать WebAuthn/FIDO2 для passwordless auth
4. Device fingerprinting для обнаружения аномалий
5. Интеграция с SIEM системой

---

**Документ обновлён:** 2025-12-21
**Версия:** 1.0.0
**Автор:** Security & Auth Team
**Владелец:** VONDI GLOBAL D.O.O., Beograd, Srbija
**Генеральный директор:** Malden Simić
