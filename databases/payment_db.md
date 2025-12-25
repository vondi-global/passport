# Паспорт: База данных payment_db

> Обновлено: 2025-12-21

## Идентификация

| Параметр | Значение |
|----------|----------|
| Тип | PostgreSQL 16 |
| Image | postgres:16 |
| Хост | localhost |
| Порт | 35433 |
| Имя БД | payment_db |
| Пользователь | payment_user |
| Пароль | payment_secret |
| Контейнер | vondipayment_postgres |

## КРИТИЧНО

**Это ОТДЕЛЬНАЯ база данных Payment микросервиса!**
- НЕ используй порт 5433 (это монолит vondi_db)
- НЕ используй порт 35432 (это Delivery микросервис)
- НЕ используй порт 35437 (это Notifications микросервис)
- Payment Service имеет СВОЮ независимую БД для платежей и транзакций
- **Расширения:** uuid-ossp (для UUID generation)

## Подключение

```bash
# Через localhost
psql "postgres://payment_user:payment_secret@localhost:35433/payment_db?sslmode=disable"

# Через Docker
docker exec -it vondipayment_postgres psql -U payment_user -d payment_db
```

## Статистика

| Параметр | Значение |
|----------|----------|
| Всего таблиц | 12+ |
| Бизнес-таблиц | 10 |
| Миграций | 12 |
| Путь к миграциям | /p/github.com/vondi-global/payment/migrations/ |

## Таблицы (полный список)

### 1. Платежи (Core Payments)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| payments | Все платежи (основная таблица) | UUID |
| refunds | Возвраты средств | UUID |

**Особенности payments:**
- Типы: platform_service (услуги платформы), product_b2c (B2C покупки), product_c2c (C2C покупки), balance_deposit (пополнение баланса)
- Статусы: pending, processing, succeeded, failed, cancelled, refunded
- Интеграции: `gateway_name` (stripe, allsecure), `gateway_payment_intent_id`
- Escrow: `escrow_enabled`, `escrow_hold_period_days`
- Связь: `user_id`, `merchant_id`, `buyer_id` (UUID из Auth MS)
- Metadata: JSONB для дополнительных данных

**Особенности refunds:**
- Связь: `payment_id` (FK к payments)
- Статусы: pending, succeeded, failed
- Интеграция: `gateway_refund_id` (ID в Stripe)
- Идемпотентность: `idempotency_key` UNIQUE

### 2. Escrow (Безопасные сделки)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| escrow_holds | Заморозка средств | UUID |

**Особенности escrow_holds:**
- Связь: `payment_id` (FK к payments)
- Статусы: holding (заморожено), released (выпущено), cancelled (отменено)
- Сроки: `hold_until` (дата автоматического release)
- Аудит: `released_at`, `cancelled_at`, `cancellation_reason`

### 3. Connected Accounts (Stripe Connect)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| connected_accounts | Подключенные аккаунты продавцов | UUID |
| balance_snapshots | Снапшоты балансов аккаунтов | UUID |

**Особенности connected_accounts:**
- Stripe ID: `stripe_account_id` (acct_xxx) UNIQUE
- Типы: express, standard, custom
- Статусы: pending, active, restricted, disabled
- Флаги: `charges_enabled`, `payouts_enabled`, `details_submitted`
- Бизнес: `business_type` (individual, company), `country` (ISO код)

**Особенности balance_snapshots:**
- Связь: `connected_account_id` (FK к connected_accounts)
- Балансы: `available_amount`, `pending_amount`, `instant_available_amount`
- Типы: scheduled (по расписанию), on_demand (по запросу), webhook (из Stripe)
- Raw data: `raw_response` JSONB

### 4. Transfers & Payouts

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| transfers | Трансферы на connected accounts | UUID |
| payouts | Выводы средств | UUID |

**Особенности transfers:**
- Stripe ID: `stripe_transfer_id` (tr_xxx) UNIQUE
- Связь: `connected_account_id`, `payment_id` (опционально)
- Статусы: pending, paid, failed, reversed, partially_reversed
- Реверс: `reversed_amount` для частичных возвратов

