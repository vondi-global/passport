# Data Protection & Security Measures

> Обновлено: 2025-12-21
> Версия: 1.0
> Статус: Production

## Обзор

Платформа Vondi реализует комплексный подход к защите данных, включающий:

1. **Криптографическая защита** - bcrypt, RSA, TLS 1.3
2. **Защита от атак** - CORS, CSRF, XSS, SQL Injection, Rate Limiting
3. **Приватность данных** - геолокация, GDPR compliance
4. **Аудит и логирование** - отслеживание всех критических операций
5. **Безопасная передача** - HTTPS/TLS, защищенные cookies
6. **Валидация данных** - file uploads, input sanitization

---

## 1. Криптография и хеширование

### 1.1 Password Hashing (Bcrypt)

**Алгоритм:** bcrypt
**Cost Factor:** 12 (2^12 = 4096 итераций)
**Библиотека:** `golang.org/x/crypto/bcrypt`

```go
// Auth Service: internal/service/auth/service.go
const bcryptCost = 12

// Хеширование при регистрации
hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcryptCost)

// Проверка при входе
err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(password))
```

**Безопасность bcrypt:**
- Адаптивная функция (cost растет с развитием железа)
- Встроенная salt генерация (случайная для каждого пароля)
- Защита от rainbow tables и brute force
- Время вычисления ~100-300ms на cost=12 (защита от перебора)

**Конфигурация:**
```bash
# .env Auth Service
VONDIAUTH_SECURITY_BCRYPT_COST=12  # Default: 12, Min: 10, Max: 14
```

**Почему не Argon2:**
- bcrypt - проверенный временем стандарт (1999 год)
- Широкая поддержка в Go ecosystem
- Достаточный уровень безопасности для нашего use case
- Возможность миграции на Argon2id в будущем

---

### 1.2 JWT RS256 (Asymmetric Cryptography)

**Алгоритм:** RS256 (RSA Signature with SHA-256)
**Key Size:** 2048 bit RSA
**Библиотека:** `github.com/golang-jwt/jwt/v5`

```go
// Auth Service: internal/service/token/jwt.go
type JWTService struct {
    privateKey *rsa.PrivateKey  // Для подписи токенов
    publicKey  *rsa.PublicKey   // Для валидации токенов
}

// Генерация ключей (один раз при инициализации)
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

**Структура токена:**
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user_id",
    "email": "user@example.com",
    "roles": ["user", "admin"],
    "iss": "https://auth.vondi.rs",
    "aud": "https://vondi.rs",
    "exp": 1735000000,
    "iat": 1734999100,
    "jti": "unique-token-id"
  },
  "signature": "RSA-SHA256-SIGNATURE"
}
```

**Преимущества RS256:**
- ✅ Публичный ключ можно безопасно распространять
- ✅ Невозможно подделать токен без приватного ключа
- ✅ Валидация токенов без обращения к Auth Service (локальная проверка)
- ✅ Поддержка key rotation без downtime

**Token Lifetime:**
- Access Token: **15 минут** (короткое время жизни для безопасности)
- Refresh Token: **720 часов (30 дней)** (для удобства пользователей)

**См. также:** `/p/github.com/vondi-global/.passport/security/authentication.md` для деталей JWT

---

### 1.3 TLS/SSL Configuration

**Протокол:** TLS 1.2, TLS 1.3 (приоритет TLS 1.3)
**Сертификаты:** Let's Encrypt (автоматическое обновление)
**Режим:** HTTPS-only (принудительный редирект с HTTP)

```nginx
# nginx.conf
server {
    listen 443 ssl;

    # TLS Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;  # Приоритет клиентским шифрам TLS 1.3

    # Cipher Suites (ECDHE для Forward Secrecy)
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:
                ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    # Session Cache (производительность)
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # HSTS (HTTP Strict Transport Security)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

**Проверка TLS:**
```bash
# Тест подключения
openssl s_client -connect vondi.rs:443 -tls1_3

# Проверка сертификата
curl -vI https://vondi.rs
```

**HSTS Preload:**
- Браузеры принудительно используют HTTPS
- max-age=31536000 (1 год)
- includeSubDomains (все поддомены)
- preload (включение в браузерный preload list)

---

## 2. Защита от атак

### 2.1 CORS (Cross-Origin Resource Sharing)

**Библиотека:** `github.com/gofiber/fiber/v2/middleware/cors`
**Конфигурация:** `backend/internal/middleware/cors.go`

```go
// Allowed Origins (whitelist)
allowedOrigins := "https://vondi.rs,https://www.vondi.rs,https://dev.vondi.rs,https://devapi.vondi.rs,https://auth.vondi.rs,http://localhost:3000,http://localhost:3001"

