# 3PL Gateway & Channel Adapters

> **Версия:** 1.0.0 | **Дата:** 2025-12-25 | **Статус:** PRODUCTION READY ✅
> **Реализация:** Phase D.3 - 3PL Platform APIs

---

## Обзор

3PL Gateway обеспечивает интеграцию WMS Prime с внешними маркетплейсами (eBay, Amazon, Etsy, Shopify) через Channel Adapters. Автоматизирует синхронизацию заказов, инвентаря и статусов отгрузки между платформами.

### Ключевые возможности

- ✅ **4 Channel Adapters** — eBay, Amazon SP-API, Etsy, Shopify
- ✅ **Bidirectional Sync** — заказы (marketplace → WMS), инвентарь (WMS → marketplace)
- ✅ **OAuth2 Flows** — авторизация для каждого marketplace
- ✅ **Webhook Handling** — реал-тайм обработка событий (orders/create, orders/paid)
- ✅ **Order Normalization** — унификация данных из разных источников
- ✅ **Tracking Pushback** — автоматическая отправка номеров отслеживания

---

## Архитектура

### Технологический стек

| Компонент | Технология |
|-----------|------------|
| **Backend** | Go 1.21+, Fiber v2 |
| **Database** | PostgreSQL (channel_integrations, channel_orders) |
| **Queue** | Redis (async job processing) |
| **External APIs** | eBay API, Amazon SP-API, Etsy API v3, Shopify GraphQL |
| **Port** | HTTP 8090 |

### Диаграмма потока данных

```
External Marketplace (eBay/Amazon/Etsy/Shopify)
       ↓ (Webhook: order created)
   Channel Adapter (OAuth2 authenticated)
       ↓
   Order Normalization (Marketplace → Vondi format)
       ↓
   WMS Gateway API → POST /api/v1/orders
       ↓
   Warehouse Service (gRPC 50055)
       ↓ (Order fulfilled, tracking number generated)
   Channel Adapter → Tracking Pushback
       ↓
   External Marketplace (tracking number updated)
```

---

## Channel Adapters

### 1. eBay Adapter

**OAuth2 Flow:** User Consent (Authorization Code)

**Endpoints:**
- `GET /channels/ebay/auth` — инициация OAuth2
- `GET /channels/ebay/callback` — OAuth callback
- `POST /channels/ebay/sync` — manual sync trigger

**Features:**
- Orders API integration (Trading API)
- Webhook handling:
  - `ITEM_SOLD` — новый заказ
  - `PAYMENT_COMPLETED` — оплата подтверждена
- Inventory sync (daily batch 02:00 UTC)
- Multi-account support

**Order Mapping:**
```json
// eBay Order
{
  "OrderID": "12345-67890",
  "BuyerUserID": "ebay_buyer_123",
  "ShippingAddress": {
    "Name": "John Doe",
    "Street1": "123 Main St",
    "CityName": "Belgrade",
    "PostalCode": "11000"
  },
  "TransactionArray": [
    {
      "Item": { "SKU": "ABC-001" },
      "QuantityPurchased": 2
    }
  ]
}

// Normalized Vondi Order
{
  "external_order_id": "EBAY-12345-67890",
  "source": "ebay",
  "customer": {
    "name": "John Doe",
    "email": "***@ebay.com"  // masked
  },
  "shipping_address": {
    "street": "123 Main St",
    "city": "Belgrade",
    "postal_code": "11000",
    "country": "RS"
  },
  "line_items": [
    { "sku": "ABC-001", "quantity": 2, "price": 1500.00 }
  ]
}
```

---

### 2. Amazon SP-API Adapter

**OAuth2 Flow:** LWA (Login with Amazon)

**Endpoints:**
- `GET /channels/amazon/auth` — OAuth2 initialization
- `GET /channels/amazon/callback` — LWA callback
- `POST /channels/amazon/orders/sync` — pull orders (polling)

