# Паспорт: База данных auth_dev_db

> Обновлено: 2025-12-21

## Идентификация

| Параметр | Значение |
|----------|----------|
| Тип | PostgreSQL 15 Alpine |
| Image | postgres:15-alpine |
| Хост | localhost |
| Порт | 25432 |
| Имя БД | auth_dev_db |
| Пользователь | auth_user |
| Пароль | auth_pass |
| Контейнер | vondi-dev-postgres-auth |

## КРИТИЧНО

**Это ОТДЕЛЬНАЯ база данных Auth Service!**
- НЕ используй порт 5433 (это монолит vondi_db)
- НЕ используй порт 35434 (это Listings microservice)
- Auth Service имеет СВОЮ независимую БД
- **Назначение:** Аутентификация, авторизация, управление пользователями

## Подключение

```bash
# Через localhost
psql "postgres://auth_user:auth_pass@localhost:25432/auth_dev_db?sslmode=disable"

# Через Docker
docker exec -it vondi-dev-postgres-auth psql -U auth_user -d auth_dev_db

# Для дампа
PGPASSWORD=auth_pass pg_dump \
  -h localhost -p 25432 -U auth_user -d auth_dev_db \
  --no-owner --no-acl --column-inserts --inserts \
  -f auth_dump_$(date +%Y%m%d_%H%M%S).sql

# Для импорта
PGPASSWORD=auth_pass psql \
  -h localhost -p 25432 -U auth_user -d auth_dev_db \
  -f auth_dump.sql
```

## Статистика

| Параметр | Значение |
|----------|----------|
| Всего таблиц | 7 |
| Бизнес-таблиц | 7 |
| Функций | 5 |
| Триггеров | 8 |
| Миграций | 9 |
| Путь к миграциям | /p/github.com/vondi-global/auth/migrations/ |

## Таблицы (полный список - 7 таблиц)

### 1. Пользователи (Users)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| users | Основная таблица пользователей | BIGSERIAL |
| user_addresses | Адресная книга пользователей для checkout | BIGSERIAL |
| user_checkout_profiles | Профили контактной информации для checkout | BIGSERIAL |

**Особенности users:**
- UUID для внешних ссылок: `uuid` (gen_random_uuid)
- Email: `email` (unique), `email_normalized` (lowercase index)
- Multi-provider auth: `password_hash`, `google_id`, `facebook_id`, `apple_id`
- Profile: `name`, `picture_url`, `phone`, `bio`, `city`, `country`, `timezone`
- Verification: `email_verified`, `phone_verified`, `term_accepted`
- Security: `two_factor_enabled`, `two_factor_secret`, `backup_codes[]`
- Roles: `roles[]` (text array, default: ['user'])
- Status: `is_active`, `is_deleted`, `deleted_at`
- Login tracking: `last_login_at`, `last_login_ip`, `last_login_user_agent`
- Brute force protection: `failed_login_attempts`, `locked_until`

**Особенности user_addresses:**
- Мультиязычность: `street_sr_latin` (required), `street_en`, `street_user_locale`
- Геокодирование: `latitude`, `longitude`, `is_validated`, `confidence_score`
- Post Express: `post_express_settlement_id`, `post_express_street_id`
- Флаги: `is_default`, `is_last_used` (только один на пользователя)
- Лимит: максимум 10 адресов на пользователя (trigger)

**Особенности user_checkout_profiles:**
- Один профиль на пользователя (UNIQUE constraint на user_id)
- Контакты: `first_name`, `last_name`, `email`, `phone` (могут отличаться от основного профиля)
- Связь с адресом: `preferred_address_id` (FK к user_addresses)

### 2. Токены и сессии (Tokens & Sessions)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| refresh_tokens | Refresh токены (JWT rotation) | BIGSERIAL |
| revoked_access_tokens | Отозванные access токены | TEXT (jti) |
| user_sessions | Активные сессии пользователей | BIGSERIAL |

**Особенности refresh_tokens:**
- Token rotation: `jti` (uuid), `token_hash` (bcrypt)
- Family tracking: `family_id` (uuid), `parent_token_id` (self FK)
- Device info: `device_id`, `device_name`, `ip_address`, `user_agent`
- Expiration: `expires_at`, `is_revoked`, `revoked_at`, `revoked_reason`
- Reuse detection: `use_count`, `used_at` (trigger для детекции повторного использования)
- Автоматическая ревокация всех токенов семьи при детекции reuse

