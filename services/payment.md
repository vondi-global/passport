# Payment Service Passport

## 1. Обзор

**Назначение:** Микросервис обработки платежей с поддержкой escrow, multi-currency wallets, split payments и автоматических выплат (payouts).

**Архитектура:** DDD (Domain-Driven Design)
**Язык:** Go 1.23
**Репозиторий:** `github.com/vondi-global/payment`
**Директория:** `/p/github.com/vondi-global/payment`

### Основные функции

- 💳 **Payment Processing** — создание checkout сессий через Stripe/AllSecure
- 🔒 **Escrow System** — холдирование средств с автоматическим release
- 💰 **Multi-Wallet** — балансы connected accounts (RSD, EUR, USD)
- 🔄 **Transfers** — переводы между платформой и вендорами
- 📊 **Fee Calculation** — гибкие правила комиссий (процент/фиксированная)
- 💸 **Payouts** — выплаты на банковские счета (standard/instant)
- ♻️ **Refunds** — полный/частичный возврат платежей
- 🔗 **Stripe Connect** — интеграция маркетплейса

---

## 2. Конфигурация

### Environment Variables (Prefix: `VONDIPAYMENT_`)

**Service:**
```bash
VONDIPAYMENT_SERVICE_NAME=payment-service
VONDIPAYMENT_ENV=development|production
VONDIPAYMENT_LOG_LEVEL=debug|info|warn|error
```

**Database (PostgreSQL):**
```bash
VONDIPAYMENT_DB_HOST=localhost
VONDIPAYMENT_DB_PORT=35433           # Нестандартный порт!
VONDIPAYMENT_DB_NAME=payment_db
VONDIPAYMENT_DB_USER=payment_user
VONDIPAYMENT_DB_PASSWORD=***
VONDIPAYMENT_DB_SSL_MODE=disable
VONDIPAYMENT_DB_MAX_CONNECTIONS=100
VONDIPAYMENT_DB_MAX_IDLE_CONNECTIONS=10
VONDIPAYMENT_DB_CONNECTION_MAX_LIFETIME=1h
VONDIPAYMENT_DB_MIGRATION_PATH=migrations
```

**Server:**
```bash
VONDIPAYMENT_SERVER_GRPC_PORT=30051          # gRPC API
VONDIPAYMENT_SERVER_METRICS_PORT=39090       # Prometheus метрики
VONDIPAYMENT_SERVER_READ_TIMEOUT=30s
VONDIPAYMENT_SERVER_WRITE_TIMEOUT=30s
VONDIPAYMENT_SERVER_IDLE_TIMEOUT=120s
VONDIPAYMENT_SERVER_SHUTDOWN_TIMEOUT=30s
```

**Payment Gateways:**
```bash
# Default gateway
VONDIPAYMENT_GATEWAYS_DEFAULT=stripe|allsecure
VONDIPAYMENT_GATEWAYS_USE_MOCK=false         # Mock для тестов

# Stripe
VONDIPAYMENT_GATEWAYS_STRIPE_ENABLED=true
VONDIPAYMENT_GATEWAYS_STRIPE_API_KEY=sk_test_***
VONDIPAYMENT_GATEWAYS_STRIPE_WEBHOOK_SECRET=whsec_***
VONDIPAYMENT_GATEWAYS_STRIPE_SUPPORTED_CURRENCIES=USD,EUR,RUB
VONDIPAYMENT_GATEWAYS_STRIPE_MIN_AMOUNT=50    # cents
VONDIPAYMENT_GATEWAYS_STRIPE_MAX_AMOUNT=99999999

# AllSecure (Serbian market)
VONDIPAYMENT_GATEWAYS_ALLSECURE_ENABLED=false
VONDIPAYMENT_GATEWAYS_ALLSECURE_BASE_URL=https://asxgw.paymentsandbox.cloud/api/v3
VONDIPAYMENT_GATEWAYS_ALLSECURE_API_KEY=***
VONDIPAYMENT_GATEWAYS_ALLSECURE_SHARED_SECRET=***
VONDIPAYMENT_GATEWAYS_ALLSECURE_USERNAME=***
VONDIPAYMENT_GATEWAYS_ALLSECURE_PASSWORD=***
VONDIPAYMENT_GATEWAYS_ALLSECURE_PUBLIC_INTEGRATION_KEY=***
VONDIPAYMENT_GATEWAYS_ALLSECURE_SUPPORTED_CURRENCIES=RSD,EUR
VONDIPAYMENT_GATEWAYS_ALLSECURE_MIN_AMOUNT=100
VONDIPAYMENT_GATEWAYS_ALLSECURE_MAX_AMOUNT=99999999
VONDIPAYMENT_GATEWAYS_ALLSECURE_SIGNATURE_ENABLED=true
```

**Auth (gRPC to Auth Service):**
```bash
VONDIPAYMENT_AUTH_ENABLED=true
VONDIPAYMENT_AUTH_TEST_MODE=false
VONDIPAYMENT_AUTH_GRPC_ADDRESS=localhost:20053
VONDIPAYMENT_AUTH_SERVICE_NAME=payment-service
VONDIPAYMENT_AUTH_SERVICE_API_KEY=***
# Методы без auth:
VONDIPAYMENT_AUTH_SKIP_METHODS=/payment.v1.PaymentService/HealthCheck
```

