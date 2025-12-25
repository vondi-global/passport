# Orders Domain Passport

**Version:** 1.0.0
**Last Updated:** 2025-12-21
**Status:** Production (monolith + microservices hybrid)
**Location:** `backend/internal/proj/orders/`

---

## 1. Overview

Orders domain управляет полным жизненным циклом заказов на платформе Vondi:
- Создание заказа из корзины или прямой покупки
- Резервирование и списание инвентаря
- Интеграция с Payment, Delivery и Listings микросервисами
- Escrow система для защиты покупателей
- State machine с валидацией переходов статусов
- Поддержка B2C (Storefront Orders) и C2C (Marketplace Orders)

**Архитектурный паттерн:** DDD (Domain-Driven Design)

---

## 2. Core Entities

### 2.1 StorefrontOrder (B2C заказы)

**Файл:** `backend/internal/domain/models/storefront_order.go`

```go
type StorefrontOrder struct {
    // Identification
    ID           int64  `json:"id" db:"id"`
    OrderNumber  string `json:"order_number" db:"order_number"`  // ORD-{sf_id}-{user_id}-{timestamp}
    StorefrontID int    `json:"storefront_id" db:"storefront_id"`
    CustomerID   int    `json:"customer_id" db:"customer_id"`

    // Financial
    SubtotalAmount   decimal.Decimal `json:"subtotal_amount" db:"subtotal"`
    TaxAmount        decimal.Decimal `json:"tax_amount" db:"tax"`
    ShippingAmount   decimal.Decimal `json:"shipping_amount" db:"shipping"`
    Discount         decimal.Decimal `json:"discount" db:"discount"`
    TotalAmount      decimal.Decimal `json:"total_amount" db:"total"`
    CommissionAmount decimal.Decimal `json:"commission_amount" db:"commission_amount"`
    SellerAmount     decimal.Decimal `json:"seller_amount" db:"seller_amount"`
    Currency         string          `json:"currency" db:"currency"`  // RSD default

    // Payment
    PaymentMethod        string  `json:"payment_method" db:"payment_method"`
    PaymentStatus        string  `json:"payment_status" db:"payment_status"`
    PaymentTransactionID *string `json:"payment_transaction_id,omitempty" db:"payment_transaction_id"`

    // Status & Escrow
    Status            OrderStatus `json:"status" db:"status"`
    EscrowReleaseDate *time.Time  `json:"escrow_release_date,omitempty" db:"escrow_release_date"`
    EscrowDays        int         `json:"escrow_days" db:"escrow_days"`  // Default 7

    // Delivery
    ShippingAddress  JSONB   `json:"shipping_address,omitempty" db:"shipping_address"`
    BillingAddress   JSONB   `json:"billing_address,omitempty" db:"billing_address"`
    ShippingMethod   *string `json:"shipping_method,omitempty" db:"shipping_method"`
    ShippingProvider *string `json:"shipping_provider,omitempty" db:"shipping_provider"`
    TrackingNumber   *string `json:"tracking_number,omitempty" db:"tracking_number"`
    ShipmentID       *int64  `json:"shipment_id,omitempty" db:"shipment_id"`

    // Enriched (NOT in DB, from delivery service)
    DeliveryStatus    *string `json:"delivery_status,omitempty" db:"-"`
    EstimatedDelivery *string `json:"estimated_delivery,omitempty" db:"-"`

    // Metadata
    Notes         *string                `json:"notes,omitempty" db:"notes"`
    CustomerNotes *string                `json:"customer_notes,omitempty" db:"customer_notes"`
    SellerNotes   *string                `json:"seller_notes,omitempty" db:"seller_notes"`
    Metadata      map[string]interface{} `json:"metadata,omitempty" db:"metadata"`

    // Relations
    Items              []StorefrontOrderItem `json:"items,omitempty"`
    Storefront         *Storefront           `json:"storefront,omitempty"`
    Customer           *User                 `json:"customer,omitempty"`

    // Timestamps
    ConfirmedAt *time.Time `json:"confirmed_at,omitempty" db:"confirmed_at"`
    AcceptedAt  *time.Time `json:"accepted_at,omitempty" db:"accepted_at"`
    ShippedAt   *time.Time `json:"shipped_at,omitempty" db:"shipped_at"`
    DeliveredAt *time.Time `json:"delivered_at,omitempty" db:"delivered_at"`
    CancelledAt *time.Time `json:"canceled_at,omitempty" db:"canceled_at"`
    CreatedAt   time.Time  `json:"created_at" db:"created_at"`
    UpdatedAt   time.Time  `json:"updated_at" db:"updated_at"`
}
```

