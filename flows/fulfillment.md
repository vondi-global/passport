# Fulfillment Flow Passport

## Обзор

Fulfillment Flow - это end-to-end процесс обработки заказа в WMS от момента подтверждения заказа до отправки товара клиенту. Процесс полностью автоматизирован и управляется событиями через Redis Streams.

**Основные этапы:**
1. **Order Confirmation** - заказ подтверждён в Listings Service
2. **Inventory Reservation** - резервирование товаров на складе
3. **Picking** - сборка товаров со складских мест
4. **Packing** - упаковка собранных товаров
5. **Shipping** - создание отправления и передача в Delivery Service

**Архитектура:**
- Event-Driven через Redis Streams
- Domain-Driven Design (DDD)
- Microservices: WMS ↔ Listings ↔ Delivery
- Idempotency-safe (повторные события не дублируют задачи)

---

## 1. Picking Workflow

### 1.1 Создание задачи сборки

**Триггер:** `order.confirmed` event от Listings Service

**FulfillmentService.ProcessOrderConfirmed():**

```
1. Проверка idempotency (если задача уже существует → skip)
2. Группировка items по warehouse_id (multi-warehouse support)
3. Для каждого warehouse:
   a. Резервирование inventory (InventoryService.ReserveInventory)
   b. Создание PickingTask через PickingService
   c. Публикация picking.created event
4. Rollback всех reservations при ошибке
```

**PickingService.CreatePickingTask():**

```
1. Валидация входных данных (orderID, warehouseID, items)
2. Создание PickingTask (status: pending)
3. Для каждого item:
   a. Получение оптимальных локаций (FIFO - oldest first)
   b. Проверка наличия достаточного stock
   c. Создание PickingTaskItem для каждой локации
4. Batch insert всех items
5. Обновление task.total_items
```

**Оптимизация FIFO:** Товары выбираются с самых старых локаций первыми для минимизации просроченного товара.

### 1.2 Статусы задачи сборки

```
pending → assigned → in_progress → completed
   ↓                      ↓
cancelled ←──────────────┘
```

**Переходы:**
- `pending → assigned`: Assign(workerID) - назначен работник
- `assigned → in_progress`: Start() - работник начал сборку
- `in_progress → completed`: Complete() - все items собраны
- `any → cancelled`: Cancel(reason) - отмена заказа

**PickingTask fields:**
```go
ID, OrderID, WarehouseID, WorkerID
Status: pending|assigned|in_progress|completed|cancelled
Priority: low|normal|high|urgent
TotalItems, PickedItems (прогресс)
StartedAt, CompletedAt, CancelledAt
Items []PickingTaskItem
```

### 1.3 Picking Items

**PickingTaskItem статусы:**
- `pending` - ожидает сборки
- `picked` - полностью собран
- `short` - частично собран (shortage)
- `skipped` - пропущен

**PickingService.PickItem():**

```
1. Валидация task.status == in_progress
2. Получение PickingTaskItem
3. Проверка quantity_picked <= remaining
4. Обновление item.status (picked или short)
5. Deduct inventory из product_location
6. Обновление task.picked_items count
```

**GetNextItem():** Возвращает следующий pending item по sequence (оптимальный маршрут).

### 1.4 Завершение сборки

**CompletePickingTask():**

```
1. Проверка: все items IsComplete() (picked/short/skipped)
2. Проверка: task.CanComplete() (progress 100%)
3. task.Complete() → status: completed
4. FulfillmentService.OnPickingCompleted() (callback)
   → Автоматическое создание PackingTask
```

**Event:** `picking.completed` (order_id, task_id)

---

## 2. Packing Workflow

### 2.1 Создание задачи упаковки

**Триггер:** PickingTask.completed (callback от FulfillmentService)

**FulfillmentService.OnPickingCompleted():**

```
1. Проверка picking_task.status == completed
2. Проверка idempotency (если packing task существует → skip)
3. Создание PackingTask через PackingService
4. Публикация packing.started event
```

**PackingService.CreatePackingTaskFromPicking():**

```
1. Получение PickingTask и валидация status
2. Создание PackingTask:
   - order_id, picking_task_id, warehouse_id
   - status: pending
   - priority (наследуется или default: normal)
3. Insert в БД
```

### 2.2 Статусы задачи упаковки