**Fixtures (для тестов):**
```bash
VONDIPAYMENT_FIXTURES_ENABLED=false
VONDIPAYMENT_FIXTURES_PATH=fixtures
```

### Порты (Локальные)

| Сервис | Порт | Назначение |
|--------|------|-----------|
| PostgreSQL | **35433** | База данных |
| gRPC | **30051** | API |
| Metrics | **39090** | Prometheus |

### Конфигурационные файлы

- `/p/github.com/vondi-global/payment/internal/config/config.go` — структуры конфигурации
- `/p/github.com/vondi-global/payment/.env.example` — шаблон переменных
- `/p/github.com/vondi-global/payment/docker-compose.yml` — PostgreSQL setup

---

## 3. Доменная модель

### 3.1 Core Entities

**Payment** (`internal/domain/entity/payment.go`)
```go
type Payment struct {
    ID                     uuid.UUID
    UserID                 uuid.UUID
    Amount                 decimal.Decimal  // Standard unit (USD, not cents)
    Currency               string
    Status                 PaymentStatus    // pending, processing, succeeded, failed, cancelled, refunded
    PaymentType            PaymentType      // platform_service, product_b2c, product_c2c, balance_deposit
    GatewayName            string           // stripe, allsecure
    GatewaySessionID       *string
    GatewayPaymentIntentID *string
    MerchantID             *uuid.UUID
    BuyerID                *uuid.UUID
    EscrowEnabled          bool
    EscrowHoldPeriodDays   *int
    Metadata               map[string]interface{}
    CreatedAt              time.Time
    UpdatedAt              time.Time
}
```

**EscrowHold** (`internal/domain/entity/escrow_hold.go`)
```go
type EscrowHold struct {
    ID                 uuid.UUID
    PaymentID          uuid.UUID
    Amount             decimal.Decimal
    Currency           string
    HoldUntil          time.Time
    Status             EscrowStatus    // holding, released, cancelled
    ReleasedAt         *time.Time
    CancelledAt        *time.Time
    CancellationReason *string
    CreatedAt          time.Time
    UpdatedAt          time.Time
}
```

**ConnectedAccount** (`internal/domain/entity/connected_account.go`)
```go
type ConnectedAccount struct {
    ID                uuid.UUID
    StripeAccountID   string           // acct_xxx
    AccountType       ConnectedAccountType  // express, standard, custom
    Email             string
    BusinessType      string           // individual, company
    ChargesEnabled    bool
    PayoutsEnabled    bool
    DetailsSubmitted  bool
    Status            ConnectedAccountStatus  // pending, active, restricted, disabled
    Country           string           // ISO code (US, RS)
    Metadata          map[string]interface{}
    CreatedAt         time.Time
    UpdatedAt         time.Time
}
```

**Transfer** (`internal/domain/entity/transfer.go`)
```go
type Transfer struct {
    ID                   uuid.UUID
    StripeTransferID     string       // tr_xxx
    ConnectedAccountID   uuid.UUID
    StripeAccountID      string       // acct_xxx
    PaymentID            *uuid.UUID   // Optional link to payment
    Amount               decimal.Decimal
    Currency             string
    Status               TransferStatus  // pending, paid, failed, reversed, partially_reversed
    Description          *string
    SourceTransaction    *string
    ReversedAmount       decimal.Decimal
    Metadata             map[string]interface{}
    CreatedAt            time.Time
    UpdatedAt            time.Time
    PaidAt               *time.Time
}
```

**BalanceSnapshot** (`internal/domain/entity/balance_snapshot.go`)
```go
type BalanceSnapshot struct {
    ID                        uuid.UUID
    ConnectedAccountID        uuid.UUID
    StripeAccountID           string
    AvailableAmount           decimal.Decimal
    AvailableCurrency         string
    PendingAmount             decimal.Decimal
    PendingCurrency           string
    InstantAvailableAmount    *decimal.Decimal
    InstantAvailableCurrency  *string
    SourceTypes               map[string]interface{}  // JSONB
    SnapshotType              string  // scheduled, on_demand, webhook
    RawResponse               map[string]interface{}  // JSONB
    CreatedAt                 time.Time
}
```

**Payout** (`internal/domain/entity/payout.go`)
```go
type Payout struct {
    ID                   uuid.UUID
    StripePayoutID       *string      // po_xxx (nullable until Stripe confirms)
    ConnectedAccountID   uuid.UUID
    StripeAccountID      string
    Amount               decimal.Decimal
    Currency             string
    Status               PayoutStatus   // pending, in_transit, paid, failed, canceled
    Method               PayoutMethod   // standard, instant
    DestinationType      *string        // bank_account, card
    DestinationID        *string        // ba_xxx, card_xxx
    ArrivalDate          *time.Time
    FailureCode          *string
    FailureMessage       *string
    Description          *string
    StatementDescriptor  *string        // Max 22 chars
    Metadata             map[string]interface{}
    IdempotencyKey       *string
    CreatedAt            time.Time
    UpdatedAt            time.Time
    PaidAt               *time.Time
    FailedAt             *time.Time
    CanceledAt           *time.Time
}
```