**Особенности revoked_access_tokens:**
- Primary key: `jti` (JWT ID)
- Cleanup: `expires_at` (для удаления истекших записей)
- Причина отзыва: `revoked_reason`

**Особенности user_sessions:**
- Session tracking: `session_id` (uuid unique)
- Device fingerprint: `device_type`, `browser`, `os`
- Location: `location_country`, `location_city` (from IP)
- Activity: `last_activity_at`, `activity_count`
- Status: `is_active`, `terminated_at`, `termination_reason`
- Связь с токеном: `refresh_token_id` (FK)

### 3. Аудит и логи (Audit & Logs)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| audit_log | Журнал аудита действий | BIGSERIAL |

**Особенности audit_log:**
- Действие: `action` (login, logout, token_reuse_detected, etc)
- Ресурс: `resource`, `resource_id`
- Пользователь: `user_id`, `ip_address`, `user_agent`, `request_id`
- Изменения: `old_values` (JSONB), `new_values` (JSONB)
- Метаданные: `metadata` (JSONB)
- Результат: `success` (boolean), `error_message`

### 4. Service Accounts (S2S Authentication)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| service_accounts | API ключи для межсервисного взаимодействия | SERIAL |

**Особенности service_accounts:**
- Идентификация: `service_name` (unique), `description`
- API Key: `api_key` (plain for lookup), `api_key_hash` (bcrypt)
- Связь с user: `user_id` (FK к users с provider='service')
- Status: `is_active`
- Tracking: `last_used_at` (обновляется при обмене API key на JWT)

## Схемы ключевых таблиц

### users (Main Users Table)

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID DEFAULT gen_random_uuid() UNIQUE,

    -- Authentication
    email TEXT NOT NULL UNIQUE,
    email_normalized TEXT,
    password_hash TEXT,

    -- OAuth providers
    google_id TEXT UNIQUE,
    facebook_id TEXT UNIQUE,
    apple_id TEXT UNIQUE,
    provider VARCHAR(20) NOT NULL, -- 'email', 'google', 'facebook', 'apple', 'service'

    -- Profile
    name TEXT NOT NULL,
    picture_url TEXT,
    phone TEXT,
    bio TEXT,
    city TEXT,
    country TEXT,
    timezone TEXT DEFAULT 'UTC',

    -- Verification
    phone_verified BOOLEAN DEFAULT FALSE NOT NULL,
    email_verified BOOLEAN DEFAULT FALSE NOT NULL,
    term_accepted BOOLEAN DEFAULT FALSE NOT NULL,

    -- Two-Factor Authentication
    two_factor_enabled BOOLEAN DEFAULT FALSE NOT NULL,
    two_factor_secret TEXT,
    backup_codes TEXT[],

    -- Authorization
    roles TEXT[] DEFAULT ARRAY['user']::TEXT[],

    -- Status
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    is_deleted BOOLEAN DEFAULT FALSE NOT NULL,
    deleted_at TIMESTAMPTZ,

    -- Security
    last_login_at TIMESTAMPTZ,
    last_login_ip TEXT,
    last_login_user_agent TEXT,
    failed_login_attempts BIGINT DEFAULT 0,
    locked_until TIMESTAMPTZ,

    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_users_uuid ON users(uuid);
CREATE UNIQUE INDEX idx_users_google_id ON users(google_id);
CREATE UNIQUE INDEX idx_users_facebook_id ON users(facebook_id);
CREATE UNIQUE INDEX idx_users_apple_id ON users(apple_id);
CREATE INDEX idx_users_email_normalized ON users(LOWER(email));
CREATE INDEX idx_users_provider ON users(provider);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
CREATE INDEX idx_users_last_login ON users(last_login_at DESC);
CREATE INDEX idx_users_roles ON users USING gin(roles);
CREATE INDEX idx_users_country ON users(country) WHERE country IS NOT NULL;
CREATE INDEX idx_users_city ON users(city) WHERE city IS NOT NULL;
CREATE INDEX idx_users_timezone ON users(timezone);

-- Trigger
CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