```
pending → in_progress → packed → shipped
   ↓            ↓          ↓
cancelled ←─────┴──────────┘
```

**Переходы:**
- `pending → in_progress`: Start(workerID) - работник начал упаковку
- `in_progress → packed`: Pack() - упаковка завершена (dimensions set)
- `packed → shipped`: Ship(tracking_number) - отправлено
- `any → cancelled`: Cancel(reason) - отмена

**PackingTask fields:**
```go
ID, OrderID, PickingTaskID, WarehouseID, WorkerID
Status: pending|in_progress|packed|shipped|cancelled
Priority: low|normal|high|urgent
PackagingType: box|bag|envelope|pallet|custom
Weight, Length, Width, Height (dimensions)
TrackingNumber, LabelURL
StartedAt, PackedAt, ShippedAt
```

### 2.3 Процесс упаковки

**1. Начало упаковки:**
```go
StartPackingTask(taskID, workerID)
→ task.status = in_progress
→ task.started_at = now
```

**2. Установка габаритов:**
```go
SetPackageDetails(taskID, weight, length, width, height, packagingType)
→ task.dimensions = {...}
→ task.packaging_type = box
```

**3. Завершение упаковки:**
```go
CompletePackingTask(taskID)
→ Проверка: dimensions set?
→ task.status = packed
→ FulfillmentService.OnPackingCompleted() (callback)
```

**Volumetric Weight Calculation:**
```go
GetVolumetricWeight() = (L × W × H) / 5000
GetChargeableWeight() = max(actual_weight, volumetric_weight)
```

### 2.4 Multi-Package Support

**Package entity:** Один PackingTask может содержать несколько packages (multi-box shipment).

```go
type Package struct {
    ID, PackingTaskID, SequenceNumber
    PackagingType, Weight, Length, Width, Height
    Barcode, LabelURL
}
```

**AddPackage():** Добавить коробку в shipment (sequence auto-increment).

---

## 3. Shipping Workflow

### 3.1 Создание отправления

**Триггер:** PackingTask.status = packed (callback от FulfillmentService)

**FulfillmentService.OnPackingCompleted():**

```
1. Проверка packing_task.status == packed
2. Получение delivery address (DeliveryClient.GetDeliveryAddressByOrder)
3. Создание shipment через DeliveryClient.CreateShipment:
   - provider (PostExpress, D Express, ACS, etc.)
   - from_address (warehouse address)
   - to_address (customer address)
   - package_info (weight, dimensions)
4. Получение tracking_number и shipment_id
5. Обновление packing_task.tracking_number
6. Обновление order status → "ready_for_shipment" (Listings Service)
7. Публикация shipment.created event
```

### 3.2 Delivery Providers

**Поддерживаемые провайдеры:**
- PostExpress (Serbia)
- D Express (Serbia)
- ACS (Greece)
- Bex (Serbia)
- YunExpress (China)

**DeliveryClient.CreateShipment() request:**
```go
{
  order_id, warehouse_id, provider
  from_address: { street, city, postal_code, country, contact_name, phone }
  to_address: { street, city, postal_code, country, contact_name, phone }
  package: { weight, length, width, height, description }
  user_id
}
```

**Response:**
```go
{
  id (shipment_id)
  tracking_number
  label_url (если поддерживается)
  estimated_delivery_date
}
```

### 3.3 Label Generation

**PrintLabel(taskID):**

```
1. Проверка status == packed || shipped
2. Генерация ZPL label data (для thermal printers)
3. Возврат label_url и labelData
```

**TODO:** Интеграция с реальными API провайдеров для label generation.

---

## 4. Статусы и переходы

### 4.1 State Machine - Picking

```
┌──────────┐  Assign   ┌──────────┐  Start  ┌─────────────┐
│ pending  │ ────────→ │ assigned │ ──────→ │ in_progress │
└──────────┘           └──────────┘         └─────────────┘
     │                      │                       │
     │                      │                       │ Complete
     │                      │                       ↓
     │                      │               ┌───────────┐
     │                      │               │ completed │
     │                      │               └───────────┘
     │                      │                       │
     └──────────────────────┴───────────────────────┤
                        Cancel                      │
                            ↓                       │
                     ┌───────────┐                 │
                     │ cancelled │←────────────────┘
                     └───────────┘
```