**SplitRule** (`internal/domain/entity/split_rule.go`)
```go
type SplitRule struct {
    ID                 uuid.UUID
    Name               string
    ConnectedAccountID uuid.UUID
    Type               SplitRuleType   // percentage, fixed
    Value              decimal.Decimal  // 10.5 (%), or 5.00 (USD)
    Priority           int              // Higher = applied first
    IsActive           bool
    Currency           *string          // Required for fixed type
    MinAmount          *decimal.Decimal // Optional filter
    MaxAmount          *decimal.Decimal // Optional filter
    CreatedAt          time.Time
    UpdatedAt          time.Time
}
```

**Refund** (`internal/domain/entity/refund.go`)
```go
type Refund struct {
    ID                uuid.UUID
    PaymentID         uuid.UUID
    Amount            decimal.Decimal
    Currency          string
    Status            RefundStatus  // pending, succeeded, failed
    GatewayRefundID   *string
    Reason            *string
    IdempotencyKey    *string
    CreatedAt         time.Time
    ProcessedAt       *time.Time
}
```

### 3.2 Enums

**PaymentStatus:** `pending`, `processing`, `succeeded`, `failed`, `cancelled`, `refunded`

**PaymentType:** `platform_service`, `product_b2c`, `product_c2c`, `balance_deposit`

**EscrowStatus:** `holding`, `released`, `cancelled`

**ConnectedAccountType:** `express`, `standard`, `custom`

**ConnectedAccountStatus:** `pending`, `active`, `restricted`, `disabled`

**TransferStatus:** `pending`, `paid`, `failed`, `reversed`, `partially_reversed`

**RefundStatus:** `pending`, `succeeded`, `failed`

**PayoutStatus:** `pending`, `in_transit`, `paid`, `failed`, `canceled`

**PayoutMethod:** `standard` (free, 2-3 days), `instant` (paid, ~30 min)

**SplitRuleType:** `percentage` (10%), `fixed` ($5.00)

---

## 4. gRPC API

**Proto:** `/p/github.com/vondi-global/payment/proto/payment/v1/payment.proto`

### 4.1 PaymentService

```protobuf
service PaymentService {
  rpc CreateCheckoutSession(CreateCheckoutSessionRequest) returns (CreateCheckoutSessionResponse);
  rpc GetPayment(GetPaymentRequest) returns (GetPaymentResponse);
  rpc GetUserPayments(GetUserPaymentsRequest) returns (GetUserPaymentsResponse);
  rpc ProcessWebhook(ProcessWebhookRequest) returns (ProcessWebhookResponse);
  rpc ProcessWebhookEvent(ProcessWebhookEventRequest) returns (ProcessWebhookEventResponse);
}
```

**CreateCheckoutSession** — создаёт checkout сессию:
- Параметры: `user_id`, `amount`, `currency`, `payment_type`, `success_url`, `cancel_url`, `escrow_enabled`, `escrow_hold_days`, `merchant_id`, `buyer_id`, `gateway_name`, `idempotency_key`
- Возвращает: `payment_id`, `session_id`, `checkout_url`, `status`

**GetPayment** — получить платёж по ID:
- Параметры: `payment_id` (UUID)
- Возвращает: `Payment` объект

**GetUserPayments** — список платежей пользователя:
- Параметры: `user_id`, `limit`, `offset`
- Возвращает: `payments[]`, `total`

**ProcessWebhook** — обработка webhook от gateway (deprecated, используй ProcessWebhookEvent):
- Параметры: `gateway_name`, `payload` (bytes), `signature`
- Возвращает: `success`, `message`

**ProcessWebhookEvent** — обработка webhook (новая версия):
- Параметры: `event_id`, `event_type`, `event_payload` (JSON), `created_at`, `gateway_name`
- Возвращает: `success`, `error_message`, `payment_id`

### 4.2 EscrowService

```protobuf
service EscrowService {
  rpc GetEscrowHold(GetEscrowHoldRequest) returns (GetEscrowHoldResponse);
  rpc GetEscrowByPaymentID(GetEscrowByPaymentIDRequest) returns (GetEscrowByPaymentIDResponse);
  rpc ListEscrows(ListEscrowsRequest) returns (ListEscrowsResponse);
  rpc ReleaseEscrowHold(ReleaseEscrowHoldRequest) returns (ReleaseEscrowHoldResponse);
  rpc CancelEscrowHold(CancelEscrowHoldRequest) returns (CancelEscrowHoldResponse);
  rpc ProcessExpiredEscrows(ProcessExpiredEscrowsRequest) returns (ProcessExpiredEscrowsResponse);
}
```

**ReleaseEscrowHold** — освободить холд вручную:
- Параметры: `escrow_id`
- Возвращает: `success`, `escrow`

**CancelEscrowHold** — отменить холд (например, при возврате товара):
- Параметры: `escrow_id`, `reason`
- Возвращает: `success`, `escrow`