**Возможные значения provider:**
- `local` - регистрация по email/password
- `google` - OAuth Google
- `facebook` - OAuth Facebook
- `apple` - OAuth Apple
- `service` - сервисный аккаунт (для service-to-service auth)

**Роли (массив roles):**
- `user` - базовый пользователь (назначается автоматически)
- `admin` - администратор системы
- `moderator` - модератор контента
- `warehouse_owner` - владелец склада
- `warehouse_staff` - сотрудник склада
- `courier` - курьер
- `seller` - продавец

### refresh_tokens (Token Rotation with Reuse Detection)

```sql
CREATE TABLE refresh_tokens (
    id BIGSERIAL PRIMARY KEY,
    jti UUID DEFAULT gen_random_uuid() UNIQUE,
    user_id BIGINT NOT NULL,

    -- Token
    token_hash TEXT NOT NULL UNIQUE,
    family_id UUID NOT NULL,
    parent_token_id BIGINT,

    -- Device info
    device_id TEXT,
    device_name TEXT,
    ip_address TEXT,
    user_agent TEXT,

    -- Expiration
    expires_at TIMESTAMPTZ NOT NULL,
    is_revoked BOOLEAN DEFAULT FALSE,
    revoked_at TIMESTAMPTZ,
    revoked_reason TEXT,

    -- Usage tracking (for reuse detection)
    used_at TIMESTAMPTZ,
    use_count BIGINT DEFAULT 0,

    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_refresh_tokens_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_refresh_tokens_parent_token FOREIGN KEY (parent_token_id)
        REFERENCES refresh_tokens(id)
);

-- Indexes
CREATE UNIQUE INDEX idx_refresh_tokens_jti ON refresh_tokens(jti);
CREATE UNIQUE INDEX idx_refresh_tokens_token_hash ON refresh_tokens(token_hash);
CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id) WHERE NOT is_revoked;
CREATE INDEX idx_refresh_tokens_family_id ON refresh_tokens(family_id);
CREATE INDEX idx_refresh_tokens_expires_at ON refresh_tokens(expires_at) WHERE NOT is_revoked;
CREATE INDEX idx_refresh_tokens_device ON refresh_tokens(device_id, user_id);

-- Reuse detection trigger
CREATE TRIGGER detect_refresh_token_reuse
    AFTER UPDATE OF use_count ON refresh_tokens
    FOR EACH ROW
    WHEN (NEW.use_count > 1)
    EXECUTE FUNCTION detect_token_reuse();
```

**Token Family механизм:**
- `family_id` - UUID семейства токенов (общий для всех токенов одной сессии)
- `parent_token_id` - ID родительского токена (цепочка ротации)
- При детекции переиспользования - вся семья токенов отзывается
- Записывается audit log с типом `token_reuse_detected`

### user_sessions (Session Management)

```sql
CREATE TABLE user_sessions (
    id BIGSERIAL PRIMARY KEY,
    session_id UUID DEFAULT gen_random_uuid() UNIQUE,
    user_id BIGINT NOT NULL,
    refresh_token_id BIGINT,

    -- Device fingerprint
    device_id TEXT,
    device_name TEXT,
    device_type TEXT,
    browser TEXT,
    os TEXT,

    -- Network
    ip_address TEXT,
    location_country TEXT,
    location_city TEXT,

    -- Activity
    last_activity_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    activity_count BIGINT DEFAULT 1,

    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    terminated_at TIMESTAMPTZ,
    termination_reason TEXT,

    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT fk_user_sessions_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_user_sessions_refresh_token FOREIGN KEY (refresh_token_id)
        REFERENCES refresh_tokens(id)
);

-- Indexes
CREATE UNIQUE INDEX idx_user_sessions_session_id ON user_sessions(session_id);
CREATE INDEX idx_user_sessions_active ON user_sessions(user_id, is_active);
CREATE INDEX idx_user_sessions_last_activity ON user_sessions(last_activity_at DESC);
CREATE INDEX idx_user_sessions_session_id_active ON user_sessions(session_id) WHERE is_active;
```

**Использование:**
- Показывает список устройств пользователя (Web, Mobile, Desktop)
- Позволяет удаленно завершить сессию (logout с другого устройства)
- Трекинг активности и местоположения

### user_addresses (Address Book for Checkout)