**Бизнес-правила:**
- Assign: только pending
- Start: только assigned (worker_id required)
- Complete: только in_progress + all items complete
- Cancel: любой статус кроме completed

### 4.2 State Machine - Packing

```
┌──────────┐  Start  ┌─────────────┐  Pack  ┌────────┐  Ship  ┌─────────┐
│ pending  │ ──────→ │ in_progress │ ─────→ │ packed │ ─────→ │ shipped │
└──────────┘         └─────────────┘        └────────┘        └─────────┘
     │                      │                    │
     │                      │                    │
     └──────────────────────┴────────────────────┤
                        Cancel                   │
                            ↓                    │
                     ┌───────────┐              │
                     │ cancelled │←─────────────┘
                     └───────────┘
```

**Бизнес-правила:**
- Start: только pending (worker_id required)
- Pack: только in_progress + dimensions set
- Ship: только packed (tracking_number required)
- Cancel: любой статус кроме shipped

### 4.3 Item-Level Statuses

**PickingTaskItem:**
- `pending` → `picked`: PickItem(full quantity)
- `pending` → `short`: PartialPick(quantity < required)
- `pending` → `skipped`: Skip() - товар отсутствует

**PackingTaskItem:**
- `pending` → `fully_packed`: PackQuantity(remaining = 0)
- Прогресс: packed_quantity / quantity × 100%

---

## 5. WMS Events (Redis Streams)

### 5.1 Streams

**Публикуемые WMS:**
- `wms:events:picking` - события сборки
- `wms:events:packing` - события упаковки
- `wms:events:shipment` - события отправки
- `wms:events:inventory` - события инвентаря

**Потребляемые WMS:**
- `listings:events:orders` - события заказов (order.confirmed, order.cancelled)

### 5.2 Event Types

**Picking:**
```yaml
picking.created:
  task_id, order_id, warehouse_id

picking.assigned:
  task_id, order_id, worker_id

picking.completed:
  task_id, order_id
```

**Packing:**
```yaml
packing.started:
  task_id, order_id

packing.completed:
  task_id, order_id
```

**Shipment:**
```yaml
shipment.created:
  shipment_id, order_id, tracking_number
```

**Inventory:**
```yaml
inventory.reserved:
  listing_id, warehouse_id, quantity, order_id

inventory.released:
  listing_id, warehouse_id, quantity, reason

inventory.placed:
  listing_id, warehouse_id, quantity, location

inventory.moved:
  listing_id, warehouse_id, from_location, to_location
```

### 5.3 Event Publisher

```go
type EventPublisher interface {
    PublishPickingCreated(ctx, taskID, orderID, warehouseID) error
    PublishPickingCompleted(ctx, taskID, orderID) error
    PublishPackingStarted(ctx, taskID, orderID) error
    PublishPackingCompleted(ctx, taskID, orderID) error
    PublishShipmentCreated(ctx, shipmentID, orderID, trackingNumber) error
    PublishInventoryReserved(ctx, listingID, warehouseID, quantity, orderID) error
}
```

**Redis XAdd:**
```go
client.XAdd(ctx, &redis.XAddArgs{
    Stream: "wms:events:picking",
    Values: {
        "type": "picking.created",
        "task_id": 123,
        "order_id": 456,
        "warehouse_id": 1,
        "timestamp": 1700000000
    }
})
```

### 5.4 Event Consumer

**Consumer Group:** `wms-consumers`

**EventHandler interface:**
```go
type EventHandler interface {
    OnOrderConfirmed(ctx, orderID, storefrontID, items) error
    OnOrderCancelled(ctx, orderID, reason) error
}
```

**FulfillmentEventHandler** адаптирует FulfillmentService к EventHandler interface.

---

## 6. Интеграция с Monolith

### 6.1 Listings Service Integration

**1. Order Lifecycle:**
```
Listings:
  confirmed → WMS: ProcessOrderConfirmed()
  cancelled → WMS: ProcessOrderCancelled()

WMS:
  packing.completed → Listings: UpdateOrderStatus("ready_for_shipment")
```

**2. ListingsClient (HTTP):**

```go
type ListingsClient interface {
    GetOrder(ctx, orderID) (*Order, error)
    UpdateOrderStatus(ctx, orderID, status, message) error
}
```