**ProcessExpiredEscrows** — batch release истекших холдов (для cron):
- Параметры: `limit` (default 100)
- Возвращает: `released_count`

### 4.3 RefundService

```protobuf
service RefundService {
  rpc CreateRefund(CreateRefundRequest) returns (CreateRefundResponse);
  rpc GetRefund(GetRefundRequest) returns (GetRefundResponse);
  rpc GetPaymentRefunds(GetPaymentRefundsRequest) returns (GetPaymentRefundsResponse);
}
```

**CreateRefund** — создать возврат:
- Параметры: `payment_id`, `amount`, `reason`, `idempotency_key`
- Возвращает: `Refund` объект

### 4.4 ConnectedAccountService

```protobuf
service ConnectedAccountService {
  rpc CreateConnectedAccount(CreateConnectedAccountRequest) returns (CreateConnectedAccountResponse);
  rpc GetConnectedAccount(GetConnectedAccountRequest) returns (GetConnectedAccountResponse);
  rpc CreateAccountLink(CreateAccountLinkRequest) returns (CreateAccountLinkResponse);
  rpc ListConnectedAccounts(ListConnectedAccountsRequest) returns (ListConnectedAccountsResponse);
}
```

**CreateConnectedAccount** — создать Stripe Connect аккаунт:
- Параметры: `email`, `account_type`, `business_type`, `country`, `metadata`
- Возвращает: `ConnectedAccount` объект

**CreateAccountLink** — сгенерировать onboarding URL:
- Параметры: `account_id`, `return_url`, `refresh_url`
- Возвращает: `url`, `expires_at`

### 4.5 TransferService

```protobuf
service TransferService {
  rpc CreateTransfer(CreateTransferRequest) returns (CreateTransferResponse);
  rpc GetTransfer(GetTransferRequest) returns (GetTransferResponse);
  rpc ListTransfers(ListTransfersRequest) returns (ListTransfersResponse);
  rpc ReverseTransfer(ReverseTransferRequest) returns (ReverseTransferResponse);
}
```

**CreateTransfer** — перевести средства на connected account:
- Параметры: `connected_account_id`, `amount`, `currency`, `payment_id`, `description`, `metadata`, `idempotency_key`
- Возвращает: `Transfer` объект

**ReverseTransfer** — отменить/вернуть трансфер (полностью или частично):
- Параметры: `transfer_id`, `amount` (optional для полного), `description`, `idempotency_key`
- Возвращает: `TransferReversal`, `Transfer` (updated)

### 4.6 BalanceService

```protobuf
service BalanceService {
  rpc GetBalance(GetBalanceRequest) returns (GetBalanceResponse);
  rpc GetBalanceHistory(GetBalanceHistoryRequest) returns (GetBalanceHistoryResponse);
}
```

**GetBalance** — текущий баланс connected account:
- Параметры: `account_id`, `force_refresh` (true = fetch from Stripe, false = cached)
- Возвращает: `BalanceSnapshot` (available, pending, instant_available)

**GetBalanceHistory** — история балансов:
- Параметры: `account_id`, `from_date`, `to_date`, `limit`, `offset`
- Возвращает: `snapshots[]`, `total`

### 4.7 PayoutService

```protobuf
service PayoutService {
  rpc CreatePayout(CreatePayoutRequest) returns (CreatePayoutResponse);
  rpc GetPayout(GetPayoutRequest) returns (GetPayoutResponse);
  rpc ListPayouts(ListPayoutsRequest) returns (ListPayoutsResponse);
  rpc CancelPayout(CancelPayoutRequest) returns (CancelPayoutResponse);
}
```

**CreatePayout** — вывести средства на банковский счёт:
- Параметры: `connected_account_id`, `amount`, `currency`, `method` (standard/instant), `description`, `statement_descriptor`, `metadata`, `idempotency_key`
- Возвращает: `Payout` объект

**CancelPayout** — отменить pending payout:
- Параметры: `payout_id`
- Возвращает: `Payout` (updated)

### 4.8 SplitService

```protobuf
service SplitService {
  rpc CreateSplitRule(CreateSplitRuleRequest) returns (CreateSplitRuleResponse);
  rpc GetSplitRule(GetSplitRuleRequest) returns (GetSplitRuleResponse);
  rpc ListSplitRules(ListSplitRulesRequest) returns (ListSplitRulesResponse);
  rpc UpdateSplitRule(UpdateSplitRuleRequest) returns (UpdateSplitRuleResponse);
  rpc DeleteSplitRule(DeleteSplitRuleRequest) returns (DeleteSplitRuleResponse);
  rpc CalculateFee(CalculateFeeRequest) returns (CalculateFeeResponse);
}
```

**CreateSplitRule** — создать правило комиссии:
- Параметры: `name`, `connected_account_id`, `type` (percentage/fixed), `value`, `priority`, `currency`, `min_amount`, `max_amount`
- Возвращает: `SplitRule`

**CalculateFee** — рассчитать комиссию по правилам:
- Параметры: `connected_account_id`, `amount`, `currency`
- Возвращает: `total_fee`, `applied_rules[]`, `transfer_amount` (amount - fee)

