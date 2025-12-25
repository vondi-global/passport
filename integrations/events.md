# Events & Messaging Passport - Vondi Platform

> **Версия:** 1.0.0 | **Обновлено:** 2025-12-21

## Краткое описание

Vondi использует **event-driven архитектуру** на базе **Redis Streams** для асинхронной коммуникации между микросервисами. Это обеспечивает слабую связанность, надежность доставки, масштабируемость и отказоустойчивость.

**Основные преимущества:**
- ✅ Асинхронная обработка
- ✅ Гарантированная доставка (consumer groups)
- ✅ Load balancing между инстансами
- ✅ Event sourcing паттерн
- ✅ Replay capability

---

## Оглавление

1. [Архитектура](#1-архитектура)
2. [Redis Streams конфигурация](#2-redis-streams-конфигурация)
3. [WMS Events](#3-wms-events)
4. [Order Events](#4-order-events)
5. [Payment Events](#5-payment-events)
6. [Delivery Events](#6-delivery-events)
7. [PVZ Events](#7-pvz-events)
8. [Notification Events](#8-notification-events)
9. [User Events](#9-user-events)
10. [Event Payload Schemas](#10-event-payload-schemas)
11. [Consumer Groups](#11-consumer-groups)
12. [Error Handling & Retry](#12-error-handling--retry)
13. [Monitoring & Observability](#13-monitoring--observability)

---

## 1. Архитектура

### 1.1 Event-Driven Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Redis Streams                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  wms:events:picking      ──▶  Picking events                        │
│  wms:events:packing      ──▶  Packing events                        │
│  wms:events:inventory    ──▶  Inventory changes                     │
│  wms:events:shipment     ──▶  Shipment events                       │
│                                                                      │
│  listings:events:orders  ──▶  Order lifecycle events                │
│                                                                      │
│  payments:events         ──▶  Payment status changes                │
│                                                                      │
│  delivery:events         ──▶  Delivery tracking                     │
│                                                                      │
│  pvz.events              ──▶  PVZ parcel/capacity events            │
│                                                                      │
│  notifications:events    ──▶  Notification delivery logs            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Producer/Consumer Pattern

```
┌──────────────┐    Publish     ┌─────────────────┐    Subscribe    ┌──────────────┐
│              │   ─────────▶    │  Redis Streams  │   ◀─────────    │              │
│   Producer   │                 │  (Message Bus)  │                 │   Consumer   │
│  (Service)   │                 └─────────────────┘                 │  (Service)   │
│              │                                                     │              │
└──────────────┘                                                     └──────────────┘
     │                                                                      │
     │ 1. Publish event                                                    │ 3. Process event
     │ 2. Get message ID                                                   │ 4. XAck message
     └─────────────────────────────────────────────────────────────────────┘
```

### 1.3 Services Communication Map

```
┌──────────────┐                                        ┌──────────────┐
│   Listings   │────▶ order.confirmed ────▶             │     WMS      │
│   Service    │                                        │   Service    │
└──────────────┘                                        └──────────────┘
       │                                                       │
       │ order.confirmed                                      │ picking.completed
       ▼                                                       ▼
┌──────────────┐                                        ┌──────────────┐
│ Notification │                                        │   Delivery   │
│   Service    │◀──── packing.completed ────            │   Service    │
└──────────────┘                                        └──────────────┘
       │                                                       │
       │ payment.completed                                     │ pvz.parcel.*
       ▼                                                       ▼
┌──────────────┐                                        ┌──────────────┐
│   Payment    │                                        │     PVZ      │
│   Service    │                                        │   Service    │
└──────────────┘                                        └──────────────┘
                                                               │
                                                               │ pvz.capacity.*
                                                               ▼
                                                        ┌──────────────┐
                                                        │   Routing    │
                                                        │   Decision   │
                                                        └──────────────┘
```

---

## 2. Redis Streams конфигурация

### 2.1 Redis Instances

| Service | Redis Port | Purpose | Database |
|---------|-----------|---------|----------|
| System | 6379 | Системный cache | 0 |
| Listings | 36380 | Order events, cache | 0 |
| Delivery | 36380 | Delivery events, cache | 0 |
| Auth | 26379 | Session, tokens | 0 |
| WMS | 36381 | WMS events | 0 |
| Notifications | 36382 | Notification queue | 0 |
| PVZ | 36384 | PVZ events, capacity cache | 0 |
| Admin Portal | 6380 | Admin cache | 0 |

### 2.2 Stream Naming Convention

**Формат:** `{service}:events:{context}` или `{service}.events`

**Примеры:**
- `wms:events:picking`
- `listings:events:orders`
- `payments:events:transactions`
- `pvz.events` (PVZ → Delivery)
- `delivery.events` (Delivery → PVZ)

### 2.3 Connection Configuration

**WMS Event Publisher (Go):**
```go
// internal/config/config.go
type Config struct {
    // WMS own Redis (for publishing WMS events)
    RedisHost     string // localhost
    RedisPort     string // 36381
    RedisPassword string
    RedisDB       int    // 0

    // Listings Redis (for consuming order events)
    ListingsRedisHost     string // localhost
    ListingsRedisPort     string // 36380
    ListingsRedisPassword string
    ListingsRedisDB       int    // 0
}

// Create Redis client
redisClient := redis.NewClient(&redis.Options{
    Addr:     fmt.Sprintf("%s:%s", cfg.RedisHost, cfg.RedisPort),
    Password: cfg.RedisPassword,
    DB:       cfg.RedisDB,
})
```

**Notifications Queue (Go):**
```go
// Notifications Redis
redisClient := redis.NewClient(&redis.Options{
    Addr:     "localhost:36382",
    Password: os.Getenv("VONDINOTIFY_REDIS_PASSWORD"),
    DB:       0,
})
```

---

## 3. WMS Events

### 3.1 Stream Overview

| Stream | Purpose | Producer | Consumers |
|--------|---------|----------|-----------|
| `wms:events:picking` | Picking task lifecycle | WMS Service | Delivery, Notifications |
| `wms:events:packing` | Packing task lifecycle | WMS Service | Delivery, Notifications |
| `wms:events:inventory` | Inventory changes | WMS Service | Listings, Analytics |
| `wms:events:shipment` | Shipment creation | WMS Service | Delivery |

### 3.2 Picking Events

#### Event Types

| Event | Trigger | Payload |
|-------|---------|---------|
| `picking.created` | Picking task created from order | task_id, order_id, warehouse_id |
| `picking.assigned` | Worker assigned to task | task_id, order_id, worker_id |
| `picking.completed` | Worker finished picking | task_id, order_id |

#### Publishing Code

```go
// internal/events/publisher.go
type EventPublisher interface {
    PublishPickingCreated(ctx context.Context, taskID int64, orderID int64, warehouseID int64) error
    PublishPickingAssigned(ctx context.Context, taskID int64, orderID int64, workerID int64) error
    PublishPickingCompleted(ctx context.Context, taskID int64, orderID int64) error
}

// Usage in service layer
func (s *PickingService) CreatePickingTask(ctx context.Context, orderID int64) error {
    task, err := s.repo.Create(ctx, orderID)
    if err != nil {
        return err
    }

    // Publish event (non-critical)
    if err := s.eventPublisher.PublishPickingCreated(ctx, task.ID, orderID, task.WarehouseID); err != nil {
        s.logger.Error().Err(err).Msg("Failed to publish picking.created event")
        // Continue - event publishing failure doesn't fail the operation
    }

    return nil
}
```

### 3.3 Packing Events

| Event | Trigger | Payload |
|-------|---------|---------|
| `packing.started` | Packing task created | task_id, order_id |
| `packing.completed` | Package ready for shipment | task_id, order_id |

### 3.4 Inventory Events

| Event | Trigger | Payload |
|-------|---------|---------|
| `inventory.placed` | Product placed in warehouse | listing_id, warehouse_id, quantity, location |
| `inventory.moved` | Product moved between locations | listing_id, warehouse_id, from_location, to_location |
| `inventory.reserved` | Inventory reserved for order | listing_id, warehouse_id, quantity, order_id |
| `inventory.released` | Reservation released/cancelled | listing_id, warehouse_id, quantity, reason |

### 3.5 Event Fields Standard

```go
// internal/events/streams.go
type EventFields struct {
    Type           string // "type"
    TaskID         string // "task_id"
    OrderID        string // "order_id"
    WarehouseID    string // "warehouse_id"
    ListingID      string // "listing_id"
    Quantity       string // "quantity"
    WorkerID       string // "worker_id"
    Timestamp      string // "timestamp"
    Reason         string // "reason"
    Location       string // "location"
    ShipmentID     string // "shipment_id"
    TrackingNumber string // "tracking_number"
}

func StandardFields() EventFields {
    return EventFields{
        Type:           "type",
        TaskID:         "task_id",
        OrderID:        "order_id",
        WarehouseID:    "warehouse_id",
        ListingID:      "listing_id",
        Quantity:       "quantity",
        WorkerID:       "worker_id",
        Timestamp:      "timestamp",
        Reason:         "reason",
        Location:       "location",
        ShipmentID:     "shipment_id",
        TrackingNumber: "tracking_number",
    }
}
```

---

## 4. Order Events

### 4.1 Stream: `listings:events:orders`

**Producer:** Listings Service
**Consumers:** WMS, Notifications

### 4.2 Event Types

| Event | Trigger | Payload |
|-------|---------|---------|
| `order.created` | Order placed by buyer | order_id, storefront_id, buyer_id, items[] |
| `order.confirmed` | Payment confirmed | order_id, storefront_id, items[] |
| `order.cancelled` | Order cancelled | order_id, reason |

### 4.3 order.confirmed Payload

```json
{
  "type": "order.confirmed",
  "order_id": 12345,
  "storefront_id": 67,
  "items": "[{\"listing_id\":100,\"quantity\":2,\"warehouse_id\":1},{\"listing_id\":200,\"quantity\":1,\"warehouse_id\":1}]",
  "timestamp": 1734825600
}
```

**Items JSON Schema:**
```go
type OrderItemEvent struct {
    ListingID   int64 `json:"listing_id"`
    Quantity    int32 `json:"quantity"`
    WarehouseID int64 `json:"warehouse_id"`
}
```

### 4.4 WMS Consumer Implementation

```go
// internal/events/consumer.go
type EventHandler interface {
    OnOrderConfirmed(ctx context.Context, orderID int64, storefrontID int64, items []OrderItemEvent) error
    OnOrderCancelled(ctx context.Context, orderID int64, reason string) error
}

// Consumer setup
consumer := events.NewRedisEventConsumer(listingsRedisClient, logger, "wms-instance-1")
handler := NewFulfillmentHandler(pickingService, inventoryService)
consumer.Start(ctx, handler)
defer consumer.Stop()
```

### 4.5 Complete Order Lifecycle Events

```
order.created (buyer places order)
    ↓
order.confirmed (payment successful)
    ↓ (triggers WMS)
picking.created
    ↓
picking.assigned
    ↓
picking.completed
    ↓ (triggers packing)
packing.started
    ↓
packing.completed
    ↓ (triggers delivery)
delivery.created
    ↓
delivery.in_transit
    ↓
delivery.delivered
    ↓
order.delivered (final status)
```

---

## 5. Payment Events

### 5.1 Stream: `payments:events:transactions`

**Producer:** Payment Service
**Consumers:** Listings, Notifications

### 5.2 Event Types (from Notification Service)

| Event | Description | Channels |
|-------|-------------|----------|
| `payment.initiated` | Payment process started | Email, Telegram |
| `payment.completed` | Payment successful | Email, Telegram, Push |
| `payment.failed` | Payment failed | Email, Telegram |
| `payment.refunded` | Refund processed | Email, Telegram |
| `payment.cod_confirmed` | COD order confirmed | Email, SMS |

### 5.3 Example Payload

```json
{
  "type": "payment.completed",
  "payment_id": "pay_abc123",
  "order_id": 12345,
  "amount": 5990,
  "currency": "RSD",
  "method": "card",
  "timestamp": 1734825600
}
```

---

## 6. Delivery Events

### 6.1 Stream: `delivery:events:tracking`

**Producer:** Delivery Service
**Consumers:** Listings, Notifications

### 6.2 Event Types

| Event | Description | Trigger |
|-------|-------------|---------|
| `delivery.created` | Delivery task created | After packing.completed |
| `delivery.confirmed` | Courier assigned | Courier accepts |
| `delivery.in_transit` | Package in transit | Tracking update |
| `delivery.out_for_delivery` | Out for final delivery | Near destination |
| `delivery.delivered` | Successfully delivered | Proof of delivery |
| `delivery.failed` | Delivery failed | Customer unavailable |
| `delivery.returned` | Returned to sender | Failed delivery |

### 6.3 Tracking Update Example

```json
{
  "type": "delivery.in_transit",
  "shipment_id": "ship_xyz789",
  "order_id": 12345,
  "tracking_number": "POST123456789",
  "status": "in_transit",
  "location": "Sorting Center Belgrade",
  "estimated_delivery": "2025-12-25T10:00:00Z",
  "timestamp": 1734825600
}
```

---

## 7. PVZ Events

> **Статус интеграции:** ✅ ПОЛНОСТЬЮ ИНТЕГРИРОВАНО (2025-12-21)

### 7.1 Stream Overview

| Stream | Direction | Producer | Consumer |
|--------|-----------|----------|----------|
| `pvz.events` | PVZ → Delivery | PVZ Service | Delivery Service |
| `delivery.events` | Delivery → PVZ | Delivery Service | PVZ Service |

**Redis:** Port 36384 (PVZ), Port 36380 (Delivery)

### 7.2 PVZ → Delivery Events (pvz.events)

**Publisher:** `pvz/internal/events/publisher.go`

| Event | Trigger | Payload |
|-------|---------|---------|
| `pvz.parcel.arrived` | Посылка принята в ПВЗ | parcel_id, pvz_id, pvz_code, recipient_id, pickup_code, storage_days, accepted_by |
| `pvz.parcel.picked_up` | Клиент забрал посылку | parcel_id, pvz_id, recipient_id, picked_up_by, pickup_code, storage_fee, days_stored |
| `pvz.parcel.returned` | Посылка возвращена | parcel_id, pvz_id, recipient_id, return_reason, returned_by, days_stored |
| `pvz.handover.accepted` | Курьер передал посылки | handover_id, pvz_id, parcel_ids, courier_id, accepted_by, parcel_count |
| `pvz.handover.rejected` | Приём отклонён | handover_id, pvz_id, parcel_ids, courier_id, rejected_by, reason |
| `pvz.capacity.warning` | Загрузка > 90% | pvz_id, pvz_code, current_capacity, max_capacity, usage_percent, level |
| `pvz.capacity.critical` | Загрузка = 100% | pvz_id, pvz_code, current_capacity, max_capacity, usage_percent, available_slots |
| `pvz.shift.started` | Оператор начал смену | shift_id, pvz_id, operator_id, operator_name |
| `pvz.shift.ended` | Оператор закончил смену | shift_id, pvz_id, operator_id, duration_minutes, parcels_handled |
| `pvz.partner.registered` | Партнёр зарегистрировался | partner_id, user_id, business_name, business_type |
| `pvz.partner.approved` | Партнёр одобрен | partner_id, user_id, pvz_id, approved_by |
| `pvz.partner.rejected` | Партнёр отклонён | partner_id, user_id, rejected_by, reason |

### 7.3 Delivery → PVZ Events (delivery.events)

**Consumer:** `pvz/internal/events/consumer.go`

| Event | Trigger | Handler Action |
|-------|---------|----------------|
| `delivery.parcel.routed_to_pvz` | Посылка назначена на ПВЗ | Создаёт Handover с ExpectedAt |
| `delivery.courier.approaching` | Курьер приближается | Уведомляет оператора |

### 7.4 Event Payloads

**pvz.parcel.arrived:**
```json
{
  "type": "pvz.parcel.arrived",
  "parcel_id": "parcel_12345",
  "pvz_id": 1,
  "pvz_code": "PVZ-BG-001",
  "recipient_id": 890,
  "pickup_code": "123456",
  "storage_days": 3,
  "accepted_by": "operator_456",
  "timestamp": 1734825600
}
```

**pvz.capacity.critical:**
```json
{
  "type": "pvz.capacity.critical",
  "pvz_id": 1,
  "pvz_code": "PVZ-BG-001",
  "current_capacity": 100,
  "max_capacity": 100,
  "usage_percent": 100.0,
  "available_slots": 0,
  "timestamp": 1734825600
}
```

**delivery.parcel.routed_to_pvz:**
```json
{
  "type": "delivery.parcel.routed_to_pvz",
  "parcel_id": "parcel_12345",
  "pvz_id": 1,
  "recipient_id": 890,
  "estimated_arrival": "2025-12-22T10:00:00Z",
  "timestamp": 1734825600
}
```

### 7.5 Integration in main.go

**PVZ Service (Consumer):**
```go
// cmd/server/main.go
handoverEventAdapter := newHandoverEventAdapter(handoverService, parcelRepo, logger)
eventConsumer := events.NewConsumer(redisClient, handoverEventAdapter, logger)
eventConsumer.Start(context.Background())
```

**Delivery Service (Consumer):**
```go
// cmd/server/main.go
if cfg.Events.Enabled {
    eventHandler := events.NewDefaultPVZEventHandler(serviceLogger)
    eventConsumer = events.NewConsumer(redisClient, eventHandler, serviceLogger, cfg.Events.ConsumerName)
    eventConsumer.Start(ctx)
}
```

### 7.6 Communication Diagram

```
PVZ Service                              Delivery Service
     │                                          │
     │◄──── delivery.parcel.routed_to_pvz ──────│
     │      (посылка назначена)                 │
     │                                          │
     │◄──── delivery.courier.approaching ───────│
     │      (курьер приближается)               │
     │                                          │
     │───── pvz.parcel.arrived ────────────────►│
     │      (посылка в ПВЗ)                     │
     │                                          │
     │───── pvz.parcel.picked_up ──────────────►│
     │      (выдано клиенту)                    │
     │                                          │
     │───── pvz.capacity.warning/critical ─────►│
     │      (capacity alert → routing)          │
     │                                          │
```

### 7.7 Consumer Groups

| Stream | Consumer Group | Purpose |
|--------|---------------|---------|
| `pvz.events` | `delivery-pvz-consumers` | Delivery обрабатывает PVZ events |
| `delivery.events` | `pvz-delivery-consumers` | PVZ обрабатывает Delivery events |

---

## 8. Notification Events

### 8.1 Notification Domain Events (from Notification Service)

**Source:** `/p/github.com/vondi-global/notifications/internal/domain/events/types.go`

### 8.2 All Supported Event Types (50+)

#### Orders (8 events)
```go
OrderCreated    EventType = "order.created"
OrderConfirmed  EventType = "order.confirmed"
OrderAccepted   EventType = "order.accepted"
OrderProcessing EventType = "order.processing"
OrderShipped    EventType = "order.shipped"
OrderDelivered  EventType = "order.delivered"
OrderCancelled  EventType = "order.cancelled"
OrderRefunded   EventType = "order.refunded"
```

#### Payments (5 events)
```go
PaymentInitiated EventType = "payment.initiated"
PaymentCompleted EventType = "payment.completed"
PaymentFailed    EventType = "payment.failed"
PaymentRefunded  EventType = "payment.refunded"
PaymentCOD       EventType = "payment.cod_confirmed"
```

#### Delivery (7 events)
```go
DeliveryCreated        EventType = "delivery.created"
DeliveryConfirmed      EventType = "delivery.confirmed"
DeliveryInTransit      EventType = "delivery.in_transit"
DeliveryOutForDelivery EventType = "delivery.out_for_delivery"
DeliveryDelivered      EventType = "delivery.delivered"
DeliveryFailed         EventType = "delivery.failed"
DeliveryReturned       EventType = "delivery.returned"
```

#### Users (7 events)
```go
UserRegistered      EventType = "user.registered"
UserEmailVerified   EventType = "user.email_verified"
UserPhoneVerified   EventType = "user.phone_verified"
UserPasswordChanged EventType = "user.password_changed"
UserPasswordReset   EventType = "user.password_reset"
UserRoleChanged     EventType = "user.role_changed"
UserBanned          EventType = "user.banned"
```

#### Marketplace (8 events)
```go
ListingCreated       EventType = "listing.created"
ListingApproved      EventType = "listing.approved"
ListingRejected      EventType = "listing.rejected"
ListingStatusChanged EventType = "listing.status_changed"
ListingPriceChanged  EventType = "listing.price_changed"
ListingFavorited     EventType = "listing.favorited"
ListingSoldOut       EventType = "listing.sold_out"
```

#### Warehouse (4 events)
```go
WarehouseInvitationCreated  EventType = "warehouse.invitation_created"
WarehouseInvitationAccepted EventType = "warehouse.invitation_accepted"
WarehouseStaffRemoved       EventType = "warehouse.staff_removed"
WarehouseStaffRoleChanged   EventType = "warehouse.staff_role_changed"
```

#### Franchise (7 events)
```go
FranchiseApplicationCreated    EventType = "franchise.application_created"
FranchiseApplicationApproved   EventType = "franchise.application_approved"
FranchiseApplicationRejected   EventType = "franchise.application_rejected"
FranchiseInfoRequested         EventType = "franchise.info_requested"
FranchiseInfoProvided          EventType = "franchise.info_provided"
FranchiseTrainingScheduled     EventType = "franchise.training_scheduled"
FranchiseContractSigned        EventType = "franchise.contract_signed"
```

### 8.3 Event Validation

```go
// Check if event type is valid
if !eventType.IsValid() {
    return errors.New("invalid event type")
}

// Get event category
category := eventType.Category() // "order", "payment", etc.

// Get event action
action := eventType.Action() // "created", "confirmed", etc.
```

---

## 9. User Events

### 9.1 Stream: `auth:events:users`

**Producer:** Auth Service
**Consumers:** Notifications

| Event | Trigger | Notification Channels |
|-------|---------|----------------------|
| `user.registered` | New user signup | Email (verification link) |
| `user.email_verified` | Email verified | Email (welcome) |
| `user.password_reset` | Password reset requested | Email (reset link) |
| `user.role_changed` | Admin changed user role | Email, Telegram |

---

## 10. Event Payload Schemas

### 10.1 Standard Event Envelope

All events должны содержать:

```json
{
  "type": "event.type",
  "timestamp": 1734825600,
  "version": "1.0",
  "payload": {
    // Event-specific data
  }
}
```

### 10.2 WMS Event Schema

```json
{
  "type": "picking.completed",
  "task_id": 123,
  "order_id": 456,
  "warehouse_id": 1,
  "timestamp": 1734825600
}
```

### 10.3 Order Event Schema

```json
{
  "type": "order.confirmed",
  "order_id": 12345,
  "storefront_id": 67,
  "buyer_id": 890,
  "items": "[{\"listing_id\":100,\"quantity\":2,\"warehouse_id\":1}]",
  "total_amount": 5990,
  "currency": "RSD",
  "timestamp": 1734825600
}
```

### 10.4 Payment Event Schema

```json
{
  "type": "payment.completed",
  "payment_id": "pay_abc123",
  "order_id": 12345,
  "amount": 5990,
  "currency": "RSD",
  "method": "card",
  "provider": "allsecure",
  "timestamp": 1734825600
}
```

### 10.5 Notification Event Schema

```json
{
  "type": "order.confirmed",
  "user_id": 890,
  "channels": ["email", "telegram"],
  "locale": "sr",
  "priority": "high",
  "metadata": {
    "order_id": 12345,
    "order_number": "ORD-2025-001",
    "total": "5.990 RSD"
  },
  "timestamp": 1734825600
}
```

---

## 11. Consumer Groups

### 11.1 Consumer Group Overview

Redis Streams поддерживает **consumer groups** для:
- ✅ Гарантированной доставки (каждое сообщение обрабатывается один раз)
- ✅ Load balancing (несколько инстансов сервиса)
- ✅ Fault tolerance (если инстанс упал, сообщение переназначается)

### 11.2 Consumer Group Configuration

| Stream | Consumer Group | Instances |
|--------|---------------|-----------|
| `listings:events:orders` | `wms-consumers` | wms-instance-1, wms-instance-2 |
| `wms:events:packing` | `delivery-consumers` | delivery-instance-1 |
| `pvz.events` | `delivery-pvz-consumers` | delivery-instance-1 |
| `delivery.events` | `pvz-delivery-consumers` | pvz-instance-1 |
| `*:events:*` | `notification-consumers` | notify-instance-1, notify-instance-2 |

### 11.3 Consumer Group Creation

```go
// internal/events/consumer.go
func (c *RedisEventConsumer) ensureConsumerGroup(ctx context.Context) error {
    streams := []string{StreamListingsOrders}

    for _, stream := range streams {
        // Create consumer group (from ID "0" to read from beginning)
        err := c.client.XGroupCreateMkStream(ctx, stream, ConsumerGroupWMS, "0").Err()
        if err != nil && err.Error() != "BUSYGROUP Consumer Group name already exists" {
            return err
        }
    }
    return nil
}
```

### 11.4 Consumer Implementation

```go
// Start consuming
func (c *RedisEventConsumer) consume(ctx context.Context) {
    streams := []string{StreamListingsOrders, ">"}
    blockDuration := 5 * time.Second

    for {
        result, err := c.client.XReadGroup(ctx, &redis.XReadGroupArgs{
            Group:    ConsumerGroupWMS,
            Consumer: c.consumerName, // "wms-instance-1"
            Streams:  streams,
            Count:    10,
            Block:    blockDuration,
        }).Result()

        if err != nil {
            // Handle error
            continue
        }

        for _, stream := range result {
            for _, message := range stream.Messages {
                // Process message
                if err := c.processMessage(ctx, stream.Stream, message); err != nil {
                    c.logger.Error().Err(err).Msg("Failed to process message")
                    continue
                }

                // Acknowledge message
                c.client.XAck(ctx, stream.Stream, ConsumerGroupWMS, message.ID)
            }
        }
    }
}
```

### 11.5 Idempotency Pattern

```go
// Check if already processed (idempotency)
func (h *MyHandler) OnOrderConfirmed(ctx context.Context, orderID int64, ...) error {
    // Check if picking task already exists
    existingTask, _ := h.pickingRepo.GetByOrderID(ctx, orderID)
    if existingTask != nil {
        h.logger.Info().Int64("order_id", orderID).Msg("Already processed, skipping")
        return nil // Idempotent - return success
    }

    // Process event...
    return h.pickingService.CreatePickingTask(ctx, orderID, items)
}
```

---

## 12. Error Handling & Retry

### 12.1 Exponential Backoff

```go
// internal/events/consumer.go
baseBackoff := 1 * time.Second
maxBackoff := 60 * time.Second
currentBackoff := baseBackoff

for {
    result, err := c.client.XReadGroup(ctx, ...).Result()
    if err != nil {
        consecutiveErrors++

        // Exponential backoff
        time.Sleep(currentBackoff)
        currentBackoff = currentBackoff * 2
        if currentBackoff > maxBackoff {
            currentBackoff = maxBackoff
        }
        continue
    }

    // Reset on success
    currentBackoff = baseBackoff
    consecutiveErrors = 0
}
```

### 12.2 Dead Letter Queue Pattern

```go
// After N failed attempts, move to DLQ
const maxRetries = 3

func (c *RedisEventConsumer) processMessage(ctx context.Context, message redis.XMessage) error {
    retryCount := getRetryCount(message)

    err := c.handler.Handle(ctx, message)
    if err != nil {
        if retryCount >= maxRetries {
            // Move to dead letter queue
            c.publishToDeadLetterQueue(message, err)
            // Ack original message
            return nil
        }

        // Increment retry counter
        return err // Will stay in pending
    }

    return nil
}
```

### 12.3 Circuit Breaker

```go
type CircuitBreaker struct {
    maxFailures int
    resetTime   time.Duration
    failures    int
    lastFailure time.Time
    state       string // "closed", "open", "half-open"
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    if cb.state == "open" {
        if time.Since(cb.lastFailure) > cb.resetTime {
            cb.state = "half-open"
        } else {
            return errors.New("circuit breaker open")
        }
    }

    err := fn()
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.maxFailures {
            cb.state = "open"
        }
        return err
    }

    cb.failures = 0
    cb.state = "closed"
    return nil
}
```

---

## 13. Monitoring & Observability

### 13.1 Redis CLI Monitoring

```bash
# Stream length
redis-cli -p 36381 XLEN wms:events:picking

# Consumer group info
redis-cli -p 36380 XINFO GROUPS listings:events:orders

# Pending messages
redis-cli -p 36380 XPENDING listings:events:orders wms-consumers

# Consumer details
redis-cli -p 36380 XINFO CONSUMERS listings:events:orders wms-consumers

# Read last 10 events
redis-cli -p 36381 XRANGE wms:events:picking - + COUNT 10

# Trim old events (keep last 10000)
redis-cli -p 36381 XTRIM wms:events:picking MAXLEN ~ 10000
```

### 13.2 Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|----------------|
| Stream length | Messages in stream | > 10000 |
| Pending count | Unprocessed messages | > 100 |
| Consumer lag | Time since last processed | > 5 minutes |
| Error rate | Failed processing attempts | > 5% |
| Processing time | P95 latency | > 1 second |

### 13.3 Logging Standards

```go
// Event published
logger.Debug().
    Str("stream", "wms:events:picking").
    Str("event_type", "picking.created").
    Int64("task_id", 123).
    Msg("Event published")

// Event consumed
logger.Info().
    Str("stream", "listings:events:orders").
    Str("event_type", "order.confirmed").
    Int64("order_id", 456).
    Msg("Processing order confirmed event")

// Error
logger.Error().
    Err(err).
    Str("stream", stream).
    Str("message_id", msgID).
    Msg("Failed to process message")
```

### 13.4 Prometheus Metrics (example)

```go
var (
    eventsPublished = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "events_published_total",
            Help: "Total number of events published",
        },
        []string{"stream", "event_type"},
    )

    eventsConsumed = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "events_consumed_total",
            Help: "Total number of events consumed",
        },
        []string{"stream", "event_type", "status"},
    )

    processingDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "event_processing_duration_seconds",
            Help: "Event processing duration",
        },
        []string{"stream", "event_type"},
    )
)
```

### 13.5 Health Check

```bash
# Check Redis connection
curl http://localhost:8085/health

# Response
{
  "status": "healthy",
  "redis": {
    "wms_events": "connected",
    "listings_events": "connected"
  },
  "consumers": {
    "wms-consumers": {
      "status": "running",
      "pending": 0,
      "lag_seconds": 0
    }
  }
}
```

---

## Итоги

### Реализовано

✅ **Redis Streams** для всех микросервисов
✅ **Consumer Groups** для надежности
✅ **Idempotency** паттерн
✅ **Exponential Backoff** для ретраев
✅ **Event schemas** стандартизированы
✅ **Мониторинг** через Redis CLI

### Best Practices

1. **Idempotency:** Все event handlers должны быть идемпотентными
2. **Error Handling:** Не fail operation если event publishing failed
3. **Graceful Shutdown:** Всегда останавливать consumers перед выключением
4. **Monitoring:** Мониторить pending messages и consumer lag
5. **Logging:** Логировать все события с достаточным контекстом
6. **Trimming:** Автоматически очищать старые события (XTRIM)

### Производительность

- **Throughput:** Redis Streams >100k msgs/sec
- **Latency:** <1ms publish, ~5ms consume
- **Memory:** Auto-trim at 10k messages
- **Persistence:** RDB/AOF enabled
- **Replication:** Master-replica для failover

---

**Документ обновлён:** 2025-12-21
**Автор:** Senior Full-Stack Architect
**Версия:** 1.0.0