**Endpoints:**
- `GET /api/v1/orders/{id}` - получить детали заказа
- `POST /api/v1/orders/{id}/status` - обновить статус

**Order struct:**
```go
type Order struct {
    ID, OrderNumber, StorefrontID
    Status: pending|confirmed|processing|shipped|delivered|cancelled
    Items: []OrderItem { listing_id, quantity, price }
}
```

### 6.2 Delivery Service Integration

**DeliveryClient (HTTP):**

```go
type DeliveryClient interface {
    GetDeliveryAddressByOrder(ctx, orderID) (*DeliveryAddress, error)
    CreateShipment(ctx, *CreateShipmentRequest) (*Shipment, error)
}
```

**Endpoints:**
- `GET /api/v1/delivery/addresses/order/{id}` - адрес доставки
- `POST /api/v1/shipments` - создать отправление

**CreateShipmentRequest:**
```go
{
  order_id, warehouse_id, provider
  from_address, to_address
  package: { weight, dimensions, description }
  user_id
}
```

**Shipment response:**
```go
{
  id, tracking_number, label_url
  provider, status, estimated_delivery
}
```

### 6.3 Event Flow Diagram

```
┌──────────────┐  order.confirmed  ┌─────────────────┐
│   Listings   │ ─────────────────→│ WMS Fulfillment │
│   Service    │                   │     Service     │
└──────────────┘                   └─────────────────┘
       ↑                                    │
       │ UpdateOrderStatus                 │ picking.created
       │ ("ready_for_shipment")            ↓
       │                            ┌──────────────┐
       │                            │   Picking    │
       │                            │   Service    │
       │                            └──────────────┘
       │                                    │ picking.completed
       │                                    ↓
       │                            ┌──────────────┐
       │                            │   Packing    │
       │                            │   Service    │
       │                            └──────────────┘
       │                                    │ packing.completed
       │                                    ↓
       └────────────────────────────┬──────────────┐
                                    │ CreateShipment
                                    ↓
                            ┌────────────────┐  shipment.created
                            │    Delivery    │ ──────────────────→
                            │    Service     │
                            └────────────────┘
```

---

## 7. Sequence Diagrams

### 7.1 Full Fulfillment Flow

```
┌─────────┐  ┌──────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────┐
│Listings │  │Fulfillment│ │Inventory│ │Picking │ │Packing │ │ Delivery │
└────┬────┘  └─────┬────┘  └────┬───┘  └───┬────┘  └───┬────┘  └────┬─────┘
     │             │            │          │           │            │
     │ order.     │            │          │           │            │
     │ confirmed  │            │          │           │            │
     ├───────────→│            │          │           │            │
     │            │ Reserve   │          │           │            │
     │            │ Inventory │          │           │            │
     │            ├──────────→│          │           │            │
     │            │ ←─────────┤ (OK)     │           │            │
     │            │ Create    │          │           │            │
     │            │ Picking   │          │           │            │
     │            ├──────────────────────→│           │            │
     │            │ ←─────────────────────┤ (Task)    │            │
     │            │ picking.created       │           │            │
     │            ├───────────────────────→ (Event)   │            │
     │            │                       │           │            │
     │            │          ... Worker picks items ...│            │
     │            │                       │           │            │
     │            │ picking.completed     │           │            │
     │            │←──────────────────────┤           │            │
     │            │ Create                │           │            │
     │            │ Packing               │           │            │
     │            ├───────────────────────────────────→│            │
     │            │ ←──────────────────────────────────┤ (Task)     │
     │            │ packing.started                    │            │
     │            ├────────────────────────────────────→ (Event)    │
     │            │                                    │            │
     │            │          ... Worker packs order ...│            │
     │            │                                    │            │
     │            │ packing.completed                  │            │
     │            │←───────────────────────────────────┤            │
     │            │ Create Shipment                               │
     │            ├───────────────────────────────────────────────→│
     │            │ ←──────────────────────────────────────────────┤
     │            │ (tracking_number)                              │
     │            │ shipment.created                               │
     │            ├────────────────────────────────────────────────→
     │ Update     │                                                │
     │ Status     │                                                │
     │←───────────┤                                                │
     │            │                                                │
```

### 7.2 Order Cancellation Flow