---

## 5. Escrow Flow

### 5.1 Создание Escrow Hold

1. Клиент создаёт checkout с `escrow_enabled=true`, `escrow_hold_days=7`
2. Payment Service создаёт `Payment` с `EscrowEnabled=true`
3. При webhook `payment_intent.succeeded` → создаётся `EscrowHold`:
   - `HoldUntil = NOW() + 7 days`
   - `Status = holding`
4. Средства **холдятся на платформе**, не переводятся на вендора

### 5.2 Release Escrow

**Автоматический release:**
- Cron job вызывает `ProcessExpiredEscrows` (рекомендуется раз в час)
- Находит все `EscrowHold` где `HoldUntil < NOW()` и `Status = holding`
- Освобождает средства, устанавливает `Status = released`, `ReleasedAt = NOW()`
- Создаёт `Transfer` на connected account вендора

**Ручной release:**
- Вызов `ReleaseEscrowHold(escrow_id)` (например, после подтверждения доставки)
- Немедленно освобождает средства

### 5.3 Cancel Escrow

- Вызов `CancelEscrowHold(escrow_id, reason)`
- Устанавливает `Status = cancelled`, `CancelledAt = NOW()`
- **НЕ** создаёт transfer (средства остаются на платформе)
- Далее нужен **Refund** для возврата покупателю

### 5.4 Пример Flow (B2C заказ)

1. Покупатель оплачивает заказ → создаётся escrow hold на 7 дней
2. Продавец отправляет товар
3. Через 7 дней (или при подтверждении доставки) → escrow release
4. Средства переводятся продавцу минус комиссия (через SplitRule)

---

## 6. Multi-Wallet System (Connected Accounts)

### 6.1 Создание Connected Account

```go
CreateConnectedAccount(
    email: "vendor@example.com",
    account_type: CONNECTED_ACCOUNT_TYPE_EXPRESS,  // Быстрая регистрация
    business_type: "individual",
    country: "US",
    metadata: {"seller_id": "12345"}
)
```

Возвращает:
- `id` (internal UUID)
- `stripe_account_id` (acct_xxx)
- `status: pending` (ждёт onboarding)

### 6.2 Onboarding

```go
CreateAccountLink(
    account_id: "...",
    return_url: "https://vondi.rs/seller/onboarding/success",
    refresh_url: "https://vondi.rs/seller/onboarding/refresh"
)
```

Возвращает URL для Stripe onboarding:
- Вендор заполняет KYC данные
- После успеха → `charges_enabled=true`, `payouts_enabled=true`, `status=active`

### 6.3 Баланс Connected Account

**Структура баланса:**
- **Available** — доступны сейчас для payout
- **Pending** — будут доступны позже (например, через 2-3 дня)
- **Instant Available** — доступны для instant payout (опционально)

**Валюты:** Один account может иметь балансы в нескольких валютах (USD, EUR, RSD)

**Получение баланса:**
```go
GetBalance(account_id, force_refresh=true)
// Возвращает BalanceSnapshot с текущими available/pending amounts
```

### 6.4 История балансов

```go
GetBalanceHistory(
    account_id: "...",
    from_date: "2024-01-01T00:00:00Z",
    to_date: "2024-12-31T23:59:59Z"
)
// Возвращает массив BalanceSnapshot (для графиков в UI)
```

---

## 7. Fee Calculation (Split Rules)

### 7.1 Типы правил

**Percentage:**
```go
CreateSplitRule(
    name: "Platform fee 10%",
    type: SPLIT_RULE_TYPE_PERCENTAGE,
    value: 10.0,  // 10%
    priority: 10
)
```

**Fixed:**
```go
CreateSplitRule(
    name: "Fixed fee $5",
    type: SPLIT_RULE_TYPE_FIXED,
    value: 5.00,
    currency: "USD",
    priority: 5
)
```

### 7.2 Фильтры (Min/Max Amount)

```go
CreateSplitRule(
    name: "Large order fee 5%",
    type: SPLIT_RULE_TYPE_PERCENTAGE,
    value: 5.0,
    min_amount: 1000.00,  // Применяется только к заказам ≥ $1000
    max_amount: null,
    priority: 20
)
```

### 7.3 Приоритеты

- **Priority:** Выше = применяется первым
- Правила применяются **последовательно** (не аккумулятивно)
- Если несколько правил подходят → используется **первое** по приоритету

### 7.4 Расчёт комиссии

```go
CalculateFee(
    connected_account_id: "...",
    amount: "1000.00",
    currency: "USD"
)

// Возвращает:
{
  total_fee: "100.00",
  applied_rules: [
    {
      rule_id: "...",
      rule_name: "Platform fee 10%",
      rule_type: SPLIT_RULE_TYPE_PERCENTAGE,
      fee_amount: "100.00"
    }
  ],
  transfer_amount: "900.00"  // amount - total_fee
}
```

### 7.5 Автоматическое применение при Transfer

При создании `Transfer` можно:
1. **Вручную указать amount** (уже с вычетом fee)
2. **Использовать CalculateFee** для автоматического расчёта