```sql
CREATE TABLE user_addresses (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,

    -- Label
    label VARCHAR(100),                    -- "Home", "Work", "Dacha"

    -- Recipient
    recipient_name VARCHAR(255) NOT NULL,
    recipient_phone VARCHAR(50) NOT NULL,

    -- Address components
    country_code VARCHAR(3) NOT NULL DEFAULT 'RS',
    postal_code VARCHAR(20) NOT NULL,

    -- Multilingual street fields
    street_sr_latin VARCHAR(255) NOT NULL,  -- Required for delivery providers
    city_sr_latin VARCHAR(100) NOT NULL,
    district_sr_latin VARCHAR(100),

    street_en VARCHAR(255),
    city_en VARCHAR(100),
    district_en VARCHAR(100),

    street_user_locale VARCHAR(255),
    city_user_locale VARCHAR(100),
    district_user_locale VARCHAR(100),
    user_locale VARCHAR(10),

    -- Formatted addresses
    formatted_sr_latin VARCHAR(500),
    formatted_en VARCHAR(500),
    formatted_user_locale VARCHAR(500),

    -- Building details
    building VARCHAR(50),
    apartment VARCHAR(50),
    entrance VARCHAR(50),
    floor VARCHAR(20),
    intercom VARCHAR(50),
    notes TEXT,

    -- Geolocation
    latitude NUMERIC(10, 8),
    longitude NUMERIC(11, 8),

    -- Validation
    is_validated BOOLEAN DEFAULT FALSE,
    validation_source VARCHAR(50),
    confidence_score NUMERIC(3, 2),
    validated_at TIMESTAMPTZ,

    -- Post Express (Serbian delivery)
    post_express_settlement_id INTEGER,
    post_express_street_id INTEGER,

    -- Flags
    is_default BOOLEAN DEFAULT FALSE,
    is_last_used BOOLEAN DEFAULT FALSE,

    -- Soft delete
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMPTZ,

    CONSTRAINT fk_user_addresses_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE,

    -- Constraints
    CONSTRAINT chk_country_code CHECK (country_code ~ '^[A-Z]{2,3}$'),
    CONSTRAINT chk_postal_code CHECK (postal_code ~ '^\d{5}$'),
    CONSTRAINT chk_latitude CHECK (latitude IS NULL OR (latitude >= -90 AND latitude <= 90)),
    CONSTRAINT chk_longitude CHECK (longitude IS NULL OR (longitude >= -180 AND longitude <= 180)),
    CONSTRAINT chk_confidence_score CHECK (confidence_score IS NULL OR (confidence_score >= 0 AND confidence_score <= 1))
);

-- Indexes
CREATE INDEX idx_user_addresses_user_id ON user_addresses(user_id);
CREATE INDEX idx_user_addresses_default ON user_addresses(user_id, is_default) WHERE is_default = TRUE;
CREATE INDEX idx_user_addresses_last_used ON user_addresses(user_id, is_last_used) WHERE is_last_used = TRUE;
CREATE INDEX idx_user_addresses_country ON user_addresses(country_code);
CREATE INDEX idx_user_addresses_postal ON user_addresses(postal_code);
CREATE INDEX idx_user_addresses_city ON user_addresses(city_sr_latin);
CREATE INDEX idx_user_addresses_validation ON user_addresses(is_validated);
CREATE INDEX idx_user_addresses_active ON user_addresses(user_id, is_deleted) WHERE is_deleted = FALSE;

-- Triggers
CREATE TRIGGER update_user_addresses_updated_at
    BEFORE UPDATE ON user_addresses
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trigger_check_max_user_addresses
    BEFORE INSERT ON user_addresses
    FOR EACH ROW
    EXECUTE FUNCTION check_max_user_addresses(); -- Limit: 10 addresses per user

CREATE TRIGGER trigger_ensure_single_default_address
    BEFORE INSERT OR UPDATE ON user_addresses
    FOR EACH ROW
    EXECUTE FUNCTION ensure_single_default_address();

CREATE TRIGGER trigger_ensure_single_last_used_address
    BEFORE INSERT OR UPDATE ON user_addresses
    FOR EACH ROW
    EXECUTE FUNCTION ensure_single_last_used_address();
```

**Мультиязычность:**
- `sr_latin` - сербский латиницей (ОБЯЗАТЕЛЬНО для провайдеров доставки)
- `en` - английский (опционально)
- `user_locale` - язык ввода пользователя (sr, ru, en)