```
┌─────────┐  ┌──────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│Listings │  │Fulfillment│ │Inventory│ │Picking │ │Packing │
└────┬────┘  └─────┬────┘  └────┬───┘  └───┬────┘  └───┬────┘
     │             │            │          │           │
     │ order.     │            │          │           │
     │ cancelled  │            │          │           │
     ├───────────→│            │          │           │
     │            │ List       │          │           │
     │            │ Reservations│          │           │
     │            ├──────────→ │          │           │
     │            │←───────────┤          │           │
     │            │ Release    │          │           │
     │            │ Each       │          │           │
     │            ├──────────→ │          │           │
     │            │←───────────┤ (OK)     │           │
     │            │            │          │           │
     │            │ Cancel Picking        │           │
     │            ├──────────────────────→│           │
     │            │←───────────────────────┤ (OK)      │
     │            │            │          │           │
     │            │ Cancel Packing                    │
     │            ├───────────────────────────────────→│
     │            │←────────────────────────────────────┤
     │            │            │          │           │
```

---

## 8. API Endpoints (gRPC)

### 8.1 Picking Methods

**PickingHandler:**

```protobuf
service WarehouseService {
  // Создать задачу сборки
  rpc CreatePickingTask(CreatePickingTaskRequest) returns (PickingTask);

  // Получить задачу
  rpc GetPickingTask(GetPickingTaskRequest) returns (PickingTask);

  // Получить задачу по order_id
  rpc GetPickingTaskByOrderID(GetPickingTaskByOrderIDRequest) returns (PickingTask);

  // Назначить работника
  rpc AssignPickingTask(AssignPickingTaskRequest) returns (PickingTask);

  // Начать сборку
  rpc StartPickingTask(StartPickingTaskRequest) returns (PickingTask);

  // Собрать item
  rpc PickItem(PickItemRequest) returns (PickItemResponse);

  // Получить следующий item для сборки
  rpc GetNextPickingItem(GetNextPickingItemRequest) returns (PickingTaskItem);

  // Завершить сборку
  rpc CompletePickingTask(CompletePickingTaskRequest) returns (PickingTask);

  // Отменить сборку
  rpc CancelPickingTask(CancelPickingTaskRequest) returns (PickingTask);

  // Список задач
  rpc ListPickingTasks(ListPickingTasksRequest) returns (ListPickingTasksResponse);
}
```

**Пример request/response:**

```protobuf
message CreatePickingTaskRequest {
  int64 order_id = 1;
  int64 warehouse_id = 2;
  Priority priority = 3; // LOW, NORMAL, HIGH, URGENT
  repeated PickingItem items = 4;
}

message PickingItem {
  int64 listing_id = 1;
  int32 quantity = 2;
}

message PickingTask {
  int64 id = 1;
  int64 order_id = 2;
  int64 warehouse_id = 3;
  int64 worker_id = 4;
  PickingTaskStatus status = 5; // PENDING, ASSIGNED, IN_PROGRESS, COMPLETED, CANCELLED
  Priority priority = 6;
  int32 total_items = 7;
  int32 picked_items = 8;
  google.protobuf.Timestamp started_at = 9;
  google.protobuf.Timestamp completed_at = 10;
  repeated PickingTaskItem items = 11;
}
```

### 8.2 Packing Methods

**PackingHandler:**

```protobuf
service WarehouseService {
  // Создать задачу упаковки
  rpc CreatePackingTask(CreatePackingTaskRequest) returns (PackingTask);

  // Получить задачу
  rpc GetPackingTask(GetPackingTaskRequest) returns (PackingTask);

  // Начать упаковку
  rpc StartPackingTask(StartPackingTaskRequest) returns (PackingTask);

  // Установить габариты
  rpc SetPackageDetails(SetPackageDetailsRequest) returns (PackingTask);

  // Завершить упаковку
  rpc CompletePackingTask(CompletePackingTaskRequest) returns (PackingTask);

  // Отменить упаковку
  rpc CancelPackingTask(CancelPackingTaskRequest) returns (PackingTask);

  // Распечатать label
  rpc PrintLabel(PrintLabelRequest) returns (PrintLabelResponse);

  // Список задач
  rpc ListPackingTasks(ListPackingTasksRequest) returns (ListPackingTasksResponse);
}
```

**Пример request/response:**