Пример:
```go
// 1. Calculate fee
feeResp := CalculateFee(account_id, "1000.00", "USD")
// total_fee = 100.00, transfer_amount = 900.00

// 2. Create transfer
CreateTransfer(
    connected_account_id: account_id,
    amount: feeResp.transfer_amount,  // 900.00
    currency: "USD",
    description: "Payment for Order #12345"
)
```

---

## 8. Payouts (Выплаты на банковский счёт)

### 8.1 Создание Payout

```go
CreatePayout(
    connected_account_id: "...",
    amount: "500.00",
    currency: "USD",
    method: PAYOUT_METHOD_STANDARD,  // Free, 2-3 days
    description: "Weekly payout",
    statement_descriptor: "VONDI PAYOUT",  // Max 22 chars
    metadata: {"week": "2024-W50"}
)
```

**Методы:**
- `PAYOUT_METHOD_STANDARD` — бесплатно, 2-3 рабочих дня
- `PAYOUT_METHOD_INSTANT` — платно (~1%), до 30 минут

### 8.2 Статусы Payout

- `pending` → `in_transit` → `paid` (успех)
- `pending` → `failed` (недостаточно средств, неверные банковские данные)
- `pending` → `canceled` (отменён вручную)

### 8.3 Отмена Payout

```go
CancelPayout(payout_id: "...")
// Работает только для pending payouts
// После in_transit → отменить нельзя
```

### 8.4 Проверка Arrival Date

```go
GetPayout(payout_id: "...")
// Возвращает:
{
  status: PAYOUT_STATUS_IN_TRANSIT,
  arrival_date: "2024-12-24T00:00:00Z",  // Когда средства поступят на счёт
  paid_at: null
}
```

### 8.5 Автоматические Payouts

Можно настроить в Stripe Dashboard:
- **Daily** — каждый день
- **Weekly** — раз в неделю
- **Monthly** — раз в месяц
- **Manual** — только через API

---

## 9. Миграции

**Директория:** `/p/github.com/vondi-global/payment/migrations`

**Количество:** 12 миграций (0001 → 0012)

### 9.1 Список миграций

| # | Файл | Описание |
|---|------|----------|
| 0001 | `create_payments_table` | Основная таблица `payments` |
| 0002 | `create_enums` | ENUM типы (payment_status, payment_type, escrow_status, refund_status) |
| 0003 | `update_payments_table` | Добавление `gateway_*`, `merchant_id`, `buyer_id`, `escrow_*` |
| 0004 | `create_escrow_holds_table` | Таблица `escrow_holds` |
| 0005 | `create_refunds_table` | Таблица `refunds` |
| 0006 | `create_webhook_logs_table` | Таблица `webhook_logs` (логи событий) |
| 0007 | `add_idempotency_key` | Поле `idempotency_key` в `payments` |
| 0008 | `create_connected_accounts_table` | Таблица `connected_accounts` + ENUMs |
| 0009 | `create_transfers_table` | Таблица `transfers` + `transfer_reversals` + ENUM |
| 0010 | `make_stripe_ids_nullable` | `stripe_payout_id` nullable (создаётся позже) |
| 0011 | `phase3_balance_payouts_splits` | Таблицы `balance_snapshots`, `payouts`, `split_rules` + ENUMs |
| 0012 | `create_payment_splits` | Таблица `payment_splits` (сохранение расчётов комиссий) |

### 9.2 Основные таблицы

| Таблица | Назначение |
|---------|-----------|
| `payments` | Основная таблица платежей |
| `escrow_holds` | Холдирование средств |
| `refunds` | Возвраты |
| `connected_accounts` | Stripe Connect аккаунты вендоров |
| `transfers` | Переводы платформа → вендор |
| `transfer_reversals` | Отмена/возврат трансферов |
| `balance_snapshots` | История балансов connected accounts |
| `payouts` | Выплаты на банковские счета |
| `split_rules` | Правила комиссий |
| `payment_splits` | Применённые комиссии к платежам |
| `webhook_logs` | Логи webhook событий |

### 9.3 ENUMs

- `payment_status`: pending, processing, succeeded, failed, cancelled, refunded
- `payment_type`: platform_service, product_b2c, product_c2c, balance_deposit
- `escrow_status`: holding, released, cancelled
- `refund_status`: pending, succeeded, failed
- `connected_account_type`: express, standard, custom
- `connected_account_status`: pending, active, restricted, disabled
- `transfer_status`: pending, paid, failed, reversed, partially_reversed
- `payout_status`: pending, in_transit, paid, failed, canceled
- `payout_method`: standard, instant
- `split_rule_type`: percentage, fixed

### 9.4 Применение миграций

```bash
cd /p/github.com/vondi-global/payment
make migrate-up       # Применить все
make migrate-down     # Откатить последнюю
make migrate-force 5  # Force версию 5
```

---

## 10. Интеграции

### 10.1 Auth Service

**Тип:** gRPC Client
**Адрес:** `localhost:20053` (dev)
**Назначение:** Валидация JWT токенов для gRPC методов