**Features:**
- SP-API v0 (Selling Partner API)
- Order polling (every 15 minutes, cron job)
- RDT (Restricted Data Token) для PII (buyer name, address)
- Fulfillment updates (CompleteFulfillmentOrder)
- Multi-marketplace support (US, UK, DE, FR, IT, ES, JP)

**Order Polling:**
```go
// Cron Job: Every 15 minutes
func PollAmazonOrders(sellerID string) {
    lastSync := GetLastSyncTime(sellerID)
    orders := spAPI.GetOrders(&OrdersRequest{
        MarketplaceIds: []string{"A1PA6795UKMFR9"},  // Germany
        CreatedAfter:   lastSync,
        OrderStatuses:  []string{"Unshipped"},
    })

    for _, order := range orders {
        normalizedOrder := NormalizeAmazonOrder(order)
        CreateWMSOrder(normalizedOrder)
    }
}
```

**RDT Example:**
```go
// Get buyer PII using Restricted Data Token
rdt := spAPI.CreateRestrictedDataToken(&RDTRequest{
    RestrictedResources: []{
        {Method: "GET", Path: "/orders/v0/orders/{orderId}/buyerInfo"}
    }
})

buyerInfo := spAPI.GetOrderBuyerInfo(orderID, rdt.Token)
// buyerInfo contains email, phone (normally masked)
```

---

### 3. Etsy Adapter

**OAuth2 Flow:** App Authentication (OAuth2 PKCE)

**Endpoints:**
- `GET /channels/etsy/auth`
- `GET /channels/etsy/callback`
- `POST /channels/etsy/sync`

**Features:**
- Etsy API v3 (Open API 3.0)
- Shop Receipts API (orders)
- Listing inventory sync
- Webhooks (not officially supported, polling every 10 min)

**Order Mapping:**
```go
// Etsy Receipt
type EtsyReceipt struct {
    ReceiptID      int64
    BuyerEmail     string
    ShippingName   string
    ShippingAddress string
    Transactions   []EtsyTransaction
}

// Normalized Order
type VondiOrder struct {
    ExternalOrderID string `json:"external_order_id"` // "ETSY-123456"
    Source          string `json:"source"`            // "etsy"
    // ... standard fields
}
```

---

### 4. Shopify Adapter

**OAuth2 Flow:** Custom App (API Key + Access Token)

**Endpoints:**
- `GET /channels/shopify/auth`
- `POST /channels/shopify/webhooks/orders-create`
- `POST /channels/shopify/webhooks/orders-paid`

**Features:**
- Shopify Admin API (REST + GraphQL)
- Webhooks (real-time):
  - `orders/create`
  - `orders/paid`
  - `orders/fulfilled`
- Product sync (GraphQL bulk operations)
- Inventory updates via InventoryLevel API

**Webhook Payload Example:**
```json
// POST /channels/shopify/webhooks/orders-create
{
  "id": 12345,
  "email": "customer@example.com",
  "total_price": "150.00",
  "line_items": [
    {
      "sku": "ABC-001",
      "quantity": 2,
      "price": "75.00"
    }
  ],
  "shipping_address": {
    "first_name": "Marko",
    "last_name": "Marković",
    "address1": "Bulevar kralja Aleksandra 15",
    "city": "Beograd",
    "zip": "11000",
    "country_code": "RS"
  }
}
```

**GraphQL Inventory Update:**
```graphql
mutation inventorySetOnHandQuantities($input: InventorySetOnHandQuantitiesInput!) {
  inventorySetOnHandQuantities(input: $input) {
    userErrors {
      field
      message
    }
    inventoryAdjustmentGroup {
      createdAt
    }
  }
}
```

---

## Database Schema

### Table: `channel_integrations`