```protobuf
message SetPackageDetailsRequest {
  int64 task_id = 1;
  double weight = 2; // kg
  double length = 3; // cm
  double width = 4;  // cm
  double height = 5; // cm
  PackagingType packaging_type = 6; // BOX, BAG, ENVELOPE, PALLET, CUSTOM
}

message PackingTask {
  int64 id = 1;
  int64 order_id = 2;
  int64 picking_task_id = 3;
  int64 warehouse_id = 4;
  int64 worker_id = 5;
  PackingTaskStatus status = 6; // PENDING, IN_PROGRESS, PACKED, SHIPPED, CANCELLED
  Priority priority = 7;
  PackagingType packaging_type = 8;
  double weight = 9;
  double length = 10;
  double width = 11;
  double height = 12;
  string tracking_number = 13;
  string label_url = 14;
  google.protobuf.Timestamp packed_at = 15;
  repeated Package packages = 16;
}
```

### 8.3 Inventory Methods

**InventoryHandler:**

```protobuf
service WarehouseService {
  // Разместить товар на складе
  rpc PlaceInventory(PlaceInventoryRequest) returns (ProductLocation);

  // Переместить товар
  rpc MoveInventory(MoveInventoryRequest) returns (ProductLocation);

  // Зарезервировать товар
  rpc ReserveInventory(ReserveInventoryRequest) returns (Reservation);

  // Освободить резерв
  rpc ReleaseReservation(ReleaseReservationRequest) returns (google.protobuf.Empty);

  // Получить локации для picking (FIFO)
  rpc GetPickingLocations(GetPickingLocationsRequest) returns (PickingLocationsResponse);

  // Убрать товар с локации (deduct)
  rpc RemoveInventory(RemoveInventoryRequest) returns (ProductLocation);
}
```

---

## 9. Database Schema

### 9.1 Picking Tables

**picking_tasks:**
```sql
CREATE TABLE picking_tasks (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    warehouse_id BIGINT NOT NULL,
    worker_id BIGINT,
    status VARCHAR(50) NOT NULL, -- pending, assigned, in_progress, completed, cancelled
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
    total_items INTEGER NOT NULL DEFAULT 0,
    picked_items INTEGER NOT NULL DEFAULT 0,
    notes TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ,
    cancellation_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_picking_tasks_order_id ON picking_tasks(order_id);
CREATE INDEX idx_picking_tasks_status ON picking_tasks(status);
CREATE INDEX idx_picking_tasks_worker_id ON picking_tasks(worker_id);
```

**picking_task_items:**
```sql
CREATE TABLE picking_task_items (
    id BIGSERIAL PRIMARY KEY,
    task_id BIGINT NOT NULL REFERENCES picking_tasks(id),
    listing_id BIGINT NOT NULL,
    location_id BIGINT NOT NULL REFERENCES locations(id),
    product_location_id BIGINT NOT NULL REFERENCES product_locations(id),
    quantity_required INTEGER NOT NULL,
    quantity_picked INTEGER NOT NULL DEFAULT 0,
    status VARCHAR(50) NOT NULL DEFAULT 'pending', -- pending, picked, short, skipped
    sequence INTEGER NOT NULL,
    picked_at TIMESTAMPTZ,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_picking_task_items_task_id ON picking_task_items(task_id);
CREATE INDEX idx_picking_task_items_status ON picking_task_items(status);
CREATE INDEX idx_picking_task_items_sequence ON picking_task_items(task_id, sequence);
```

### 9.2 Packing Tables

**packing_tasks:**
```sql
CREATE TABLE packing_tasks (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    picking_task_id BIGINT REFERENCES picking_tasks(id),
    warehouse_id BIGINT NOT NULL,
    worker_id BIGINT,
    status VARCHAR(50) NOT NULL, -- pending, in_progress, packed, shipped, cancelled
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
    packaging_type VARCHAR(50),
    weight DECIMAL(10,2), -- kg
    length DECIMAL(10,2), -- cm
    width DECIMAL(10,2),  -- cm
    height DECIMAL(10,2), -- cm
    tracking_number VARCHAR(255),
    label_url TEXT,
    notes TEXT,
    started_at TIMESTAMPTZ,
    packed_at TIMESTAMPTZ,
    shipped_at TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ,
    cancellation_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_packing_tasks_order_id ON packing_tasks(order_id);
CREATE INDEX idx_packing_tasks_status ON packing_tasks(status);
CREATE INDEX idx_packing_tasks_worker_id ON packing_tasks(worker_id);
```