**Конфигурация:**
```bash
VONDIPAYMENT_AUTH_ENABLED=true
VONDIPAYMENT_AUTH_GRPC_ADDRESS=localhost:20053
VONDIPAYMENT_AUTH_SERVICE_API_KEY=***
```

**Пропуск методов:**
```bash
VONDIPAYMENT_AUTH_SKIP_METHODS=/payment.v1.PaymentService/HealthCheck
```

### 10.2 Stripe Gateway

**Тип:** HTTP Client (Stripe Go SDK)
**Функции:** Checkout, Payments, Connect, Transfers, Payouts

**Подключение:**
```go
stripe.Key = cfg.Gateways.Stripe.APIKey
```

**Webhook Signature:**
```go
stripe.VerifySignature(
    payload,
    signature,
    cfg.Gateways.Stripe.WebhookSecret
)
```

### 10.3 AllSecure Gateway

**Тип:** HTTP Client (Custom)
**Регион:** Serbian market (RSD, EUR)
**Base URL:** `https://asxgw.paymentsandbox.cloud/api/v3`

**Конфигурация:**
```bash
VONDIPAYMENT_GATEWAYS_DEFAULT=allsecure
VONDIPAYMENT_GATEWAYS_ALLSECURE_ENABLED=true
VONDIPAYMENT_GATEWAYS_ALLSECURE_API_KEY=***
```

---

## 11. Локальный запуск

### 11.1 Подготовка

```bash
cd /p/github.com/vondi-global/payment

# 1. Установить зависимости
go mod download

# 2. Запустить PostgreSQL
docker-compose up -d

# 3. Применить миграции
make migrate-up

# 4. (Опционально) Загрузить фикстуры
VONDIPAYMENT_FIXTURES_ENABLED=true make run
```

### 11.2 Запуск сервиса

```bash
# Через Makefile
make run

# Или напрямую
go run cmd/server/main.go
```

**Проверка:**
```bash
# Health check (без auth)
grpcurl -plaintext localhost:30051 payment.v1.PaymentService/HealthCheck

# Метрики
curl http://localhost:39090/metrics
```

### 11.3 Тестирование

```bash
# Unit тесты
make test

# Интеграционные тесты
make test-integration

# E2E тесты
make test-e2e

# Все тесты
make test-all
```

### 11.4 Линтеры

```bash
# Lint Go code
make lint

# Lint proto files
make proto-lint

# Format code
make format

# Fix imports
make imports
```

---

## 12. Примеры использования

### 12.1 Example Shop

**Директория:** `/p/github.com/vondi-global/payment/examples/shop`

**Описание:** Полноценный интернет-магазин с интеграцией Payment Service:
- Backend (Go + Fiber)
- Frontend (HTML + Vanilla JS)
- Checkout flow
- Webhook processing
- Admin panel (refunds)

**Запуск:**
```bash
cd examples/shop
make docker-up
# Открыть http://localhost:8080
```

**Функции:**
- Просмотр каталога товаров
- Создание checkout сессии
- Оплата через Stripe (или Mock)
- Обработка webhook
- Просмотр истории платежей
- Создание возвратов

### 12.2 gRPC Client Example

```go
import (
    paymentv1 "github.com/vondi-global/payment/pkg/pb/payment/v1"
    "google.golang.org/grpc"
)

// 1. Подключение
conn, err := grpc.Dial("localhost:30051", grpc.WithInsecure())
client := paymentv1.NewPaymentServiceClient(conn)

// 2. Создание checkout
resp, err := client.CreateCheckoutSession(ctx, &paymentv1.CreateCheckoutSessionRequest{
    UserId:       userID,
    Amount:       "100.50",
    Currency:     "USD",
    PaymentType:  paymentv1.PaymentType_PAYMENT_TYPE_PRODUCT_B2C,
    SuccessUrl:   "https://vondi.rs/payment/success",
    CancelUrl:    "https://vondi.rs/payment/cancel",
    EscrowEnabled: true,
    EscrowHoldDays: 7,
    MerchantId:   merchantID,
})

// 3. Redirect пользователя
// resp.CheckoutUrl → Stripe Checkout

// 4. После успешной оплаты → webhook
// ProcessWebhookEvent() → Payment status = succeeded → создаётся EscrowHold

// 5. Через 7 дней → cron вызывает ProcessExpiredEscrows
// → Release escrow → CreateTransfer на вендора (с учётом fee)
```

---

## 13. Документация

### 13.1 Основные файлы

| Файл | Описание |
|------|----------|
| `README.md` | Быстрый старт |
| `CLAUDE.md` | Инструкции для Claude (Pre-PR checklist) |
| `CHANGELOG.md` | История версий |
| `docs/STRIPE_CONNECT_GATEWAY_API.md` | Stripe Connect API |
| `docs/ALLSECURE_INTEGRATION_GUIDE.md` | AllSecure Integration |
| `examples/shop/README.md` | Example Shop |

### 13.2 Proto документация

**Генерация:**
```bash
make proto       # Генерация Go кода из .proto
make proto-lint  # Lint proto файлов
```

**Просмотр методов:**
```bash
grpcurl -plaintext localhost:30051 list payment.v1.PaymentService
```

---

## 14. CI/CD