**Особенности payouts:**
- Stripe ID: `stripe_payout_id` (po_xxx) UNIQUE (nullable до подтверждения)
- Методы: standard, instant
- Destination: `destination_type` (bank_account, card), `destination_id`
- Статусы: pending, in_transit, paid, failed, canceled
- Ошибки: `failure_code`, `failure_message`

### 5. Split Rules (Разделение платежей)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| split_rules | Правила разделения комиссий | UUID |

**Особенности split_rules:**
- Связь: `connected_account_id` (FK к connected_accounts)
- Типы: percentage (процент), fixed (фиксированная сумма)
- Значение: `value` (10.5 для %, 5.00 для USD)
- Приоритет: `priority` (выше = применяется первым)
- Фильтры: `min_amount`, `max_amount` для условного применения

### 6. Системные Таблицы

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| schema_migrations | Версии миграций (goose) | - |

## Схемы ключевых таблиц

### payments (Платежи)

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'RSD',
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    payment_type VARCHAR(30) NOT NULL,
    gateway_name VARCHAR(20) NOT NULL,
    gateway_session_id VARCHAR(255),
    gateway_payment_intent_id VARCHAR(255),
    merchant_id UUID,
    buyer_id UUID,
    escrow_enabled BOOLEAN NOT NULL DEFAULT false,
    escrow_hold_period_days INT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_payment_status CHECK (status IN (
        'pending', 'processing', 'succeeded', 'failed', 'cancelled', 'refunded'
    )),
    CONSTRAINT chk_payment_type CHECK (payment_type IN (
        'platform_service', 'product_b2c', 'product_c2c', 'balance_deposit'
    )),
    CONSTRAINT chk_amount_positive CHECK (amount > 0)
);

-- Indexes
CREATE INDEX idx_payments_user_id ON payments(user_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_merchant_id ON payments(merchant_id) WHERE merchant_id IS NOT NULL;
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);
```

### connected_accounts (Stripe Connect)

```sql
CREATE TABLE connected_accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    stripe_account_id VARCHAR(255) NOT NULL UNIQUE,
    account_type VARCHAR(20) NOT NULL DEFAULT 'express',
    email VARCHAR(255) NOT NULL,
    business_type VARCHAR(20) NOT NULL DEFAULT 'individual',
    charges_enabled BOOLEAN NOT NULL DEFAULT false,
    payouts_enabled BOOLEAN NOT NULL DEFAULT false,
    details_submitted BOOLEAN NOT NULL DEFAULT false,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    country VARCHAR(2) NOT NULL DEFAULT 'RS',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_account_type CHECK (account_type IN ('express', 'standard', 'custom')),
    CONSTRAINT chk_account_status CHECK (status IN ('pending', 'active', 'restricted', 'disabled'))
);

-- Indexes
CREATE INDEX idx_connected_accounts_email ON connected_accounts(email);
CREATE INDEX idx_connected_accounts_status ON connected_accounts(status);
```

### escrow_holds (Эскроу)

```sql
CREATE TABLE escrow_holds (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    payment_id UUID NOT NULL REFERENCES payments(id) ON DELETE RESTRICT,
    amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'RSD',
    hold_until TIMESTAMPTZ NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'holding',
    released_at TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ,
    cancellation_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_escrow_status CHECK (status IN ('holding', 'released', 'cancelled')),
    CONSTRAINT chk_escrow_amount_positive CHECK (amount > 0)
);

-- Indexes
CREATE INDEX idx_escrow_holds_payment_id ON escrow_holds(payment_id);
CREATE INDEX idx_escrow_holds_status ON escrow_holds(status);
CREATE INDEX idx_escrow_holds_hold_until ON escrow_holds(hold_until) WHERE status = 'holding';
```

## Полезные запросы

```sql
-- Успешные платежи за сегодня
SELECT
    id, user_id, amount, currency,
    payment_type, gateway_name,
    created_at
FROM payments
WHERE status = 'succeeded'
  AND created_at >= CURRENT_DATE
ORDER BY created_at DESC;

-- Активные эскроу (ожидают release)
SELECT
    e.id, e.amount, e.currency,
    e.hold_until,
    p.user_id, p.merchant_id,
    e.created_at