corsHandler := cors.New(cors.Config{
    AllowOrigins:     allowedOrigins,
    AllowMethods:     "GET,POST,HEAD,PUT,DELETE,PATCH,OPTIONS",
    AllowHeaders:     "Origin,Content-Type,Accept,Authorization,X-Requested-With,X-CSRF-Token",
    ExposeHeaders:    "Content-Length,Set-Cookie",
    AllowCredentials: true,  // Разрешаем cookies
    MaxAge:           300,   // Preflight cache 5 минут
})
```

**Безопасность:**
- ❌ НЕТ wildcards (`*`) - только конкретные домены
- ✅ Credentials только для whitelisted origins
- ✅ Strict header validation
- ✅ Preflight caching для производительности

**BFF Proxy Architecture:**
```
Browser → /api/v2/* (Next.js BFF) → /api/v1/* (Backend)
         └─ httpOnly cookies     └─ Authorization: Bearer <JWT>
```

**Преимущества BFF:**
- Нет CORS проблем (same-origin)
- JWT в httpOnly cookies (не доступны JS → защита от XSS)
- Централизованная авторизация

---

### 2.2 CSRF Protection

**Статус:** Отключена (использование BFF Proxy Architecture)
**Библиотека:** `github.com/gofiber/fiber/v2/middleware/csrf` (доступна, но не используется)

**Почему отключена CSRF:**
1. **BFF Proxy** - все запросы через Next.js backend (same-origin)
2. **SameSite Cookies** - защита от CSRF на уровне браузера
3. **Custom Headers** - требование `X-Requested-With` для AJAX

**Для legacy endpoints (если потребуется):**
```go
// Auth Service: examples/tasktracker/backend/internal/middleware/csrf.go
csrf.New(csrf.Config{
    KeyLookup:      "header:X-CSRF-Token",
    CookieName:     "_csrf",
    CookieSameSite: "Strict",
    CookieSecure:   true,   // Только HTTPS
    CookieHTTPOnly: true,   // Защита от JS
    Expiration:     24 * time.Hour,
})
```

**Cookie Attributes:**
```
Set-Cookie: session_token=xxx; HttpOnly; Secure; SameSite=Strict
```

- **HttpOnly** - защита от XSS (JS не может читать)
- **Secure** - только HTTPS
- **SameSite=Strict** - защита от CSRF

---

### 2.3 XSS (Cross-Site Scripting) Prevention

**Библиотека:** `github.com/microcosm-cc/bluemonday`
**Конфигурация:** `backend/pkg/utils/sanitize.go`

```go
// Strict Policy - удаляет все HTML
func SanitizeText(text string) string {
    policy := bluemonday.StrictPolicy()
    return policy.Sanitize(text)
}

// Basic Policy - разрешает безопасное форматирование
func SanitizeHTML(html string) string {
    policy := bluemonday.NewPolicy()

    // Разрешенные теги
    policy.AllowElements("b", "i", "u", "strong", "em", "code", "pre", "br", "p")

    // Безопасные ссылки
    policy.AllowElements("a")
    policy.AllowAttrs("href").OnElements("a")
    policy.AllowURLSchemes("http", "https", "mailto")
    policy.RequireNoReferrerOnLinks(true)
    policy.AddTargetBlankToFullyQualifiedLinks(true)

    return policy.Sanitize(html)
}
```

**Применение:**
```go
// В хендлерах перед сохранением в БД
import "backend/pkg/utils"

// Пользовательский ввод (отзывы, описания)
sanitizedDescription := utils.SanitizeHTML(req.Description)

// Заголовки, имена (полная очистка)
sanitizedTitle := utils.SanitizeText(req.Title)
```

**Security Headers:**
```go
// backend/internal/middleware/security_headers.go
func SecurityHeaders() fiber.Handler {
    return func(c *fiber.Ctx) error {
        // XSS Protection
        c.Set("X-XSS-Protection", "1; mode=block")

        // Content Security Policy
        c.Set("Content-Security-Policy",
            "default-src 'self'; "+
            "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://accounts.google.com; "+
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; "+
            "img-src 'self' data: https: blob:; "+
            "connect-src 'self' wss: https:")

        // Clickjacking Protection
        c.Set("X-Frame-Options", "DENY")

        // MIME Sniffing Protection
        c.Set("X-Content-Type-Options", "nosniff")

        return c.Next()
    }
}
```

**Тестирование XSS:**
```go
// backend/internal/proj/admin/testing/service/security_tests.go
func testXSSProtection(ctx context.Context, baseURL, token string) {
    xssPayloads := []string{
        "<script>alert('XSS')</script>",
        "<img src=x onerror=alert('XSS')>",
        "javascript:alert('XSS')",
    }

    // Проверяем, что payload не выполняется
    for _, payload := range xssPayloads {
        // Send to API
        // Verify sanitized in response
    }
}
```

---

### 2.4 SQL Injection Protection

**Подход:** Prepared Statements (параметризованные запросы)
**ORM:** Не используется - ручные queries с placeholders
**Библиотека:** `github.com/lib/pq` (PostgreSQL driver)

```go
// ✅ ПРАВИЛЬНО - prepared statement
query := `SELECT * FROM users WHERE email = $1`
row := db.QueryRowContext(ctx, query, email)

// ✅ ПРАВИЛЬНО - multiple parameters
query := `INSERT INTO listings (title, description, price) VALUES ($1, $2, $3)`
_, err := db.ExecContext(ctx, query, title, description, price)

// ❌ НЕПРАВИЛЬНО - string concatenation
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)  // НИКОГДА!
```

**Дополнительная защита:**
```go
// backend/pkg/utils/sanitize.go - для пользовательского ввода
func SanitizeText(text string) string {
    // Удаляет опасные символы и HTML
    policy := bluemonday.StrictPolicy()
    return policy.Sanitize(text)
}
```

**OpenSearch (NoSQL) - безопасность:**
```go
// Экранирование спецсимволов в поисковых запросах
func escapeOpenSearchQuery(query string) string {
    // Экранируем: + - = && || > < ! ( ) { } [ ] ^ " ~ * ? : \ /
    replacer := strings.NewReplacer(
        `\`, `\\`,
        `+`, `\+`,
        `-`, `\-`,
        `=`, `\=`,
        // ... остальные символы
    )
    return replacer.Replace(query)
}
```

**Автоматическое тестирование:**
```go
// backend/internal/proj/admin/testing/service/security_tests.go
func testSQLInjection(ctx context.Context, baseURL, token string) {
    sqlPayloads := []string{
        "' OR '1'='1",
        "'; DROP TABLE users; --",
        "1' UNION SELECT NULL, NULL, NULL--",
    }

    for _, payload := range sqlPayloads {
        // Отправляем в API
        // Проверяем, что НЕТ Internal Server Error (500)
        // Ожидаем: 400 Bad Request или 200 с safe results
    }
}
```

---

### 2.5 Rate Limiting

**Библиотека:** `github.com/gofiber/fiber/v2/middleware/limiter`
**Конфигурация:** `backend/internal/middleware/rate_limiter.go`

**Типы лимитеров:**

| Endpoint | Limit | Window | Назначение |
|----------|-------|--------|------------|
| `/api/v1/auth/login` | 5 | 15m | Защита от brute force паролей |
| `/api/v1/auth/register` | 3 | 1h | Защита от массовой регистрации |
| `/api/v1/auth/refresh` | 5 | 15m | Защита refresh token endpoint |
| `/api/v1/payments/*` | 10 | 1m | Защита платежных операций |
| `/api/v1/webhooks/*` | 100 | 1m | Высокий лимит для внешних сервисов |

**Реализация:**
```go
// Auth Rate Limiter (brute force protection)
func AuthRateLimit() fiber.Handler {
    return limiter.New(limiter.Config{
        Max:        5,                // 5 попыток
        Expiration: 15 * time.Minute, // за 15 минут

        KeyGenerator: func(c *fiber.Ctx) string {
            return "auth_" + c.IP()  // По IP адресу
        },

        LimitReached: func(c *fiber.Ctx) error {
            return utils.ErrorResponse(c, 429, "users.errors.tooManyAttempts")
        },

        SkipFailedRequests: true,      // Не считаем ошибки запроса
        SkipSuccessfulRequests: false, // Считаем успешные попытки
    })
}

// Strict Rate Limiter (после первого нарушения)
func StrictAuthRateLimit() fiber.Handler {
    return limiter.New(limiter.Config{
        Max:        2,              // Только 2 попытки
        Expiration: 1 * time.Hour,  // за 1 час

        KeyGenerator: func(c *fiber.Ctx) string {
            return "strict_auth_" + c.IP()
        },

        LimitReached: func(c *fiber.Ctx) error {
            return utils.ErrorResponse(c, 429, "users.errors.accountTemporarilyLocked")
        },
    })
}

// Payment Rate Limiter (критические операции)
func StrictPaymentRateLimit() fiber.Handler {
    return limiter.New(limiter.Config{
        Max:        3,               // 3 операции
        Expiration: 5 * time.Minute, // за 5 минут

        KeyGenerator: func(c *fiber.Ctx) string {
            userID, _ := c.Locals("user_id").(int)
            return fmt.Sprintf("strict_payment_user_%d", userID)
        },

        LimitReached: func(c *fiber.Ctx) error {
            // Логирование подозрительной активности
            logger.Error().
                Interface("user_id", c.Locals("user_id")).
                Str("ip", c.IP()).
                Msg("CRITICAL: Payment rate limit exceeded - possible fraud")

            // Метрики для мониторинга
            metrics.RecordRateLimitExceeded(c.Path(), userID, c.IP())

            return utils.ErrorResponse(c, 429, "payments.errors.suspiciousActivity")
        },
    })
}
```

**Конфигурация через .env:**
```bash
# Auth Service
VONDIAUTH_SECURITY_RATE_LIMIT_ENABLED=true
VONDIAUTH_SECURITY_RATE_LIMIT_LOGIN=5/15m
VONDIAUTH_SECURITY_RATE_LIMIT_REGISTER=3/1h
VONDIAUTH_SECURITY_RATE_LIMIT_TOKEN_REFRESH=30/1m
```

**Redis для distributed rate limiting:**
```go
// Для кластера с несколькими инстансами
import "github.com/gofiber/storage/redis"

storage := redis.New(redis.Config{
    Host: "redis:6379",
    Port: 6379,
})

limiter.New(limiter.Config{
    Storage: storage,  // Общее хранилище счетчиков
    // ... остальная конфигурация
})
```

---

### 2.6 File Upload Security

**Библиотека:** Custom validation
**Конфигурация:** `backend/pkg/utils/file_validation.go`

```go
const (
    MaxImageSize = 10 * 1024 * 1024  // 10 MB
    MaxFileSize  = 20 * 1024 * 1024  // 20 MB
)

var (
    AllowedImageTypes = []string{"image/jpeg", "image/png", "image/webp", "image/gif"}
    AllowedFileTypes  = []string{"application/pdf", "application/zip"}
)

func ValidateImageUpload(file *multipart.FileHeader) error {
    // 1. Проверка размера
    if file.Size > MaxImageSize {
        return errors.New("file.size.too_large")
    }

    // 2. Проверка MIME type (header)
    contentType := file.Header.Get("Content-Type")
    if !isAllowedImageType(contentType) {
        return errors.New("file.type.not_allowed")
    }

    // 3. Проверка magic bytes (фактический тип файла)
    fileContent, err := file.Open()
    if err != nil {
        return err
    }
    defer fileContent.Close()

    buffer := make([]byte, 512)
    _, err = fileContent.Read(buffer)
    if err != nil {
        return err
    }

    detectedType := http.DetectContentType(buffer)
    if !isAllowedImageType(detectedType) {
        return errors.New("file.type.mismatch")
    }

    // 4. Дополнительная валидация изображения
    fileContent.Seek(0, 0)
    img, _, err := image.Decode(fileContent)
    if err != nil {
        return errors.New("file.invalid.image")
    }

    // 5. Проверка разрешения (опционально)
    bounds := img.Bounds()
    if bounds.Dx() > 10000 || bounds.Dy() > 10000 {
        return errors.New("file.resolution.too_large")
    }

    return nil
}
```

**Безопасное хранилище:**
```go
// MinIO S3-compatible storage
// - Изоляция публичных файлов от приватных
// - Bucket policies (read-only для listings)
// - Генерация уникальных имен файлов (UUID)
// - Отдельные buckets для разных типов данных

buckets:
  - listings/          # Публичные изображения товаров
  - chat-files/        # Приватные файлы чата
  - user-avatars/      # Публичные аватары
  - documents/         # Приватные документы (KYC, invoices)
```

**Автоматическое тестирование:**
```go
// backend/internal/proj/admin/testing/service/security_tests.go
func testFileUploadValidation(ctx context.Context, baseURL, token string) {
    tests := []struct {
        name     string
        file     []byte
        mimeType string
        expectOK bool
    }{
        {"valid_jpeg", validJPEG, "image/jpeg", true},
        {"malicious_exe", exeFile, "image/jpeg", false},  // Fake MIME
        {"php_shell", phpShell, "image/png", false},      // Code injection
        {"svg_xss", svgXSS, "image/svg+xml", false},      // XSS в SVG
    }

    for _, tt := range tests {
        // Upload file
        // Verify result matches expectation
    }
}
```

---

## 3. Приватность данных

### 3.1 Геолокация (Privacy Levels)

**Библиотека:** Custom implementation
**Конфигурация:** `backend/pkg/utils/geo_privacy.go`

```go
type PrivacyLevel string

const (
    PrivacyExact        PrivacyLevel = "exact"       // Точный адрес (44.787197, 20.457273)
    PrivacyStreet       PrivacyLevel = "street"      // Улица без номера (44.787, 20.457)
    PrivacyApproximate  PrivacyLevel = "approximate" // Округление до 3 знаков
    PrivacyDistrict     PrivacyLevel = "district"    // Район/микрорайон (44.79, 20.46)
    PrivacyCity         PrivacyLevel = "city"        // Центр города (44.8, 20.5)
    PrivacyCityOnly     PrivacyLevel = "city_only"   // Только название города
    PrivacyHidden       PrivacyLevel = "hidden"      // Координаты скрыты (0, 0)
)

func GetCoordinatesWithGeocoding(
    ctx context.Context,
    lat, lng float64,
    address string,
    privacyLevel PrivacyLevel,
    geocoder GeocodingService,
) (float64, float64, error) {
    switch privacyLevel {
    case PrivacyExact:
        return lat, lng, nil

    case PrivacyStreet, PrivacyApproximate:
        // Геокодирование улицы (без номера дома)
        if address != "" && geocoder != nil {
            streetLat, streetLng, err := geocoder.GetStreetCoordinates(ctx, address)
            if err == nil {
                return streetLat, streetLng, nil
            }
        }
        // Fallback: округление
        return roundToDecimalPlaces(lat, 3), roundToDecimalPlaces(lng, 3), nil

    case PrivacyDistrict:
        // Центр района через Nominatim API
        // Fallback: округление до 2 знаков

    case PrivacyCity, PrivacyCityOnly:
        // Центр города через Nominatim API
        // Fallback: округление до 1 знака

    case PrivacyHidden:
        return 0, 0, nil
    }
}
```

**Применение в listings:**
```go
// При создании объявления
listing.Latitude, listing.Longitude, err = utils.GetCoordinatesWithGeocoding(
    ctx,
    req.Latitude,
    req.Longitude,
    req.Address,
    req.PrivacyLevel,  // От пользователя
    geocoder,
)

// При отображении на карте - используются уже обработанные координаты
```

**Geocoding через Nominatim (OpenStreetMap):**
```go
type NominatimGeocoding struct{}

func (n *NominatimGeocoding) GetStreetCoordinates(ctx context.Context, address string) (float64, float64, error) {
    // Удаляем номер дома
    streetOnly := removeHouseNumber(address)

    // Запрос к Nominatim API
    url := fmt.Sprintf("https://nominatim.openstreetmap.org/search?format=json&q=%s", url.QueryEscape(streetOnly))

    req, _ := http.NewRequest("GET", url, nil)
    req.Header.Add("User-Agent", "Vondi-Platform/1.0")

    // Parse response, extract lat/lng
    return lat, lng, nil
}
```

---

### 3.2 GDPR Compliance

**Права пользователей:**

| Право | Реализация | API Endpoint |
|-------|-----------|--------------|
| Right to Access | Экспорт всех данных пользователя | `GET /api/v1/users/profile/export` |
| Right to Rectification | Редактирование профиля | `PUT /api/v1/users/profile` |
| Right to Erasure | Мягкое удаление (soft delete) | `DELETE /api/v1/users/profile` |
| Right to Portability | JSON export с историей | `GET /api/v1/users/data-export` |
| Right to Object | Отключение обработки данных | `PUT /api/v1/users/privacy-settings` |

**Soft Delete (мягкое удаление):**
```sql
-- Пользователи НЕ удаляются физически
-- Устанавливается флаг deleted_at
UPDATE users SET deleted_at = NOW() WHERE id = $1;

-- Восстановление возможно в течение 30 дней
UPDATE users SET deleted_at = NULL WHERE id = $1 AND deleted_at > NOW() - INTERVAL '30 days';

-- Permanent delete (после 30 дней, cron job)
DELETE FROM users WHERE deleted_at < NOW() - INTERVAL '30 days';
```

**Data Retention Policy:**
```go
// backend/internal/domain/models/user.go
type DataRetentionPolicy struct {
    UserData        time.Duration // 30 дней после soft delete
    ChatMessages    time.Duration // 90 дней
    TransactionLogs time.Duration // 7 лет (финансовые записи)
    Analytics       time.Duration // 2 года (анонимизированные)
}
```

**Consent Management:**
```go
// Auth Service: internal/domain/user.go
type User struct {
    // ...
    EmailVerified      bool      `json:"email_verified"`
    PhoneVerified      bool      `json:"phone_verified"`
    MarketingConsent   bool      `json:"marketing_consent"`
    AnalyticsConsent   bool      `json:"analytics_consent"`
    ConsentUpdatedAt   time.Time `json:"consent_updated_at"`
}
```

**Privacy Settings:**
```go
// backend/internal/domain/models/user.go
type PrivacySettings struct {
    UserID             int          `json:"user_id"`
    ProfileVisibility  string       `json:"profile_visibility"`   // public, friends, private
    ShowEmail          bool         `json:"show_email"`
    ShowPhone          bool         `json:"show_phone"`
    LocationPrivacy    PrivacyLevel `json:"location_privacy"`     // exact, street, city, hidden
    OnlineStatus       bool         `json:"online_status"`
    LastSeenVisibility string       `json:"last_seen_visibility"` // everyone, contacts, nobody
}
```

---

### 3.3 PII (Personally Identifiable Information) Protection

**Категории PII:**

| Тип данных | Хранилище | Шифрование | Доступ |
|-----------|-----------|-----------|--------|
| Email | PostgreSQL | Plaintext (indexed) | User, Admin |
| Phone | PostgreSQL | Plaintext (indexed) | User, Admin |
| Password | PostgreSQL | Bcrypt hash | - |
| Full Name | PostgreSQL | Plaintext | User, Admin |
| Address | PostgreSQL | Plaintext | User (privacy levels) |
| Passport/ID | PostgreSQL | Encrypted | Admin only |
| Payment Cards | External (Stripe) | Tokenized | - |

**Логирование PII:**
```go
// ❌ НЕПРАВИЛЬНО - PII в логах
logger.Info().Str("email", user.Email).Msg("User logged in")

// ✅ ПРАВИЛЬНО - только ID
logger.Info().Int("user_id", user.ID).Msg("User logged in")

// ✅ Маскирование email в логах (если необходимо)
func maskEmail(email string) string {
    parts := strings.Split(email, "@")
    if len(parts) != 2 {
        return "***@***"
    }
    // u***@example.com
    return string(parts[0][0]) + "***@" + parts[1]
}
```

**Database Access Control:**
```sql
-- Ограничение доступа к PII на уровне БД
-- Read-only роль для analytics
CREATE ROLE analytics_readonly WITH LOGIN PASSWORD 'xxx';
GRANT SELECT ON users(id, created_at, country, city) TO analytics_readonly;
-- НЕТ доступа к email, phone, full_name

-- Admin роль (полный доступ)
CREATE ROLE admin_user WITH LOGIN PASSWORD 'xxx';
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO admin_user;
```

---

## 4. Audit Logging

**Библиотека:** `github.com/rs/zerolog`
**Конфигурация:** `backend/internal/logger`

**Категории логов:**

| Категория | Level | События |
|-----------|-------|---------|
| Authentication | INFO | Login, Logout, Register, Password Reset |
| Authorization | WARN | Access denied, Role change |
| Data Access | INFO | Profile view, Export data |
| Data Modification | INFO | Profile update, Settings change |
| Financial | INFO | Payment created, Refund, Withdrawal |
| Security | WARN/ERROR | Rate limit, Failed login, Suspicious activity |
| Admin Actions | INFO | User ban, Role assignment, Data deletion |

**Реализация:**
```go
// Auth Service: internal/repository/postgres/user.go
func (r *UserRepository) RecordLogin(ctx context.Context, userID int, ipAddress, userAgent string) error {
    query := `
        INSERT INTO user_login_history (user_id, ip_address, user_agent, logged_in_at)
        VALUES ($1, $2, $3, NOW())
    `
    _, err := r.db.ExecContext(ctx, query, userID, ipAddress, userAgent)

    // Дополнительно логируем в structured logs
    logger.Info().
        Int("user_id", userID).
        Str("ip", ipAddress).
        Str("user_agent", userAgent).
        Msg("User logged in successfully")

    return err
}
```

**Таблица аудита:**
```sql
CREATE TABLE user_login_history (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    ip_address VARCHAR(45),  -- IPv6 support
    user_agent TEXT,
    logged_in_at TIMESTAMP NOT NULL DEFAULT NOW(),

    INDEX idx_user_login_user_id (user_id),
    INDEX idx_user_login_time (logged_in_at DESC)
);

-- Retention policy: 90 дней
DELETE FROM user_login_history WHERE logged_in_at < NOW() - INTERVAL '90 days';
```

**Критические события (требуют логирования):**
```go
// Payment operations
logger.Info().
    Int("user_id", userID).
    Str("payment_id", paymentID).
    Float64("amount", amount).
    Str("currency", "RSD").
    Msg("Payment created")

// Admin actions
logger.Warn().
    Int("admin_id", adminID).
    Int("target_user_id", targetUserID).
    Str("action", "ban_user").
    Str("reason", reason).
    Msg("User banned by admin")

// Security incidents
logger.Error().
    Str("ip", ipAddress).
    Int("failed_attempts", attemptCount).
    Msg("Multiple failed login attempts - possible brute force")
```

**Log rotation и хранение:**
```yaml
# Docker logging driver
services:
  backend:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"      # Максимум 10 MB на файл
        max-file: "3"        # Хранить 3 ротированных файла
        compress: "true"     # Сжимать старые логи
```

---

## 5. Secure Cookies & Session Management

### 5.1 Cookie Attributes

**Все cookies используют максимальные security настройки:**

```go
// Auth Service: internal/transport/http/handlers/auth.go
cookie := &http.Cookie{
    Name:     "session_token",
    Value:    refreshToken,
    Path:     "/",
    Domain:   "vondi.rs",
    Expires:  time.Now().Add(30 * 24 * time.Hour),
    HttpOnly: true,   // ✅ Защита от XSS (JS не может читать)
    Secure:   true,   // ✅ Только HTTPS
    SameSite: http.SameSiteStrictMode,  // ✅ Защита от CSRF
}
http.SetCookie(w, cookie)
```

**Атрибуты:**

| Атрибут | Значение | Назначение |
|---------|----------|-----------|
| HttpOnly | true | JavaScript не может читать cookie (защита от XSS) |
| Secure | true | Передача только по HTTPS (защита от MITM) |
| SameSite | Strict | Защита от CSRF (cookie не отправляется в cross-site запросах) |
| Domain | vondi.rs | Shared cookies для поддоменов |
| Path | / | Cookie доступен для всего сайта |
| MaxAge | 2592000 (30 дней) | Автоматическое истечение |

**SameSite варианты:**
- **Strict** - cookie НЕ отправляется в cross-site запросах (максимальная безопасность)
- **Lax** - cookie отправляется только в safe HTTP методах (GET)
- **None** - cookie отправляется всегда (требует Secure=true)

---

### 5.2 Session Management

**Refresh Token Architecture:**

```go
// Auth Service: internal/domain/refresh_token.go
type RefreshToken struct {
    ID           int       `json:"id"`
    UserID       int       `json:"user_id"`
    TokenHash    string    `json:"token_hash"`      // SHA-256 hash
    FamilyID     uuid.UUID `json:"family_id"`       // Rotation family
    DeviceID     string    `json:"device_id"`       // Unique device
    DeviceName   string    `json:"device_name"`     // "Chrome on MacOS"
    IPAddress    string    `json:"ip_address"`
    UserAgent    string    `json:"user_agent"`
    ExpiresAt    time.Time `json:"expires_at"`
    IssuedAt     time.Time `json:"issued_at"`
    LastUsedAt   *time.Time `json:"last_used_at"`
    RevokedAt    *time.Time `json:"revoked_at"`
    RevokedBy    *int      `json:"revoked_by"`
    RevokeReason *string   `json:"revoke_reason"`
}
```

**Token Rotation (защита от replay attacks):**

1. User login → генерируется Refresh Token + Family ID
2. User запрашивает новый Access Token → Refresh Token ротируется (старый инвалидируется)
3. Если старый Refresh Token использован повторно → вся семья токенов отзывается

```go
// Auth Service: internal/service/token/refresh.go
func (s *RefreshService) RotateRefreshToken(
    oldToken *domain.RefreshToken,
    user *domain.User,
) (*domain.RefreshToken, string, error) {
    // 1. Проверяем, что токен не был использован ранее
    if oldToken.LastUsedAt != nil {
        // REPLAY ATTACK DETECTED!
        // Отзываем всю семью токенов
        s.tokenRepo.RevokeTokenFamily(ctx, oldToken.FamilyID, "replay_attack_detected")
        return nil, "", ErrTokenReused
    }

    // 2. Отмечаем старый токен как использованный
    s.tokenRepo.RecordTokenUse(ctx, oldToken.ID)

    // 3. Генерируем новый токен (та же FamilyID)
    newToken := &domain.RefreshToken{
        UserID:   user.ID,
        FamilyID: oldToken.FamilyID,  // ✅ Та же семья
        // ... остальные поля
    }

    tokenString, _ := s.jwtService.GenerateRefreshToken(user, oldToken.FamilyID, oldToken.DeviceID)
    newToken.TokenHash = s.HashToken(tokenString)

    s.tokenRepo.CreateRefreshToken(ctx, newToken)

    return newToken, tokenString, nil
}
```

**Revocation API:**
```go
// Auth Service: endpoints
POST /api/v1/auth/logout              // Отзыв текущей сессии
POST /api/v1/auth/logout-all          // Отзыв всех сессий пользователя
POST /api/v1/auth/revoke-device/:id   // Отзыв конкретного устройства
```

**Redis Cache для быстрой валидации:**
```go
// Blacklist для отозванных Access Tokens
func IsAccessTokenRevoked(ctx context.Context, jti string) (bool, error) {
    // Проверка в Redis (быстро)
    exists, err := cache.IsTokenBlacklisted(ctx, jti)
    if err == nil {
        return exists, nil
    }

    // Fallback на PostgreSQL
    return tokenRepo.IsAccessTokenRevoked(ctx, jti)
}
```

---

## 6. Input Validation

**Библиотека:** `github.com/go-playground/validator/v10`
**Применение:** DTO validation во всех handlers

```go
// Структура с validation tags
type CreateListingRequest struct {
    Title       string  `json:"title" validate:"required,min=3,max=100"`
    Description string  `json:"description" validate:"required,min=10,max=5000"`
    Price       float64 `json:"price" validate:"required,gt=0"`
    Email       string  `json:"email" validate:"required,email"`
    Phone       string  `json:"phone" validate:"omitempty,e164"` // International format
    URL         string  `json:"url" validate:"omitempty,url"`
    Category    string  `json:"category" validate:"required,oneof=electronics clothing furniture"`
}

// Валидация в handler
func (h *Handler) CreateListing(c *fiber.Ctx) error {
    var req CreateListingRequest
    if err := c.BodyParser(&req); err != nil {
        return utils.ErrorResponse(c, 400, "invalid.request.body")
    }

    // Validate struct
    if err := h.validator.Struct(req); err != nil {
        return utils.ValidationErrorResponse(c, err)
    }

    // Sanitize HTML content
    req.Description = utils.SanitizeHTML(req.Description)
    req.Title = utils.SanitizeText(req.Title)

    // ... business logic
}
```

**Validation tags:**

| Tag | Описание | Пример |
|-----|----------|--------|
| `required` | Обязательное поле | `validate:"required"` |
| `min`, `max` | Длина строки или значение числа | `validate:"min=3,max=100"` |
| `email` | Email формат | `validate:"email"` |
| `e164` | Международный телефон | `validate:"e164"` |
| `url` | URL формат | `validate:"url"` |
| `oneof` | Один из списка | `validate:"oneof=a b c"` |
| `gt`, `gte`, `lt`, `lte` | Сравнение чисел | `validate:"gt=0"` |
| `uuid` | UUID формат | `validate:"uuid"` |
| `datetime` | ISO8601 дата/время | `validate:"datetime=2006-01-02"` |

---

## 7. Security Monitoring & Metrics

**Prometheus Metrics:**
```go
// backend/internal/monitoring/metrics.go
var (
    // Authentication metrics
    loginAttempts = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "auth_login_attempts_total",
            Help: "Total login attempts",
        },
        []string{"status", "method"},  // success/failed, password/oauth
    )

    // Rate limiting metrics
    rateLimitExceeded = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "rate_limit_exceeded_total",
            Help: "Rate limit violations",
        },
        []string{"endpoint", "user_id"},
    )

    // Security incidents
    securityIncidents = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "security_incidents_total",
            Help: "Security incidents detected",
        },
        []string{"type"},  // sql_injection, xss, csrf, etc.
    )
)

func RecordRateLimitExceeded(endpoint string, userID int, ip string) {
    rateLimitExceeded.WithLabelValues(endpoint, fmt.Sprintf("%d", userID)).Inc()

    // Также логируем для детального анализа
    logger.Warn().
        Str("endpoint", endpoint).
        Int("user_id", userID).
        Str("ip", ip).
        Msg("Rate limit exceeded")
}
```

**Health Checks:**
```go
// backend/cmd/api/main.go
app.Get("/health", func(c *fiber.Ctx) error {
    return c.JSON(fiber.Map{
        "status": "ok",
        "version": version.Version,
        "checks": fiber.Map{
            "database":    checkDatabase(),
            "redis":       checkRedis(),
            "opensearch":  checkOpenSearch(),
            "auth_service": checkAuthService(),
        },
    })
})
```

---

## 8. Checklist перед продакшном

### 8.1 Credentials & Keys

- [ ] RSA ключи сгенерированы (2048 bit минимум)
- [ ] Private key НЕ в git (проверить .gitignore)
- [ ] Public key можно распространять
- [ ] .env файлы в .gitignore
- [ ] Secrets в environment variables (НЕ hardcoded)
- [ ] Database passwords уникальные и сложные
- [ ] API keys rotation policy (30-90 дней)

### 8.2 HTTPS/TLS

- [ ] Let's Encrypt сертификаты установлены
- [ ] Auto-renewal настроен (certbot)
- [ ] TLS 1.2+ только (отключить TLS 1.0/1.1)
- [ ] HSTS enabled (max-age=31536000)
- [ ] Тест: https://www.ssllabs.com/ssltest/

### 8.3 Headers & CORS

- [ ] Security headers установлены (X-Frame-Options, CSP, etc.)
- [ ] CORS whitelist (без wildcards)
- [ ] SameSite cookies (Strict или Lax)
- [ ] HttpOnly + Secure для всех auth cookies

### 8.4 Rate Limiting

- [ ] Auth endpoints protected (5/15m)
- [ ] Payment endpoints protected (10/1m)
- [ ] Webhook endpoints protected (100/1m)
- [ ] Redis для distributed rate limiting (если кластер)
- [ ] Логирование rate limit violations

### 8.5 Input Validation

- [ ] Validator на всех DTOs
- [ ] HTML sanitization (bluemonday)
- [ ] File upload validation (MIME, magic bytes, size)
- [ ] SQL injection tests passed
- [ ] XSS tests passed

### 8.6 Logging & Monitoring

- [ ] Audit logging для критических операций
- [ ] PII НЕ в логах (только user_id)
- [ ] Log rotation настроен (max 10MB, keep 3 files)
- [ ] Prometheus metrics configured
- [ ] Alerting rules для security incidents

### 8.7 Database

- [ ] Prepared statements ВЕЗДЕ
- [ ] Connection pooling (max 100 connections)
- [ ] Backup strategy (daily + retention 30 days)
- [ ] Soft delete для users (GDPR compliance)
- [ ] Encryption at rest (PostgreSQL TDE)

### 8.8 Dependencies

- [ ] Go modules updated (go get -u)
- [ ] Vulnerability scan (govulncheck)
- [ ] Outdated packages check (go list -u -m all)
- [ ] License compliance check

---

## 9. Compliance & Standards

### 9.1 Следуем стандартам:

| Стандарт | Область | Статус |
|----------|---------|--------|
| **OWASP Top 10** | Web security | ✅ Implemented |
| **GDPR** | Data privacy (EU) | ✅ Compliant |
| **PCI DSS** | Payment card data | ⚠️ Partial (Stripe handles) |
| **ISO 27001** | Information security | 🔄 In progress |
| **CWE/SANS Top 25** | Software weaknesses | ✅ Mitigated |

### 9.2 OWASP Top 10 Coverage:

| #  | Угроза | Защита |
|----|--------|--------|
| A01 | Broken Access Control | ✅ RBAC, JWT validation |
| A02 | Cryptographic Failures | ✅ TLS 1.3, bcrypt, RS256 |
| A03 | Injection | ✅ Prepared statements, sanitization |
| A04 | Insecure Design | ✅ Security by design |
| A05 | Security Misconfiguration | ✅ Security headers, HSTS |
| A06 | Vulnerable Components | ✅ Dependency scanning |
| A07 | Authentication Failures | ✅ Rate limiting, MFA ready |
| A08 | Software/Data Integrity | ✅ Audit logging |
| A09 | Logging Failures | ✅ Structured logging, rotation |
| A10 | SSRF | ✅ URL validation, whitelist |

---

## 10. Incident Response

### 10.1 Security Incident Classification:

| Severity | Примеры | Response Time | Actions |
|----------|---------|---------------|---------|
| **Critical** | Data breach, RCE, Mass account takeover | < 1 hour | Immediate shutdown, forensics, notification |
| **High** | SQL injection, Authentication bypass | < 4 hours | Patch deployment, user notification |
| **Medium** | XSS, CSRF, Rate limit bypass | < 24 hours | Fix and deploy, monitoring |
| **Low** | Info disclosure, Deprecated TLS | < 1 week | Scheduled fix |

### 10.2 Incident Response Plan:

1. **Detection** - Automated alerts, monitoring, user reports
2. **Containment** - Isolate affected systems, revoke tokens
3. **Eradication** - Patch vulnerability, remove backdoors
4. **Recovery** - Restore from backup, verify integrity
5. **Lessons Learned** - Post-mortem, update procedures

### 10.3 Contact Points:

```yaml
Security Team:
  Email: security@vondi.rs
  Emergency: +381-XXX-XXXXX
  PGP Key: [публичный ключ для шифрования уведомлений]

Responsible Disclosure:
  - Report security vulnerabilities to security@vondi.rs
  - Allow 90 days for fix before public disclosure
  - Bug bounty program (coming soon)
```

---

## 11. Ссылки и документация

### Внутренняя документация:
- [Authentication Security](/p/github.com/vondi-global/.passport/security/authentication.md)
- [Authorization & RBAC](/p/github.com/vondi-global/.passport/security/authorization.md)
- [Auth Service Integration](/p/github.com/vondi-global/auth/docs/MARKETPLACE_INTEGRATION_SPEC.md)

### Внешние ресурсы:
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Cheat Sheets](https://cheatsheetseries.owasp.org/)
- [GDPR Official Text](https://gdpr-info.eu/)
- [CWE Top 25](https://cwe.mitre.org/top25/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)

### Security Testing Tools:
- `govulncheck` - Go vulnerability scanner
- `golangci-lint` - Static analysis (security linters enabled)
- OWASP ZAP - Automated web app security testing
- Burp Suite - Manual penetration testing

---

**Последнее обновление:** 2025-12-21
**Ответственный:** Security Team
**Следующий review:** 2025-03-21 (quarterly)