**packages:**
```sql
CREATE TABLE packages (
    id BIGSERIAL PRIMARY KEY,
    packing_task_id BIGINT NOT NULL REFERENCES packing_tasks(id),
    sequence_number INTEGER NOT NULL,
    packaging_type VARCHAR(50) NOT NULL,
    weight DECIMAL(10,2),
    length DECIMAL(10,2),
    width DECIMAL(10,2),
    height DECIMAL(10,2),
    barcode VARCHAR(255),
    label_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_packages_packing_task_id ON packages(packing_task_id);
```

### 9.3 Reservation Table

**wms_reservations:**
```sql
CREATE TABLE wms_reservations (
    id BIGSERIAL PRIMARY KEY,
    listing_id BIGINT NOT NULL,
    warehouse_id BIGINT NOT NULL,
    order_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending', -- pending, fulfilled, cancelled
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_wms_reservations_order_id ON wms_reservations(order_id);
CREATE INDEX idx_wms_reservations_status ON wms_reservations(status);
CREATE INDEX idx_wms_reservations_listing_id ON wms_reservations(listing_id, warehouse_id);
```

---

## 10. Error Handling & Idempotency

### 10.1 Idempotency Keys

**Цель:** Предотвратить дублирование задач при повторной доставке events.

**Реализация:**

```go
// ProcessOrderConfirmed - idempotent
existingTasks, _ := pickingTaskRepo.List(ctx, PickingTaskFilter{
    OrderID: orderID,
    Limit: 1,
})
if len(existingTasks) > 0 {
    log.Info("picking task already exists, skipping")
    return nil // idempotent skip
}
```

**Альтернатива:** idempotency_keys таблица (будущее улучшение).

### 10.2 Rollback Strategy

**Inventory Reservation Rollback:**

```go
func (s *FulfillmentService) ProcessOrderConfirmed(...) error {
    var reservations []*domain.WMSReservation

    for _, item := range items {
        reservation, err := inventorySvc.ReserveInventory(...)
        if err != nil {
            // Rollback all previous reservations
            s.rollbackReservations(ctx, reservations)
            return err
        }
        reservations = append(reservations, reservation)
    }

    // Create picking task...
    if err != nil {
        s.rollbackReservations(ctx, reservations)
        return err
    }
}
```

### 10.3 Error Types

**Domain errors:**
- `ValidationError` - неверные входные данные (400)
- `NotFoundError` - ресурс не найден (404)
- `ConflictError` - невозможный переход статуса (409)
- `UnauthorizedError` - нет прав (403)

**Пример:**
```go
if !task.CanStart() {
    return domain.NewConflictError(
        fmt.Sprintf("cannot start task with status %s", task.Status))
}
```

### 10.4 Retry & Circuit Breaker

**TODO:** Добавить retry logic для external service calls (Listings, Delivery).

**Circuit Breaker pattern:**
- После N ошибок → open circuit
- Fallback: skip shipment creation, log error
- Health check: периодическая проверка

---

## 11. Performance & Optimization

### 11.1 FIFO Picking Optimization

**GetPickingLocations():**
```sql
SELECT * FROM product_locations
WHERE warehouse_id = $1 AND listing_id = $2
  AND available_quantity > 0
ORDER BY received_at ASC, id ASC
LIMIT 10
```

**Цель:** Минимизировать просроченный товар, выбирая oldest stock первым.

### 11.2 Batch Operations

**Batch Insert Items:**
```go
itemRepo.CreateBatch(ctx, []PickingTaskItem{...})
// vs
for _, item := range items {
    itemRepo.Create(ctx, item) // N queries - SLOW
}
```

**N+1 Query Prevention:**
```go
// Load all items for task
items, _ := itemRepo.GetByTaskID(ctx, taskID)
task.Items = items
```

### 11.3 Caching Strategy

**Redis Cache:** (future)
- Active picking tasks by worker_id
- Order → PickingTask mapping
- Warehouse locations metadata

**TTL:** 5 minutes для hot data.

---

## 12. Monitoring & Metrics

### 12.1 Key Metrics

**Picking Performance:**
- `picking_task_duration_seconds` - время от created до completed
- `picking_items_per_hour` - производительность работника
- `picking_shortage_rate` - % short picks