### 14.1 GitHub Actions

**Файл:** `.github/workflows/ci.yml`

**Проверки:**
1. `make proto-lint` + `make proto`
2. `make check` (golangci-lint)
3. `make test` (unit тесты)
4. `make build` (сборка)
5. `examples/shop` build (Go + yarn lint + yarn build)

**Runner:** Self-hosted (svetu.rs)

### 14.2 Pre-commit hooks

```bash
# Install
make install-hooks

# Runs automatically before git commit:
# - make proto-lint
# - make proto
# - make imports
# - make format
# - make lint
```

---

## 15. Мониторинг и Логирование

### 15.1 Prometheus Метрики

**Endpoint:** `http://localhost:39090/metrics`

**Метрики:**
- `payment_checkout_sessions_total` — количество checkout сессий
- `payment_webhook_events_total` — обработанные webhook
- `escrow_holds_total` — количество escrow holds
- `escrow_releases_total` — количество releases
- `transfers_total` — количество трансферов
- `payouts_total` — количество payouts

### 15.2 Логирование

**Библиотека:** `github.com/rs/zerolog`

**Уровни:**
```bash
VONDIPAYMENT_LOG_LEVEL=debug|info|warn|error
```

**Формат:** JSON (для Elasticsearch/Loki)

---

## 16. Безопасность

### 16.1 Idempotency Keys

Поддерживаются для методов:
- `CreateCheckoutSession`
- `CreateRefund`
- `CreateTransfer`
- `ReverseTransfer`
- `CreatePayout`

**Использование:**
```go
CreateCheckoutSession(
    idempotency_key: "order-12345-payment-v1"
)
// Повторный вызов → вернёт существующий payment
```

**Хранение:** В БД (`payments.idempotency_key`, `refunds.idempotency_key`, и т.д.)

### 16.2 Webhook Signature Verification

**Stripe:**
```go
stripe.VerifySignature(payload, signature, webhookSecret)
```

**AllSecure:**
```go
// HMAC-SHA256 signature verification
```

**Reject неверифицированные webhook!**

### 16.3 Auth (JWT)

**Все gRPC методы** требуют валидный JWT (кроме `/HealthCheck`):
- Токен передаётся в metadata: `authorization: Bearer <token>`
- Проверяется через Auth Service gRPC

---

## 17. Troubleshooting

### 17.1 Database Connection Failed

**Проблема:** `connection refused on port 35433`

**Решение:**
```bash
docker-compose up -d  # Запустить PostgreSQL
docker ps | grep payment  # Проверить статус
```

### 17.2 Stripe Webhook Verification Failed

**Проблема:** `stripe.InvalidSignature`

**Решение:**
1. Проверить `VONDIPAYMENT_GATEWAYS_STRIPE_WEBHOOK_SECRET`
2. Использовать webhook secret из Stripe Dashboard → Developers → Webhooks
3. **Локально:** использовать Stripe CLI `stripe listen --forward-to localhost:30051/webhooks/stripe`

### 17.3 Payout Failed (Insufficient Funds)

**Проблема:** `payout.status = failed`, `failure_code = insufficient_funds`

**Решение:**
1. Проверить баланс: `GetBalance(account_id, force_refresh=true)`
2. Проверить `available_amount >= payout.amount`
3. Подождать пока pending средства станут available (~2-3 дня после transfer)

### 17.4 Escrow не release автоматически

**Проблема:** Escrow hold истёк, но status всё ещё `holding`

**Решение:**
1. Проверить cron job: `ProcessExpiredEscrows` должен вызываться регулярно (раз в час)
2. Проверить логи: есть ли ошибки при release?
3. Вручную release: `ReleaseEscrowHold(escrow_id)`

### 17.5 Split Rule не применяется

**Проблема:** `CalculateFee` возвращает `total_fee = 0`

**Решение:**
1. Проверить `split_rule.is_active = true`
2. Проверить `min_amount` / `max_amount` фильтры
3. Проверить `currency` (для fixed type)
4. Проверить priority (если несколько правил)

---

## 18. Roadmap

### 18.1 Планируемые функции

- [ ] **PayPal Gateway** — альтернативный платёжный шлюз
- [ ] **Recurring Payments** — подписки и автоплатежи
- [ ] **Multi-Account Payouts** — выплата одним payout на несколько вендоров
- [ ] **Dispute Management** — обработка споров (chargebacks)
- [ ] **Tax Calculation** — интеграция с налоговыми API (Stripe Tax)
- [ ] **Fraud Detection** — интеграция с Stripe Radar

### 18.2 Оптимизации

- [ ] Redis caching для balance snapshots
- [ ] Batch processing для webhook events
- [ ] Dead letter queue для failed transfers
- [ ] Metrics dashboard (Grafana)

---

## 19. Контакты и поддержка

**Документация:** `/p/github.com/vondi-global/payment/README.md`
**Проблемы:** GitHub Issues в `vondi-global/payment`
**Вопросы:** Slack канал `#payment-service`

---

**Версия паспорта:** 1.0.0
**Дата обновления:** 2025-12-21
**Автор:** Claude Opus 4.5