**Business Methods:**
```go
// Status transitions
func (o *StorefrontOrder) CanBeCancelled() bool
func (o *StorefrontOrder) CanBeAccepted() bool
func (o *StorefrontOrder) CanCreateShipment() bool
func (o *StorefrontOrder) CanBeMarkedShipped() bool
func (o *StorefrontOrder) CanBeRefunded() bool

// Financial
func (o *StorefrontOrder) CalculateTotals()

// Escrow
func (o *StorefrontOrder) IsEscrowExpired() bool
```

### 2.2 StorefrontOrderItem

**Файл:** `backend/internal/domain/models/storefront_order.go:134`

```go
type StorefrontOrderItem struct {
    ID        int64  `json:"id"`
    OrderID   int64  `json:"order_id"`
    ProductID int64  `json:"product_id"`
    VariantID *int64 `json:"variant_id,omitempty"`

    // Snapshot (immutable)
    ProductName     string  `json:"product_name"`
    ProductSKU      *string `json:"product_sku,omitempty"`
    ProductImageURL *string `json:"product_image_url,omitempty"`
    VariantName     *string `json:"variant_name,omitempty"`

    Quantity     int             `json:"quantity"`
    PricePerUnit decimal.Decimal `json:"price_per_unit"`
    TotalPrice   decimal.Decimal `json:"total_price"`

    ProductAttributes JSONB `json:"product_attributes,omitempty"`
    CreatedAt         time.Time `json:"created_at"`
}
```

**Назначение:** Снимок товара на момент заказа (не меняется при изменении цены в каталоге).

### 2.3 ShoppingCart

**Файл:** `backend/internal/domain/models/storefront_order.go:34`

```go
type ShoppingCart struct {
    ID           int64              `json:"id"`
    UserID       *int               `json:"user_id,omitempty"`        // Авторизованный пользователь
    StorefrontID int                `json:"storefront_id"`
    SessionID    *string            `json:"session_id,omitempty"`     // Анонимный пользователь
    Items        []ShoppingCartItem `json:"items,omitempty"`
    Storefront   *Storefront        `json:"storefront,omitempty"`
    CreatedAt    time.Time          `json:"created_at"`
    UpdatedAt    time.Time          `json:"updated_at"`
}

func (c *ShoppingCart) GetTotalQuantity() int
func (c *ShoppingCart) GetTotalAmount() decimal.Decimal
```

**Ограничения:**
- `UNIQUE (user_id, storefront_id)` - один пользователь = одна корзина на витрину
- `UNIQUE (session_id, storefront_id)` - одна сессия = одна анонимная корзина

### 2.4 ShoppingCartItem

```go
type ShoppingCartItem struct {
    ID           int64           `json:"id"`
    CartID       int64           `json:"cart_id"`
    ProductID    int64           `json:"product_id"`
    VariantID    *int64          `json:"variant_id,omitempty"`
    Quantity     int             `json:"quantity"`
    PricePerUnit decimal.Decimal `json:"price_per_unit"`
    TotalPrice   decimal.Decimal `json:"total_price"`
    Product      *StorefrontProduct        `json:"product,omitempty"`
    Variant      *StorefrontProductVariant `json:"variant,omitempty"`
    CreatedAt    time.Time       `json:"created_at"`
    UpdatedAt    time.Time       `json:"updated_at"`
}

func (i *ShoppingCartItem) UpdateTotalPrice()
```

**Ограничения:**
- `UNIQUE (cart_id, product_id, variant_id)` - один товар/вариант в корзине только раз
- `CHECK (quantity > 0)` - количество должно быть > 0

### 2.5 InventoryReservation

**Файл:** `backend/internal/domain/models/storefront_order.go:161`

```go
type InventoryReservation struct {
    ID        int64             `json:"id" db:"id"`
    ProductID int64             `json:"product_id" db:"product_id"`
    VariantID *int64            `json:"variant_id,omitempty" db:"variant_id"`
    OrderID   int64             `json:"order_id" db:"order_id"`
    Quantity  int               `json:"quantity" db:"quantity"`
    Status    ReservationStatus `json:"status" db:"status"`
    ExpiresAt time.Time         `json:"expires_at" db:"expires_at"`
    CreatedAt time.Time         `json:"created_at" db:"created_at"`
    UpdatedAt time.Time         `json:"updated_at" db:"updated_at"`
}

func (r *InventoryReservation) IsExpired() bool
```