**Ограничения:**
- Максимум 10 адресов на пользователя
- Только один default адрес
- Только один last_used адрес

### service_accounts (Service-to-Service Authentication)

```sql
CREATE TABLE service_accounts (
    id SERIAL PRIMARY KEY,
    service_name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,

    -- API Key (stored hashed)
    api_key VARCHAR(255) UNIQUE NOT NULL,
    api_key_hash VARCHAR(255) NOT NULL,

    -- Link to virtual user (provider='service')
    user_id INT NOT NULL,

    -- Status
    is_active BOOLEAN DEFAULT TRUE,

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    last_used_at TIMESTAMP,

    CONSTRAINT fk_service_account_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE
);

-- Indexes
CREATE INDEX idx_service_accounts_api_key ON service_accounts(api_key);
CREATE INDEX idx_service_accounts_user_id ON service_accounts(user_id);
CREATE INDEX idx_service_accounts_service_name ON service_accounts(service_name);
```

**Workflow:**
1. Сервис регистрируется через API с уникальным `service_name`
2. Создается виртуальный user с `provider=service`, `roles=['service']`
3. Генерируется API key (хранится hashed как bcrypt)
4. Сервис обменивает API key на JWT токен (TTL 1 час)
5. JWT используется для всех S2S вызовов

**Примеры service_name:**
- `monolith` - основной монолит Vondi
- `listings-service` - микросервис объявлений
- `delivery-service` - микросервис доставки
- `warehouse-service` - WMS
- `payment-service` - платежный сервис

### audit_log (Security Audit Trail)

```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT,

    -- Action
    action TEXT NOT NULL,           -- login, logout, token_reuse_detected, etc
    resource TEXT,                  -- refresh_token, user, etc
    resource_id TEXT,

    -- Context
    ip_address TEXT,
    user_agent TEXT,
    request_id BYTEA,

    -- Changes
    old_values JSONB,
    new_values JSONB,
    metadata JSONB,

    -- Result
    success BOOLEAN DEFAULT TRUE,
    error_message TEXT,

    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_action ON audit_log(action, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource, resource_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

**Типы действий (action):**
- `user_login`, `user_logout`, `user_register`
- `password_change`, `password_reset_request`, `password_reset_complete`
- `2fa_enabled`, `2fa_disabled`, `2fa_verify_success`, `2fa_verify_fail`
- `token_refresh`, `token_reuse_detected`, `token_revoked`
- `email_verification_sent`, `email_verified`
- `role_added`, `role_removed`
- `profile_updated`, `account_deleted`

### revoked_access_tokens (Access Token Blacklist)

```sql
CREATE TABLE revoked_access_tokens (
    jti TEXT PRIMARY KEY,
    user_id BIGINT,
    expires_at TIMESTAMPTZ NOT NULL,
    revoked_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    revoked_reason TEXT
);

-- Indexes
CREATE INDEX idx_revoked_access_tokens_expires_at ON revoked_access_tokens(expires_at);
CREATE INDEX idx_revoked_access_tokens_user_id ON revoked_access_tokens(user_id);
```

**Причины отзыва (revoked_reason):**
- `user_logout` - пользователь вышел
- `admin_revoke` - принудительный выход администратором
- `password_change` - смена пароля
- `security_breach` - подозрительная активность
- `token_reuse_detected` - детекция переиспользования refresh токена

### user_checkout_profiles (Checkout Contact Profiles)

```sql
CREATE TABLE user_checkout_profiles (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,  -- One profile per user
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255),
    phone VARCHAR(50),
    preferred_address_id BIGINT,
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_checkout_profiles_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_checkout_profiles_address FOREIGN KEY (preferred_address_id)
        REFERENCES user_addresses(id) ON DELETE SET NULL
);

-- Indexes
CREATE INDEX idx_checkout_profiles_user_id ON user_checkout_profiles(user_id);
CREATE INDEX idx_checkout_profiles_email ON user_checkout_profiles(email) WHERE email IS NOT NULL;
CREATE INDEX idx_checkout_profiles_phone ON user_checkout_profiles(phone) WHERE phone IS NOT NULL;
CREATE INDEX idx_checkout_profiles_address ON user_checkout_profiles(preferred_address_id) WHERE preferred_address_id IS NOT NULL;