FROM escrow_holds e
JOIN payments p ON p.id = e.payment_id
WHERE e.status = 'holding'
ORDER BY e.hold_until;

-- Connected accounts статистика
SELECT
    status,
    COUNT(*) as count,
    SUM(CASE WHEN charges_enabled THEN 1 ELSE 0 END) as charges_enabled,
    SUM(CASE WHEN payouts_enabled THEN 1 ELSE 0 END) as payouts_enabled
FROM connected_accounts
GROUP BY status;

-- Pending refunds
SELECT
    r.id, r.amount, r.currency,
    r.status, r.reason,
    p.user_id, p.gateway_name,
    r.created_at
FROM refunds r
JOIN payments p ON p.id = r.payment_id
WHERE r.status = 'pending'
ORDER BY r.created_at;

-- Transfers to connected accounts
SELECT
    t.id, t.amount, t.currency,
    t.status,
    ca.stripe_account_id, ca.email,
    t.created_at
FROM transfers t
JOIN connected_accounts ca ON ca.id = t.connected_account_id
WHERE t.status = 'pending'
ORDER BY t.created_at;

-- Payout failures
SELECT
    p.id, p.amount, p.currency,
    p.failure_code, p.failure_message,
    ca.email, ca.stripe_account_id,
    p.failed_at
FROM payouts p
JOIN connected_accounts ca ON ca.id = p.connected_account_id
WHERE p.status = 'failed'
ORDER BY p.failed_at DESC;

-- Split rules по аккаунтам
SELECT
    ca.email,
    sr.name, sr.type, sr.value, sr.priority,
    sr.is_active
FROM split_rules sr
JOIN connected_accounts ca ON ca.id = sr.connected_account_id
WHERE sr.is_active = true
ORDER BY ca.email, sr.priority DESC;
```

## Связь с другими сервисами

### Интеграция с монолитом (vondi)
- `payments.user_id` - FK к users (без constraint - cross-DB)
- `payments.merchant_id`, `payments.buyer_id` - FK к users (без constraint)

### Интеграция с Auth Service
- gRPC запросы для валидации user_id
- Проверка прав (кто может создавать connected account)

### Интеграция с WMS
- При returns: WMS вызывает RefundService для возврата средств
- При QC failed: автоматический refund

### Интеграция со Stripe
- Webhooks: payment_intent.succeeded, payout.paid, etc.
- Connect: onboarding, account updates

### События (Redis Streams)
**Публикует:**
- `payment.succeeded` - платёж успешен
- `payment.failed` - платёж неудачен
- `escrow.released` - эскроу выпущен
- `payout.completed` - вывод завершён

**Подписывается:**
- `orders.completed` - для release escrow
- `returns.approved` - для создания refund

## Environment Variables

```bash
# В .env файле payment микросервиса
VONDIPAYMENT_DB_HOST=localhost
VONDIPAYMENT_DB_PORT=35433              # НЕ 5433! НЕ 35432!
VONDIPAYMENT_DB_NAME=payment_db         # НЕ vondi_db!
VONDIPAYMENT_DB_USER=payment_user       # НЕ postgres!
VONDIPAYMENT_DB_PASSWORD=payment_secret
VONDIPAYMENT_DB_SSL_MODE=disable

# gRPC
VONDIPAYMENT_SERVER_GRPC_PORT=30051

# HTTP (Metrics)
VONDIPAYMENT_METRICS_PORT=39090

# Stripe
VONDIPAYMENT_STRIPE_SECRET_KEY=sk_live_...
VONDIPAYMENT_STRIPE_WEBHOOK_SECRET=whsec_...
```

## Документация

- **Service Passport:** `/p/github.com/vondi-global/passport/services/payment.md`
- **Миграции:** `/p/github.com/vondi-global/payment/migrations/`
- **Domain models:** `/p/github.com/vondi-global/payment/internal/domain/entity/`
- **gRPC API:** `/p/github.com/vondi-global/payment/api/proto/`

---

**Дата создания паспорта:** 2025-12-21
**Автор:** Claude Code
**Версия:** 1.0.0