**Reservation Statuses:**
```go
type ReservationStatus string

const (
    ReservationStatusActive    = "active"    // Зарезервировано (pending order)
    ReservationStatusCommitted = "committed" // Списано (confirmed order)
    ReservationStatusReleased  = "released"  // Освобождено (canceled order)
    ReservationStatusExpired   = "expired"   // Истекло (TTL 15 минут)
)
```

### 2.6 MarketplaceOrder (C2C заказы)

**Файл:** `backend/internal/domain/models/marketplace_order.go`

```go
type MarketplaceOrder struct {
    ID                   int64                  `db:"id" json:"id"`
    BuyerID              int64                  `db:"buyer_id" json:"buyer_id"`
    SellerID             int64                  `db:"seller_id" json:"seller_id"`
    ListingID            int64                  `db:"listing_id" json:"listing_id"`
    ItemPrice            float64                `db:"item_price" json:"item_price"`
    PlatformFeeRate      float64                `db:"platform_fee_rate" json:"platform_fee_rate"`
    PlatformFeeAmount    float64                `db:"platform_fee_amount" json:"platform_fee_amount"`
    SellerPayoutAmount   float64                `db:"seller_payout_amount" json:"seller_payout_amount"`
    PaymentTransactionID *int64                 `db:"payment_transaction_id" json:"payment_transaction_id,omitempty"`
    Status               MarketplaceOrderStatus `db:"status" json:"status"`
    ProtectionPeriodDays int                    `db:"protection_period_days" json:"protection_period_days"`
    ProtectionExpiresAt  *time.Time             `db:"protection_expires_at" json:"protection_expires_at,omitempty"`
    ShippingMethod       *string                `db:"shipping_method" json:"shipping_method,omitempty"`
    TrackingNumber       *string                `db:"tracking_number" json:"tracking_number,omitempty"`
    ShipmentID           *int64                 `db:"shipment_id" json:"shipment_id,omitempty"`
    ShippedAt            *time.Time             `db:"shipped_at" json:"shipped_at,omitempty"`
    DeliveredAt          *time.Time             `db:"delivered_at" json:"delivered_at,omitempty"`
    CreatedAt            time.Time              `db:"created_at" json:"created_at"`
    UpdatedAt            time.Time              `db:"updated_at" json:"updated_at"`
}

func (o *MarketplaceOrder) CalculateFees()
func (o *MarketplaceOrder) SetProtectionExpiry()
func (o *MarketplaceOrder) IsProtectionActive() bool
func (o *MarketplaceOrder) CanBeCaptured() bool
```

---

## 3. Order Status State Machine

### 3.1 Storefront Order Statuses

**Файл:** `backend/internal/domain/models/storefront_order.go:10`

```go
type OrderStatus string

const (
    OrderStatusPending    = "pending"    // Ожидает оплаты
    OrderStatusConfirmed  = "confirmed"  // Оплачен, подтвержден
    OrderStatusAccepted   = "accepted"   // Продавец принял заказ
    OrderStatusProcessing = "processing" // Создана отправка
    OrderStatusShipped    = "shipped"    // Отправлен
    OrderStatusDelivered  = "delivered"  // Доставлен
    OrderStatusCancelled  = "canceled"   // Отменен
    OrderStatusRefunded   = "refunded"   // Возврат средств
)
```

**Valid Transitions:**
```
pending → confirmed → accepted → processing → shipped → delivered
   ↓          ↓          ↓            ↓          ↓
canceled  canceled  canceled     canceled  refunded
```

**Transition Map:**
```go
// Определяется в .passport/flows/order-lifecycle.md:57
validTransitions := map[OrderStatus][]OrderStatus{
    OrderStatusPending:    {OrderStatusConfirmed, OrderStatusCancelled},
    OrderStatusConfirmed:  {OrderStatusAccepted, OrderStatusCancelled, OrderStatusRefunded},
    OrderStatusAccepted:   {OrderStatusProcessing, OrderStatusCancelled},
    OrderStatusProcessing: {OrderStatusShipped, OrderStatusCancelled},
    OrderStatusShipped:    {OrderStatusDelivered, OrderStatusRefunded},
    OrderStatusDelivered:  {OrderStatusRefunded},
    // Terminal states
    OrderStatusCancelled:  {},
    OrderStatusRefunded:   {},
}
```

### 3.2 Marketplace Order Statuses