-- Trigger
CREATE TRIGGER update_checkout_profiles_updated_at
    BEFORE UPDATE ON user_checkout_profiles
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

**Зачем отдельная таблица:**
- Email для уведомлений может отличаться от email аутентификации
- Телефон для курьера может быть другим
- Имя получателя может отличаться от имени пользователя

## Database Functions

### update_updated_at_column()

```sql
CREATE FUNCTION update_updated_at_column() RETURNS TRIGGER
    LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$;
```

**Использование:** Автоматически обновляет `updated_at` при UPDATE строки.

### detect_token_reuse()

```sql
CREATE FUNCTION detect_token_reuse() RETURNS TRIGGER
    LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.use_count > 1 THEN
        -- Revoke entire token family
        UPDATE refresh_tokens
        SET is_revoked = TRUE,
            revoked_at = CURRENT_TIMESTAMP,
            revoked_reason = 'reuse_detected'
        WHERE family_id = NEW.family_id;

        -- Log security event
        INSERT INTO audit_log (user_id, action, resource, resource_id, metadata, created_at)
        VALUES (
            NEW.user_id,
            'token_reuse_detected',
            'refresh_token',
            NEW.jti::text,
            jsonb_build_object(
                'family_id', NEW.family_id,
                'ip_address', NEW.ip_address,
                'user_agent', NEW.user_agent
            ),
            CURRENT_TIMESTAMP
        );
    END IF;

    RETURN NEW;
END;
$$;
```

**Назначение:** Детекция повторного использования refresh токенов (security breach).

### check_max_user_addresses()

```sql
CREATE FUNCTION check_max_user_addresses() RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT COUNT(*) FROM user_addresses WHERE user_id = NEW.user_id AND is_deleted = FALSE) >= 10 THEN
        RAISE EXCEPTION 'addresses.limit_exceeded' USING ERRCODE = 'P0001';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Назначение:** Лимит 10 адресов на пользователя.

### ensure_single_default_address()

```sql
CREATE FUNCTION ensure_single_default_address() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.is_default = TRUE THEN
        UPDATE user_addresses
        SET is_default = FALSE
        WHERE user_id = NEW.user_id
            AND id != NEW.id
            AND is_default = TRUE;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Назначение:** Только один default адрес на пользователя.

### ensure_single_last_used_address()

```sql
CREATE FUNCTION ensure_single_last_used_address() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.is_last_used = TRUE THEN
        UPDATE user_addresses
        SET is_last_used = FALSE
        WHERE user_id = NEW.user_id
            AND id != NEW.id
            AND is_last_used = TRUE;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Назначение:** Только один last_used адрес на пользователя.

## Полезные запросы

```sql
-- Активные пользователи
SELECT id, email, name, roles, last_login_at
FROM users
WHERE is_active = true AND is_deleted = false
ORDER BY last_login_at DESC NULLS LAST;

-- Пользователи с 2FA
SELECT id, email, name, two_factor_enabled
FROM users
WHERE two_factor_enabled = true;

-- Администраторы
SELECT id, email, name, roles
FROM users
WHERE 'admin' = ANY(roles)
ORDER BY created_at;

-- Активные refresh токены пользователя
SELECT rt.id, rt.jti, rt.device_name, rt.ip_address, rt.created_at, rt.expires_at
FROM refresh_tokens rt
WHERE rt.user_id = 123
  AND rt.is_revoked = false
  AND rt.expires_at > NOW()
ORDER BY rt.created_at DESC;

-- Активные сессии пользователя
SELECT session_id, device_name, browser, os, ip_address,
       last_activity_at, location_city, location_country
FROM user_sessions
WHERE user_id = 123
  AND is_active = true
ORDER BY last_activity_at DESC;

-- Адреса пользователя
SELECT id, label, recipient_name,
       street_sr_latin, city_sr_latin, postal_code,
       is_default, is_last_used
FROM user_addresses
WHERE user_id = 123
  AND is_deleted = false
ORDER BY
    is_default DESC,
    is_last_used DESC,
    created_at DESC;

-- Детекция подозрительной активности (токены с повторным использованием)
SELECT al.created_at, al.user_id, u.email,
       al.metadata->>'family_id' as family_id,
       al.metadata->>'ip_address' as ip_address