**Packing Performance:**
- `packing_task_duration_seconds`
- `packing_tasks_per_hour`
- `average_package_weight`

**Fulfillment E2E:**
- `order_to_shipment_duration_seconds` - от order.confirmed до shipment.created
- `fulfillment_success_rate` - % успешных fulfillments

### 12.2 Logging

**Structured Logging (zerolog):**

```go
log.Info().
    Int64("order_id", orderID).
    Int64("picking_task_id", taskID).
    Int("items_count", len(items)).
    Msg("picking task created")
```

**Log Levels:**
- `DEBUG` - event publishing, item picks
- `INFO` - task creation, status changes
- `WARN` - external service failures (non-critical)
- `ERROR` - critical failures, rollbacks

### 12.3 Alerts

**Critical Alerts:**
- Picking task stuck in `in_progress` > 2 hours
- Packing task без dimensions > 1 hour
- Reservation expired but not released
- External service (Listings/Delivery) unavailable > 5 min

---

## 13. Future Enhancements

### 13.1 Batch Picking

**Цель:** Собирать несколько заказов одновременно.

**BatchPickingTask:**
```go
type BatchPickingTask struct {
    ID, WarehouseID, WorkerID
    OrderIDs []int64 // multiple orders
    Status: pending|in_progress|completed
    Items []BatchPickingItem
}
```

**Benefit:** Снижение времени на picking для multiple small orders.

### 13.2 Quality Control (QC)

**QCTask после Picking:**
```go
picking.completed → QC Task → packing.started
```

**Checklist:**
- Product condition check
- Quantity verification
- Photo capture (damaged items)

### 13.3 Returns Processing

**ReturnTask:**
```go
shipment.returned → ReturnTask → Inventory Placement
```

**Workflow:**
- Receive return
- Inspect quality
- Restock (good) or dispose (damaged)

### 13.4 Wave Picking

**Цель:** Группировать orders по zones/locations для эффективного picking.

**Wave:**
```go
type Wave struct {
    ID, WarehouseID
    ZoneID, Priority
    Orders []int64
    Status: pending|in_progress|completed
}
```

### 13.5 Real-Time Tracking

**WebSocket updates:**
- Picking progress для order detail page
- Worker location tracking (zone changes)
- Live dashboard: active tasks, avg duration

---

## Итоговая блок-схема

```
┌──────────────────────────────────────────────────────────────────────┐
│                         FULFILLMENT FLOW                             │
└──────────────────────────────────────────────────────────────────────┘

  Listings Service                  WMS Service                Delivery Service
       │                                │                            │
       │  order.confirmed               │                            │
       ├───────────────────────────────→│                            │
       │                                │                            │
       │                        Reserve Inventory                    │
       │                                ├────────┐                   │
       │                                │        │                   │
       │                                ←────────┘                   │
       │                                │                            │
       │                        Create PickingTask                   │
       │                                ├────────┐                   │
       │                                │        │                   │
       │                                ←────────┘                   │
       │                                │                            │
       │                        picking.created (event)              │
       │                                ├──────────────→             │
       │                                │                            │
       │                        ... Worker picks items ...           │
       │                                │                            │
       │                        picking.completed (event)            │
       │                                ├──────────────→             │
       │                                │                            │
       │                        Create PackingTask                   │
       │                                ├────────┐                   │
       │                                │        │                   │
       │                                ←────────┘                   │
       │                                │                            │
       │                        packing.started (event)              │
       │                                ├──────────────→             │
       │                                │                            │
       │                        ... Worker packs order ...           │
       │                                │                            │
       │                        packing.completed (event)            │
       │                                ├──────────────→             │
       │                                │                            │
       │                                │  CreateShipment            │
       │                                ├───────────────────────────→│
       │                                │                            │
       │                                │  ←─────────────────────────┤
       │                                │  (tracking_number)         │
       │                                │                            │
       │                        shipment.created (event)             │
       │                                ├──────────────→             │
       │                                │                            │
       │  UpdateOrderStatus             │                            │
       │  ("ready_for_shipment")        │                            │
       ←────────────────────────────────┤                            │
       │                                │                            │
```

---

**Версия:** 1.0
**Последнее обновление:** 2025-12-21
**Автор:** WMS Team
**Статус:** Production-Ready