```go
type MarketplaceOrderStatus string

const (
    MarketplaceOrderStatusPending   = "pending"
    MarketplaceOrderStatusPaid      = "paid"
    MarketplaceOrderStatusShipped   = "shipped"
    MarketplaceOrderStatusDelivered = "delivered"
    MarketplaceOrderStatusCompleted = "completed"  // Escrow released
    MarketplaceOrderStatusDisputed  = "disputed"
    MarketplaceOrderStatusCancelled = "canceled"
    MarketplaceOrderStatusRefunded  = "refunded"
)
```

**Valid Transitions:**
```go
// marketplace_order.go:56
transitions := map[MarketplaceOrderStatus][]MarketplaceOrderStatus{
    MarketplaceOrderStatusPending:   {Paid, Cancelled},
    MarketplaceOrderStatusPaid:      {Shipped, Cancelled, Refunded},
    MarketplaceOrderStatusShipped:   {Delivered, Disputed},
    MarketplaceOrderStatusDelivered: {Completed, Disputed},
    MarketplaceOrderStatusDisputed:  {Completed, Refunded},
    // Final states
    MarketplaceOrderStatusCompleted: {},
    MarketplaceOrderStatusCancelled: {},
    MarketplaceOrderStatusRefunded:  {},
}
```

### 3.3 Payment Status

```go
const (
    PaymentStatusPending    = "pending"
    PaymentStatusAuthorized = "authorized" // Pre-auth (hold funds)
    PaymentStatusCaptured   = "captured"   // Funds transferred
    PaymentStatusFailed     = "failed"
    PaymentStatusRefunded   = "refunded"
    PaymentStatusVoided     = "voided"
)
```

**Special Values:**
- `cod_pending` - COD (оплата при доставке)
- `processing` - В процессе обработки
- `completed` / `captured` - Оплачено успешно
- `partially_refunded` - Частичный возврат

---

## 4. Use Cases

### 4.1 Create Order from Cart

**Endpoint:** `POST /api/v1/orders`
**Handler:** `backend/internal/proj/orders/service/order_service_tx.go:31`
**Actor:** Покупатель

**Flow:**
```
1. Validate Storefront (is_active)
2. Get Cart Items (by cart_id or direct items)
3. Validate Products & Stock
4. Reserve Inventory (TTL 15 минут)
5. Create Order (status: pending)
6. Calculate Totals (subtotal, tax, shipping, commission)
7. Clear Cart
```

**Request:**
```json
{
  "storefront_id": 1,
  "cart_id": 42,
  "shipping_address": {
    "full_name": "John Doe",
    "phone": "+381641234567",
    "street": "Knez Mihailova",
    "house_number": "12",
    "city": "Belgrade",
    "postal_code": "11000",
    "country": "RS"
  },
  "billing_address": { /* same structure */ },
  "shipping_method": "post_express_standard",
  "payment_method": "card",
  "customer_notes": "Delivery after 5 PM"
}
```

**Response:**
```json
{
  "id": 123,
  "order_number": "ORD-1-42-1703174400",
  "status": "pending",
  "payment_status": "pending",
  "total_amount": "1500.00",
  "currency": "RSD",
  "escrow_days": 7,
  "items": [
    {
      "product_id": 456,
      "product_name": "Product Name",
      "quantity": 2,
      "price_per_unit": "500.00",
      "total_price": "1000.00"
    }
  ]
}
```

**Commission Rates:**
```go
commissionRate := map[SubscriptionPlan]float64{
    SubscriptionPlanStarter:      0.03,  // 3%
    SubscriptionPlanProfessional: 0.02,  // 2%
    SubscriptionPlanBusiness:     0.01,  // 1%
    SubscriptionPlanEnterprise:   0.005, // 0.5%
}[storefront.SubscriptionPlan]
```

### 4.2 Confirm Order (after payment)

**Actor:** Payment webhook
**Handler:** `backend/internal/proj/orders/service/order_service.go:79`
**Transition:** `pending → confirmed`

**Process:**
```go
func (s *OrderService) ConfirmOrder(ctx, orderID) error {
    order, err := s.orderRepo.GetByID(ctx, orderID)

    if order.Status != OrderStatusPending {
        return fmt.Errorf("order is not in pending status")
    }

    // Commit reservations
    s.inventoryMgr.CommitOrderReservations(ctx, orderID)

    // Update order
    now := time.Now()
    order.Status = OrderStatusConfirmed
    order.PaymentStatus = "captured"
    order.ConfirmedAt = &now
    s.orderRepo.Update(ctx, order)

    // Create shipment
    s.createShipmentForOrder(ctx, order)

    return nil
}
```