FROM audit_log al
JOIN users u ON u.id = al.user_id
WHERE al.action = 'token_reuse_detected'
ORDER BY al.created_at DESC
LIMIT 20;

-- Service accounts
SELECT sa.service_name, sa.is_active, sa.last_used_at,
       u.email as service_email, u.roles
FROM service_accounts sa
JOIN users u ON u.id = sa.user_id
ORDER BY sa.last_used_at DESC NULLS LAST;

-- Статистика по провайдерам
SELECT provider, COUNT(*) as count
FROM users
WHERE is_deleted = false
GROUP BY provider
ORDER BY count DESC;

-- Неактивные пользователи (не логинились >30 дней)
SELECT id, email, name, last_login_at
FROM users
WHERE is_active = true
  AND is_deleted = false
  AND (last_login_at < NOW() - INTERVAL '30 days' OR last_login_at IS NULL)
ORDER BY last_login_at DESC NULLS LAST;

-- Истекшие refresh токены для cleanup
SELECT COUNT(*) as expired_count
FROM refresh_tokens
WHERE expires_at < NOW()
  AND is_revoked = false;

-- Checkout профили с адресами
SELECT cp.id, cp.first_name, cp.last_name, cp.email, cp.phone,
       ua.label, ua.street_sr_latin, ua.city_sr_latin
FROM user_checkout_profiles cp
LEFT JOIN user_addresses ua ON ua.id = cp.preferred_address_id
WHERE cp.user_id = 123;

-- Аудит действий пользователя за последние 24 часа
SELECT created_at, action, resource,
       metadata, success, error_message
FROM audit_log
WHERE user_id = 123
  AND created_at > NOW() - INTERVAL '24 hours'
ORDER BY created_at DESC;

-- История токенов пользователя (token family)
SELECT jti, family_id, parent_token_id,
       device_name, is_revoked, revoked_reason,
       created_at, expires_at
FROM refresh_tokens
WHERE user_id = 123
ORDER BY created_at DESC;

-- Провайдеры аутентификации (статистика)
SELECT provider,
       COUNT(*) as users_count,
       COUNT(CASE WHEN last_login_at > NOW() - INTERVAL '30 days' THEN 1 END) as active_30d
FROM users
WHERE is_deleted = FALSE
GROUP BY provider
ORDER BY users_count DESC;
```

## Связь с другими сервисами

### Интеграция с монолитом (vondi)
- `users` таблица синхронизируется в монолит (read-only cache)
- Auth Service - source of truth для пользовательских данных
- Монолит использует pkg библиотеку для аутентификации

### Интеграция с Listings Service
- `users` таблица синхронизируется в Listings (read-only cache)
- `listings.user_id` - FK к Auth users
- JWT токены валидируются через Auth Service

### Интеграция с Delivery Service
- Использует `user_addresses` для адресов доставки
- Integration через pkg библиотеку

### Service-to-Service Authentication
- `service_accounts` таблица для API ключей микросервисов
- Каждый сервис получает JWT токен с ролью 'service'
- Пример: tasktracker, payment-service, monolith

## Миграции

### Порядок выполнения

| # | Миграция | Описание | Дата |
|---|----------|----------|------|
| 0001 | `functions.up.sql` | Utility функции (update_updated_at) | 2024-11 |
| 0002 | `users.up.sql` | Таблица users + индексы | 2024-11 |
| 0003 | `refresh_tokens.up.sql` | Refresh tokens + audit_log + detect_token_reuse | 2024-11 |
| 0004 | `user_sessions.up.sql` | User sessions для трекинга устройств | 2024-11 |
| 0005 | `revoked_access_tokens.up.sql` | Blacklist для access токенов | 2024-11 |
| 0006 | `user_profile_extended.up.sql` | Расширение users (bio, timezone, city, country) | 2024-11 |
| 0007 | `create_service_accounts.up.sql` | Service-to-service аккаунты | 2024-11 |
| 0008 | `user_addresses.up.sql` | Адресная книга для Unified Checkout | 2025-11 |
| 0009 | `user_checkout_profiles.up.sql` | Профили для чекаута | 2025-11 |

### Применение миграций

```bash
cd /p/github.com/vondi-global/auth

# Применить все миграции
./bin/migrator up

# Откатить последнюю миграцию
./bin/migrator down