```sql
CREATE TABLE channel_integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID REFERENCES partners(id),
    channel_type VARCHAR(20) NOT NULL,  -- ebay, amazon, etsy, shopify
    shop_id VARCHAR(255),  -- external shop/seller ID
    access_token_encrypted TEXT NOT NULL,
    refresh_token_encrypted TEXT,
    token_expires_at TIMESTAMPTZ,
    marketplace_id VARCHAR(50),  -- для Amazon (A1PA6795UKMFR9)
    webhook_secret VARCHAR(255),
    sync_enabled BOOLEAN DEFAULT true,
    last_sync_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_integrations_partner ON channel_integrations(partner_id);
CREATE INDEX idx_integrations_channel ON channel_integrations(channel_type);
```

### Table: `channel_orders`

```sql
CREATE TABLE channel_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id UUID REFERENCES channel_integrations(id),
    external_order_id VARCHAR(255) UNIQUE NOT NULL,
    wms_order_id VARCHAR(255) REFERENCES orders(id),  -- после создания в WMS
    channel_type VARCHAR(20) NOT NULL,
    order_data JSONB NOT NULL,  -- raw order data
    normalized_data JSONB NOT NULL,  -- vondi format
    sync_status VARCHAR(20) DEFAULT 'pending',  -- pending, synced, failed
    synced_at TIMESTAMPTZ,
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_channel_orders_external ON channel_orders(external_order_id);
CREATE INDEX idx_channel_orders_wms ON channel_orders(wms_order_id);
CREATE INDEX idx_channel_orders_status ON channel_orders(sync_status);
```

### Table: `inventory_sync_log`

```sql
CREATE TABLE inventory_sync_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id UUID REFERENCES channel_integrations(id),
    sku VARCHAR(255) NOT NULL,
    old_quantity INT,
    new_quantity INT,
    sync_direction VARCHAR(10),  -- to_channel, from_channel
    sync_status VARCHAR(20) DEFAULT 'success',
    error_message TEXT,
    synced_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_sync_log_integration ON inventory_sync_log(integration_id);
CREATE INDEX idx_sync_log_date ON inventory_sync_log(synced_at DESC);
```

---

## API Endpoints

### Integration Management

#### POST /channels/{channel}/connect

Подключить новый marketplace.

**Request:**
```json
{
  "partner_id": "partner_abc123",
  "shop_id": "ebay_shop_456",
  "access_token": "oauth_token_xyz",
  "refresh_token": "refresh_abc",
  "marketplace_id": "EBAY_US"  // опционально для Amazon
}
```

**Response:**
```json
{
  "integration_id": "int_789",
  "status": "connected",
  "last_sync": null
}
```

---

#### GET /channels/{channel}/orders

Получить список заказов с marketplace.

**Query Parameters:**
- `integration_id` (required)
- `from_date` (optional)
- `sync_status` (optional) — pending, synced, failed

**Response:**
```json
{
  "orders": [
    {
      "id": "order_123",
      "external_order_id": "EBAY-12345",
      "wms_order_id": "ORD-789",
      "sync_status": "synced",
      "created_at": "2025-12-25T10:00:00Z"
    }
  ]
}
```

---

#### POST /channels/{channel}/sync/inventory

Синхронизировать инвентарь с marketplace.

**Request:**
```json
{
  "integration_id": "int_789",
  "inventory_updates": [
    {
      "sku": "ABC-001",
      "quantity": 450
    }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "updated": 1,
  "failed": 0
}
```

---

### Webhook Endpoints

#### POST /channels/shopify/webhooks/orders-create

Shopify webhook handler для новых заказов.

**Headers:**
```
X-Shopify-Topic: orders/create
X-Shopify-Hmac-SHA256: <signature>
X-Shopify-Shop-Domain: myshop.myshopify.com
```

**Signature Verification:**
```go
func VerifyShopifyWebhook(body []byte, hmac string, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    expectedMAC := base64.StdEncoding.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(expectedMAC), []byte(hmac))
}
```

---