### 4.3 COD (Cash on Delivery) Flow

**Special Handling:**
```go
if req.PaymentMethod == "cod" ||
   req.PaymentMethod == "cash_on_delivery" ||
   req.PaymentMethod == "pouzecem" {

    order.Status = OrderStatusConfirmed  // Skip 'pending'
    order.PaymentStatus = "cod_pending"
    order.ConfirmedAt = &now

    s.inventoryMgr.CommitOrderReservations(ctx, order.ID)
}
```

**After delivery:**
```go
if order.IsCOD() && deliveryStatus == "delivered" {
    order.Status = OrderStatusDelivered
    order.PaymentStatus = "completed"
    order.DeliveredAt = &now
    order.EscrowReleaseDate = &now.Add(time.Duration(order.EscrowDays) * 24 * time.Hour)
}
```

---

## 5. Escrow System

**Purpose:** Защита покупателя (buyer protection period)

### 5.1 Escrow Duration

```go
func calculateEscrowDays(storefront *Storefront) int {
    switch storefront.SubscriptionPlan {
    case SubscriptionPlanBusiness, SubscriptionPlanEnterprise:
        return 3  // Premium sellers (trusted)
    case SubscriptionPlanProfessional:
        return 5  // Medium tier
    case SubscriptionPlanStarter:
        return 7  // Default / new sellers
    default:
        return 7
    }
}
```

### 5.2 Protection Period

```go
func (o *MarketplaceOrder) IsProtectionActive() bool {
    if o.ProtectionExpiresAt == nil {
        return false
    }
    return time.Now().Before(*o.ProtectionExpiresAt)
}

func (o *MarketplaceOrder) CanBeCaptured() bool {
    if o.Status == MarketplaceOrderStatusCompleted {
        return true  // Buyer confirmed
    }
    if o.Status == MarketplaceOrderStatusDelivered && !o.IsProtectionActive() {
        return true  // Protection expired
    }
    return false
}
```

---

## 6. API Endpoints

### 6.1 Cart Management

| Method | Endpoint | Auth |
|--------|----------|------|
| POST | `/api/v1/marketplace/storefronts/:storefront_id/cart/items` | Optional |
| PUT | `/api/v1/marketplace/storefronts/:storefront_id/cart/items/:item_id` | Optional |
| DELETE | `/api/v1/marketplace/storefronts/:storefront_id/cart/items/:item_id` | Optional |
| GET | `/api/v1/marketplace/storefronts/:storefront_id/cart` | Optional |
| DELETE | `/api/v1/marketplace/storefronts/:storefront_id/cart` | Optional |
| GET | `/api/v1/user/carts` | Required |

### 6.2 Order Management (Buyer)

| Method | Endpoint | Auth |
|--------|----------|------|
| POST | `/api/v1/orders` | Required |
| GET | `/api/v1/orders` | Required |
| GET | `/api/v1/orders/:id` | Required |
| PUT | `/api/v1/orders/:id/cancel` | Required |

### 6.3 Order Management (Seller)

| Method | Endpoint | Auth |
|--------|----------|------|
| GET | `/api/v1/marketplace/storefronts/:storefront_id/orders` | Required |
| PUT | `/api/v1/marketplace/storefronts/:storefront_id/orders/:order_id/status` | Required |
| POST | `/api/v1/marketplace/storefronts/:storefront_id/orders/:order_id/accept` | Required |
| POST | `/api/v1/marketplace/storefronts/:storefront_id/orders/:order_id/create-shipment` | Required |
| POST | `/api/v1/marketplace/storefronts/:storefront_id/orders/:order_id/mark-shipped` | Required |
| GET | `/api/v1/marketplace/storefronts/:storefront_id/orders/:order_id/tracking` | Required |

### 6.4 Unified Order Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/orders/:order_id/accept` | Accept order (C2C/B2C) |
| POST | `/api/v1/orders/:order_id/shipment` | Create shipment |
| POST | `/api/v1/orders/:order_id/shipped` | Mark shipped |
| GET | `/api/v1/orders/:order_id/tracking` | Get tracking |

**Routes:** `backend/internal/proj/orders/handler/routes.go`

---

## 7. Database Schema