# Откатить все миграции
./bin/migrator reset

# Статус миграций
./bin/migrator version
```

## Мониторинг и диагностика

```sql
-- Размер базы данных
SELECT pg_size_pretty(pg_database_size('auth_dev_db'));

-- Размер таблиц
SELECT tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Количество подключений
SELECT COUNT(*) FROM pg_stat_activity WHERE datname = 'auth_dev_db';

-- Активные запросы
SELECT pid, usename, state, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE datname = 'auth_dev_db' AND state != 'idle'
ORDER BY duration DESC;

-- Vacuum status
SELECT schemaname, tablename, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
ORDER BY last_autovacuum DESC NULLS LAST;

-- Index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

## Cleanup Tasks

```sql
-- Удалить истекшие refresh токены (рекомендуется запускать периодически)
DELETE FROM refresh_tokens
WHERE expires_at < NOW() - INTERVAL '7 days'
  AND is_revoked = true;

-- Удалить истекшие revoked access tokens
DELETE FROM revoked_access_tokens
WHERE expires_at < NOW() - INTERVAL '1 day';

-- Удалить старые audit log записи (>90 дней)
DELETE FROM audit_log
WHERE created_at < NOW() - INTERVAL '90 days';

-- Удалить неактивные сессии
DELETE FROM user_sessions
WHERE expires_at < NOW()
  OR (is_active = false AND terminated_at < NOW() - INTERVAL '30 days');
```

## Безопасность

### Token Security

**Refresh Token:**
- Хранится как bcrypt hash
- Family механизм для детекции переиспользования
- Автоматическая ротация при refresh
- TTL: 30 дней (настраиваемый)

**Access Token (JWT):**
- Подписывается RSA-2048 ключами
- Содержит: user_id, email, roles, uuid
- TTL: 15 минут (короткий для безопасности)
- Blacklist в `revoked_access_tokens` для принудительного отзыва

**Service Token (S2S):**
- API key храниться как bcrypt hash
- Обменивается на JWT с TTL 1 час
- JWT содержит: user_id (виртуального user), service_name, roles=['service']
- Автоматическое обновление через `ServiceTokenManager`

### Password Security

- Bcrypt hashing (cost factor 10)
- Минимальная длина: 8 символов
- Rate limiting на login endpoint
- Account locking после 5 неудачных попыток (locked_until)
- Audit log для всех попыток входа

### 2FA (Two-Factor Authentication)

- TOTP (Time-based One-Time Password)
- Секрет хранится в `two_factor_secret`
- Backup codes в массиве `backup_codes[]`
- Audit log при включении/отключении

## Environment Variables

```bash
# В .env файле auth service
VONDIAUTH_DB_HOST=localhost
VONDIAUTH_DB_PORT=25432               # НЕ 5433! НЕ 35434!
VONDIAUTH_DB_NAME=auth_dev_db         # НЕ vondi_db! НЕ listings_dev_db!
VONDIAUTH_DB_USER=auth_user           # НЕ postgres!
VONDIAUTH_DB_PASSWORD=auth_pass
VONDIAUTH_DB_SSLMODE=disable

# Redis (для rate limiting, caching)
VONDIAUTH_REDIS_HOST=localhost
VONDIAUTH_REDIS_PORT=26379

# Server
VONDIAUTH_SERVER_HTTP_PORT=28086
VONDIAUTH_SERVER_GRPC_PORT=20053

# JWT Keys
VONDIAUTH_JWT_PRIVATE_KEY_PATH=/opt/vondi-dev/keys/private.pem
VONDIAUTH_JWT_PUBLIC_KEY_PATH=/opt/vondi-dev/keys/public.pem
```

## Документация

- **CLAUDE.md:** `/p/github.com/vondi-global/auth/CLAUDE.md`
- **Миграции:** `/p/github.com/vondi-global/auth/migrations/`
- **Architecture:** `/p/github.com/vondi-global/auth/docs/ARCHITECTURE.md`
- **Integration Guide:** `/p/github.com/vondi-global/auth/docs/MARKETPLACE_INTEGRATION_SPEC.md`
- **PKG Library:** `/p/github.com/vondi-global/auth/pkg/README.md`

---

**Дата создания паспорта:** 2025-12-21
**Автор:** Claude Code
**Версия:** 1.0.0