## Order Normalization

### Унифицированный формат

Все заказы из маркетплейсов конвертируются в единый формат:

```go
type NormalizedOrder struct {
    ExternalOrderID string          `json:"external_order_id"`
    Source          string          `json:"source"`  // ebay, amazon, etsy, shopify
    Customer        CustomerInfo    `json:"customer"`
    ShippingAddress AddressInfo     `json:"shipping_address"`
    LineItems       []LineItem      `json:"line_items"`
    TotalAmount     float64         `json:"total_amount"`
    Currency        string          `json:"currency"`  // RSD, USD, EUR
    OrderDate       time.Time       `json:"order_date"`
}

type CustomerInfo struct {
    Name  string `json:"name"`
    Email string `json:"email"`   // может быть замаскирован
    Phone string `json:"phone"`   // опционально
}

type LineItem struct {
    SKU      string  `json:"sku"`
    Quantity int     `json:"quantity"`
    Price    float64 `json:"price"`
    TaxRate  float64 `json:"tax_rate"`  // опционально
}
```

---

## Error Handling & Retry Logic

### Retry Strategy

**Exponential Backoff:**
```go
func RetryWithBackoff(fn func() error, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        err := fn()
        if err == nil {
            return nil
        }

        if !isRetryable(err) {
            return err  // не ретраим permanent errors
        }

        backoff := time.Duration(math.Pow(2, float64(i))) * time.Second
        time.Sleep(backoff)
    }
    return errors.New("max retries exceeded")
}
```

**Retryable Errors:**
- HTTP 429 (Rate Limit)
- HTTP 503 (Service Unavailable)
- Network timeouts
- OAuth token expiration (auto-refresh)

**Non-Retryable:**
- HTTP 400 (Bad Request)
- HTTP 401 (Unauthorized)
- HTTP 404 (Not Found)

---

## Monitoring & Logging

### Prometheus Metrics

```go
channel_orders_synced_total{channel="ebay", status="success|failed"}
channel_inventory_synced_total{channel="shopify"}
channel_api_requests_total{channel, endpoint, status_code}
channel_webhook_received_total{channel, event_type}
oauth_token_refreshed_total{channel}
```

### Structured Logging

```go
log.Info("Order synced from marketplace",
    "channel", "amazon",
    "external_order_id", "AMZN-12345",
    "wms_order_id", "ORD-789",
    "duration_ms", 450,
)
```

---

## Deployment

### Docker Compose (Development)

```yaml
version: '3.9'
services:
  3pl-gateway:
    build: .
    ports:
      - "8090:8090"
    environment:
      - POSTGRES_DSN=postgres://wms_user:password@postgres:5432/wms_3pl
      - REDIS_URL=redis://redis:6379
      - WMS_GATEWAY_URL=http://wms-gateway:8093
    depends_on:
      - postgres
      - redis
      - wms-gateway
```

### Kubernetes

**Deployment:**
- Replicas: 2
- Resources: 512MB RAM, 200m CPU (requests)
- HPA: 2-5 pods

---

## Безопасность

### OAuth Token Storage

- ✅ Tokens encrypted at rest (AES-256-GCM)
- ✅ Separate encryption key per environment
- ✅ Auto-refresh для expired tokens

### Webhook Security

- ✅ HMAC signature verification (eBay, Shopify)
- ✅ IP whitelist (опционально)
- ✅ Replay attack protection (timestamp validation)

---

## Ссылки

- [WMS Gateway](/p/github.com/vondi-global/passport/services/wms-gateway.md)
- [Developer Portal](/p/github.com/vondi-global/passport/frontend/wms-dev-portal.md)
- [WMS Prime Complete Report](/p/github.com/vondi-global/docs/wms-prime/WMS_PRIME_100_PERCENT_COMPLETE.md)

---

**Последнее обновление:** 2025-12-25
**Maintainer:** WMS Prime Team