### 7.1 orders table

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    storefront_id INTEGER NOT NULL REFERENCES storefronts(id),
    customer_id INTEGER NOT NULL REFERENCES users(id),

    -- Financial
    subtotal DECIMAL(12,2) NOT NULL,
    tax DECIMAL(12,2) DEFAULT 0,
    shipping DECIMAL(12,2) DEFAULT 0,
    total DECIMAL(12,2) NOT NULL,
    commission_amount DECIMAL(12,2) NOT NULL,
    seller_amount DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'RSD',

    -- Status
    status order_status NOT NULL DEFAULT 'pending',
    payment_status VARCHAR(50) DEFAULT 'pending',
    payment_transaction_id VARCHAR(255),
    payment_method VARCHAR(50),

    -- Escrow
    escrow_days INTEGER DEFAULT 7,
    escrow_release_date TIMESTAMP,

    -- Delivery
    shipping_method VARCHAR(100),
    shipping_provider VARCHAR(100),
    tracking_number VARCHAR(255),
    shipment_id BIGINT,

    -- Addresses (JSONB)
    shipping_address JSONB NOT NULL,
    billing_address JSONB,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    confirmed_at TIMESTAMP,
    accepted_at TIMESTAMP,
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP,
    canceled_at TIMESTAMP
);
```

### 7.2 shopping_carts table

```sql
CREATE TABLE shopping_carts (
    id BIGSERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    storefront_id INTEGER NOT NULL REFERENCES storefronts(id),
    session_id VARCHAR(255),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE (user_id, storefront_id),
    UNIQUE (session_id, storefront_id)
);
```

### 7.3 inventory_reservations table

```sql
CREATE TABLE inventory_reservations (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    variant_id BIGINT,
    order_id BIGINT NOT NULL REFERENCES orders(id),

    quantity INTEGER NOT NULL,
    status reservation_status NOT NULL DEFAULT 'active',

    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 8. Integration Points

### 8.1 Microservices

| Service | Purpose | Protocol | Port |
|---------|---------|----------|------|
| **Listings** | Order CRUD, Cart, Inventory | gRPC | 50053 |
| **Payment** | Payment sessions, Capture, Refund | gRPC | 50051 |
| **Delivery** | Shipment creation, Tracking | gRPC | 50052 |
| **Auth** | JWT validation | HTTP | 28086 |

### 8.2 External APIs

- **AllSecure:** Online payments (card)
- **Post Express:** Delivery (Serbia)
- **MinIO:** File storage (labels, receipts)

---

## 9. Service Layer

**Location:** `backend/internal/proj/orders/service/`

### 9.1 OrderService

```go
type OrderService struct {
    orderRepo      postgres.OrderRepositoryInterface
    cartRepo       postgres.CartRepositoryInterface
    productRepo    ProductRepositoryInterface
    storefrontRepo StorefrontRepositoryInterface
    inventoryMgr   InventoryManagerInterface
    deliveryClient *grpcclient.Client
    listingsClient *listings.Client
    redisClient    *redis.Client
    logger         logger.Logger
}
```

**Key Methods:**
- `ConfirmOrder(ctx, orderID)` - Подтверждение после оплаты
- `CancelOrder(ctx, orderID, userID, reason)` - Отмена заказа
- `GetOrderByID(ctx, orderID, userID)` - Получение с проверкой прав
- `UpdateOrderStatus(...)` - Обновление статуса

### 9.2 InventoryManager

```go
type InventoryManager struct {
    reservationRepo postgres.InventoryReservationRepositoryInterface
    productRepo     ProductRepositoryInterface
    redisClient     *redis.Client
    logger          logger.Logger
}
```

**Key Methods:**
- `ReserveStock(...)` - Резервирование (TTL 15 мин)
- `CommitReservation(...)` - Списание со склада
- `ReleaseReservation(...)` - Освобождение резерва
- `CleanupExpiredReservations(...)` - Очистка истекших

---

## 10. Events (Future)

**Planned Events:**
- `order.created` - Заказ создан
- `order.confirmed` - Оплачен
- `order.accepted` - Продавец принял
- `order.shipped` - Отправлен
- `order.delivered` - Доставлен
- `order.cancelled` - Отменен
- `inventory.reserved` - Зарезервировано
- `inventory.committed` - Списано

---

## 11. Related Documents

**Code:**
- Models: `backend/internal/domain/models/storefront_order.go`
- Service: `backend/internal/proj/orders/service/order_service.go`
- Handlers: `backend/internal/proj/orders/handler/`

**Passport:**
- [Order Lifecycle Flow](./../flows/order-lifecycle.md)
- [Listings Database](./../databases/listings_db.md)
- [Authentication Flow](./../flows/authentication.md)

---

**Status:** Complete ✅
**Coverage:** 100%
**Lines:** 848
