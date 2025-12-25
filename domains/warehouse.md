# Domain Passport: Warehouse (WMS)

> **Версия:** 1.5.0
> **Создан:** 2025-12-21
> **Статус:** Production Ready

---

## Обзор домена

**Warehouse Management System (WMS)** - микросервис управления складскими операциями для платформы Vondi.

### Основные функции

1. **Управление складами** - создание, конфигурация складов, зон, локаций
2. **Управление запасами** - размещение, перемещение, резервирование товаров
3. **Picking Workflow** - сборка заказов по оптимальным маршрутам (FIFO)
4. **Packing Workflow** - упаковка, расчет габаритов, печать этикеток
5. **Quality Control** - контроль качества товаров
6. **Returns Processing** - обработка возвратов с фото-инспекцией
7. **Gamification** - система уровней и очков для мотивации работников
8. **Event-Driven Integration** - интеграция с Listings/Delivery через Redis Streams

### Архитектура

- **Паттерн:** Domain-Driven Design (DDD)
- **Стиль:** Event-Driven Microservice
- **Протокол:** gRPC (50055), HTTP (8085 - health/metrics)
- **База данных:** PostgreSQL (35435)
- **Кеш/события:** Redis (36381 - WMS), Redis (36380 - Listings)

---

## Доменные сущности (Entities)

### 1. Warehouse (Склад)

Физический склад с зонами и локациями.

```go
type Warehouse struct {
    ID, Code, Name, Type, Address, City, Country
    Latitude, Longitude, TotalArea *float64
    ContactName, ContactPhone, ContactEmail *string
    WorkingHours map[string]any
    IsActive bool
    Zones []WarehouseZone, ZonesCount int
}
```

**Типы:** main, regional, partner, dark_store
**Правила:** Code уникальный, координаты (-90..90, -180..180), площадь > 0

### 2. WarehouseZone (Зона склада)

```go
type WarehouseZone struct {
    ID, WarehouseID int64
    Code, Name, Type string // receiving, storage, picking, packing, shipping, returns
    Capacity *int, TemperatureMin, TemperatureMax *float64
    IsActive bool, Locations []StorageLocation
}
```

**Типы:** receiving, storage, picking, packing, shipping, returns

### 3. StorageLocation (Локация хранения)

```go
type StorageLocation struct {
    ID, ZoneID int64
    Code string // A1-01-01 (aisle-rack-shelf)
    Aisle, Rack, Shelf, Bin *string
    Type, Status string // Type: shelf/pallet/floor/bin, Status: available/occupied/reserved/maintenance
    Width, Height, Depth, MaxWeight *float64
}
```

### 4. ProductLocation (Размещение товара)

```go
type ProductLocation struct {
    ID, ListingID, LocationID, WarehouseID int64
    BatchID *int64
    Quantity, ReservedQuantity, AvailableQuantity int
    PlacedAt time.Time // FIFO: ORDER BY PlacedAt ASC
}
```

### 5. ProductBatch (Партия товара)

```go
type ProductBatch struct {
    ID, ListingID, WarehouseID int64
    BatchNumber string // LOT-2024-001
    ManufacturedAt, ExpiresAt *time.Time // FEFO: ORDER BY ExpiresAt ASC
    InitialQuantity, CurrentQuantity int
}
```

### 6. WMSReservation (Резервирование)

```go
type WMSReservation struct {
    ID, OrderID, ListingID, ProductLocationID int64
    Quantity int
    Status string // active, committed, released, expired
    ExpiresAt *time.Time // TTL: 30 min (auto-release)
}
```

### 7. Worker (Работник склада)

```go
type Worker struct {
    ID int64
    WorkerCode, Name string
    Email, Phone *string
    Role string // picker, packer, manager
    PINHash, Status string // Status: active/inactive/on_break
    AssignedWarehouse *int64
}
```

### 8. PickingTask (Задача сборки)

```go
type PickingTask struct {
    ID, OrderID, WarehouseID int64
    WorkerID *int64
    Status string // pending→assigned→in_progress→completed (или cancelled)
    Priority string // low, normal, high, urgent
    TotalItems, PickedItems int
    StartedAt, CompletedAt, CancelledAt *time.Time
    Items []PickingTaskItem
}
// Методы: Assign(workerID), Start(), Complete(), Cancel(reason), GetProgress()
```

### 9. PickingTaskItem (Позиция сборки)

```go
type PickingTaskItem struct {
    ID, TaskID, ListingID, LocationID, ProductLocationID int64
    QuantityRequired, QuantityPicked int
    Status string // pending, picked, short, skipped
    Sequence int // Порядок FIFO
    LocationCode string // A1-01-01
}
```

**Статусы:** pending, picked, short (shortage), skipped
**Методы:** Pick(qty), PartialPick(qty), MarkShort(), Skip(), RemainingQuantity(), GetFulfillmentRate()

### 10. PackingTask (Задача упаковки)

```go
type PackingTask struct {
    ID, OrderID, WarehouseID int64
    PickingTaskID, WorkerID *int64
    Status string // pending→in_progress→packed→shipped (или cancelled)
    PackagingType string // box, bag, envelope, pallet, custom
    Weight, Length, Width, Height *float64
    TrackingNumber, LabelURL *string
    Items []PackingTaskItem, Packages []Package // Multi-box
}
// Методы: Start(), Pack(), Ship(tracking), GetVolume(), GetVolumetricWeight() = volume/5000
```

### 11. Package (Посылка)

```go
type Package struct {
    ID, PackingTaskID int64
    SequenceNumber int // 1, 2, 3... (multi-box)
    PackagingType string
    Weight, Length, Width, Height *float64
    Barcode, LabelURL *string
}
```

### 12. ReturnTask (Задача возврата)

```go
type ReturnTask struct {
    ID, OrderID, WarehouseID int64, WorkerID *int64
    Status string // pending→inspection→approved→restocked (или rejected)
    Reason string, Notes *string
    Items []ReturnItem
}
```

### 13. ReturnItem (Позиция возврата)

```go
type ReturnItem struct {
    ID, ReturnTaskID, ListingID int64
    Quantity int
    Condition string // perfect, good, damaged, defective
    Action string // restock, discard, refurbish
    Photos []ReturnItemPhoto
}
```

### 14. ReturnItemPhoto, QCTask, QCChecklistItem

```go
type ReturnItemPhoto struct { ID, ReturnItemID int64, PhotoURL string }
type QCTask struct {
    ID, ReferenceID, WarehouseID int64, WorkerID *int64
    ReferenceType string // picking/packing/return
    Status string // pending, in_progress, passed, failed
    Checklist []QCChecklistItem, Photos []QCPhoto
}
type QCChecklistItem struct { ID, QCTaskID int64, Question string, Passed *bool, Notes *string }
```

### 15. WorkerPoints (Геймификация)

```go
type WorkerPoints struct {
    WorkerID int64, TotalPoints int, Level int, StreakDays int, LastActivity time.Time
}
```

**Уровни:** 1=Новичок(0), 2=Стажёр(100), 3=Работник(500), 4=Специалист(1500), 5=Эксперт(5000)
**Методы:**
- `GetCurrentLevel()` - текущий уровень
- `GetNextLevel()` - следующий уровень
- `GetProgressToNextLevel()` - прогресс до след. уровня (%)
- `GetStreakBonus()` - бонус за streak (0-50%)

### 16. PointTransaction (Начисление очков)

```go
type PointTransaction struct {
    ID, WorkerID int64, Points int, Reason string, TaskID *int64, TaskType *string
}
```

**Начисления:** pick_item(+1), complete_pick_task(+10), complete_pack_task(+10), batch_bonus(+25), qc_pass(+5), streak_bonus(0-50%), speed_bonus(0-20%), penalty(<0)

### 17. BatchPickingTask, StockAdjustment, CycleCount

```go
type BatchPickingTask struct {
    ID, WarehouseID int64, WorkerID *int64
    OrderIDs []int64, Status string, Items []BatchPickingItem
}
type StockAdjustment struct {
    ID, WarehouseID, LocationID, ListingID int64
    Quantity int, Reason string // damage/loss/found/cycle_count_correction
    Status string, CreatedBy int64, ApprovedBy *int64
}
type CycleCount struct {
    ID, WarehouseID int64, LocationID, WorkerID *int64
    Status string, Items []CycleCountItem
}
```

**Workflow:**
1. Создать CycleCount для локации/склада
2. Работник пересчитывает товары
3. Система сравнивает expected vs counted
4. Если расхождение → создать StockAdjustment

---

## Workflows (Бизнес-процессы)

### Workflow 1: Fulfillment (Picking → Packing → Shipping)

**End-to-end процесс обработки заказа.**

#### Этап 1: Order Confirmed → Picking Created

**Триггер:** Event `order.confirmed` от Listings Service

```
1. Event Consumer получает order.confirmed
2. FulfillmentService.ProcessOrderConfirmed()
   a. Проверка idempotency (задача уже есть?)
   b. Резервирование inventory (ReserveInventory)
   c. Создание PickingTask
   d. Публикация picking.created event
3. Rollback reservations при ошибке
```

**Idempotency:** Проверка существования PickingTask по orderID.

#### Этап 2: Picking (Сборка товаров)

```
pending → assigned → in_progress → completed
```

**Шаги:**

1. **Assign Worker** (Назначить работника)
   - `AssignPickingTask(taskID, workerID)`
   - Статус: `pending → assigned`

2. **Start Picking** (Начать сборку)
   - `StartPickingTask(taskID)`
   - Статус: `assigned → in_progress`

3. **Pick Items** (Сбор позиций)
   - `GetNextItem()` - получить следующую позицию (по sequence)
   - `PickItem(itemID, quantity)` - отметить собранным
   - Gamification: +1 очко за каждую позицию
   - Deduct inventory: уменьшить quantity в ProductLocation

4. **Complete Picking** (Завершить сборку)
   - `CompletePickingTask(taskID)`
   - Проверка: все items picked/short/skipped?
   - Статус: `in_progress → completed`
   - Callback: FulfillmentService.OnPickingCompleted()
   - Gamification: +10 очков + speed bonus
   - Publish: `picking.completed` event

**FIFO оптимизация:**
- GetPickingLocations() сортирует по PlacedAt ASC
- Sequence в PickingTaskItem для оптимального маршрута

#### Этап 3: Packing (Упаковка)

```
pending → in_progress → packed → shipped
```

**Шаги:**

1. **Auto-create PackingTask**
   - Триггер: OnPickingCompleted() callback
   - `CreatePackingTaskFromPicking(pickingTaskID)`
   - Статус: `pending`

2. **Start Packing** (Начать упаковку)
   - `StartPackingTask(taskID, workerID)`
   - Статус: `pending → in_progress`

3. **Set Package Details** (Установить габариты)
   - `SetPackageDetails(taskID, weight, L, W, H, packagingType)`
   - Обязательно для перехода к packed

4. **Pack** (Завершить упаковку)
   - `CompletePackingTask(taskID)`
   - Проверка: dimensions set?
   - Статус: `in_progress → packed`
   - Callback: FulfillmentService.OnPackingCompleted()
   - Gamification: +10 очков

#### Этап 4: Shipping (Отправка)

**Триггер:** OnPackingCompleted() callback

```
1. Получить адрес доставки (DeliveryClient.GetDeliveryAddressByOrder)
2. Создать shipment (DeliveryClient.CreateShipment):
   - provider (PostExpress, D Express, ACS, etc.)
   - from_address (warehouse address)
   - to_address (customer address)
   - package_info (weight, dimensions)
3. Получить tracking_number
4. Обновить packing_task.tracking_number
5. Обновить order status → "ready_for_shipment" (Listings)
6. Публикация shipment.created event
```

**Delivery Providers:**
- PostExpress (Serbia)
- D Express (Serbia)
- ACS (Greece)
- Bex (Serbia)
- YunExpress (China)

### Workflow 2: Returns (Обработка возвратов)

```
pending → inspection → (approved | rejected) → restocked
```

**Шаги:**

1. **Create ReturnTask**
   - `CreateReturnTask(orderID, reason, items[])`
   - Статус: `pending`

2. **Start Inspection** (Начать инспекцию)
   - `StartInspection(taskID, workerID)`
   - Статус: `inspection`

3. **Inspect Each Item** (Проверка каждой позиции)
   - `InspectReturnItem(itemID, condition, action, notes)`
   - `AddReturnItemPhoto(itemID, photoURL)` (опционально)

4. **QC Check** (Опционально)
   - `CreateQCTask(returnTaskID, "return")`
   - Чеклист + фото

5. **Complete Return** (Завершить возврат)
   - `CompleteReturn(taskID)`
   - Статус: `approved` или `rejected`
   - Если approved + action=restock:
     - PlaceProduct() - вернуть товар на склад
     - UpdateOrderStatus() - обновить статус заказа
     - Payment Service - возврат денег клиенту
   - Notification Service - уведомить клиента

### Workflow 3: Quality Control (Контроль качества)

**Когда запускается:**
- После Picking (случайные проверки)
- Перед Shipping (опционально)
- При Returns (обязательно)
- Периодические проверки товаров

**Процесс:**

1. **Create QC Task**
   - `CreateQCTask(referenceID, referenceType)`
   - referenceType: "picking", "packing", "return", "inventory"

2. **Fill Checklist** (Заполнить чеклист)
   - `UpdateChecklist(itemID, passed, notes)`

3. **Add Photos** (Добавить фото)
   - `AddPhoto(taskID, photoURL, notes)`

4. **Complete QC**
   - `CompleteQC(taskID, result)` - result: "passed" или "failed"
   - Если failed:
     - Создать StockAdjustment (если товар поврежден)
     - Создать ReturnTask (если нужно изъять товар)
   - Gamification: +5 очков за QC pass

### Workflow 4: Inventory Management (Управление запасами)

#### Размещение товара (Placement)

```
1. PlaceProduct(listingID, locationID, quantity, batchID?)
2. Создать/обновить ProductLocation
3. Если batch указан:
   - Создать/обновить ProductBatch
   - Связать ProductLocation с BatchID
4. Create InventoryMovement (type: placement)
5. Publish inventory.placed event
```

**Idempotency:** Использовать `idempotency_key` для предотвращения дубликатов.

#### Перемещение товара (Move)

```
1. MoveProduct(productLocationID, toLocationID, quantity)
2. Deduct from source ProductLocation
3. Add to destination ProductLocation
4. Create InventoryMovement (type: move)
5. Publish inventory.moved event
```

#### Резервирование (Reserve)

```
1. ReserveInventory(orderID, listingID, warehouseID, quantity)
2. GetPickingLocations(FIFO) - выбрать оптимальные локации
3. Создать WMSReservation для каждой локации
4. Обновить product_location.reserved_quantity
5. Установить TTL (default: 30 min)
6. Publish inventory.reserved event
```

**Auto-release:** Cron job освобождает expired резервации.

#### Освобождение резерва (Release)

```
1. ReleaseReservation(orderID | reservationID)
2. Найти все reservations
3. Обновить статус → released
4. Уменьшить product_location.reserved_quantity
5. Publish inventory.released event
```

#### Commit резерва (Commit)

```
1. CommitReservation(orderID)
2. Найти все active reservations
3. Обновить статус → committed
4. Reservations остаются до завершения packing
5. При packing.completed → auto-release committed reservations
```

---

## Gamification (Геймификация)

### Уровни работников

| Уровень | Название | Мин. очков | Badge |
|---------|----------|-----------|-------|
| 1 | Новичок | 0 | /badges/level1.png |
| 2 | Стажёр | 100 | /badges/level2.png |
| 3 | Работник | 500 | /badges/level3.png |
| 4 | Специалист | 1500 | /badges/level4.png |
| 5 | Эксперт | 5000 | /badges/level5.png |

### Начисление очков

| Действие | Базовые очки | Примечание |
|----------|-------------|-----------|
| Сборка позиции | +1 | За каждый PickItem |
| Завершение picking task | +10 | CompletePickingTask |
| Завершение packing task | +10 | CompletePackingTask |
| Batch picking bonus | +25 | За завершение batch |
| QC pass | +5 | Прохождение проверки |

### Бонусы

#### Streak Bonus (Серия дней подряд)

```
Бонус = StreakDays × 10%
Макс: 50% (5+ дней подряд)

Пример:
- 1 день: 0% бонуса
- 2 дня: +10%
- 3 дня: +20%
- 5+ дней: +50%
```

**Механизм:**
- Каждый день работы увеличивает streak
- Пропуск дня → streak сбрасывается на 1
- UpdateStreak() при каждом действии работника

#### Speed Bonus (Бонус за скорость)

```
Improvement = (AvgTime - ActualTime) / AvgTime
SpeedBonus = Improvement × 0.4 (max 20%)

Пример:
- Avg time: 10 мин
- Actual time: 6 мин
- Improvement: 40%
- Bonus: 16% (0.4 × 0.4 = 16%)
```

**Применение:** При завершении picking/packing task.

### Leaderboard (Таблица лидеров)

**API:** `GetLeaderboard(warehouseID, period)`

**Периоды:**
- `day` - За сегодня
- `week` - За неделю
- `month` - За месяц
- `all_time` - За все время

**Формат:**
```json
[
  {
    "rank": 1,
    "worker_id": 42,
    "worker_name": "Ivan Petrov",
    "total_points": 1250,
    "level": 4,
    "level_name": "Специалист"
  }
]
```

---

## Use Cases (Варианты использования)

### UC-1: Создать склад с зонами

**Актор:** Admin

**Сценарий:**
1. CreateWarehouse(code, name, type, address, city, country)
2. CreateZone(warehouseID, code, name, type="storage")
3. CreateZone(warehouseID, code, name, type="picking")
4. CreateZone(warehouseID, code, name, type="packing")
5. CreateLocations для каждой зоны

**Результат:** Полностью настроенный склад с зонами и локациями.

### UC-2: Разместить товар на складе

**Актор:** Warehouse Manager

**Сценарий:**
1. GetWarehouse(warehouseID) - выбрать склад
2. ListLocations(warehouseID, zoneType="storage", status="available")
3. PlaceProduct(listingID, locationID, quantity, batchNumber?)
4. Publish inventory.placed event

**Результат:** Товар размещен на локации, inventory обновлен.

### UC-3: Обработать заказ (Fulfillment)

**Актор:** Event Consumer (автоматически)

**Сценарий:**
1. **Receive event:** order.confirmed от Listings
2. **Reserve:** ReserveInventory для всех items
3. **Create:** CreatePickingTask(orderID, items)
4. **Assign:** Worker берет задачу → AssignPickingTask
5. **Pick:** Worker собирает items → PickItem (×N)
6. **Complete:** CompletePickingTask → auto-create PackingTask
7. **Pack:** Worker упаковывает → SetPackageDetails, CompletePackingTask
8. **Ship:** CreateShipment → получить tracking_number
9. **Update:** UpdateOrderStatus("ready_for_shipment")

**Результат:** Заказ собран, упакован, готов к отправке.

### UC-4: Обработать возврат с инспекцией

**Актор:** Warehouse Worker

**Сценарий:**
1. CreateReturnTask(orderID, reason, items)
2. StartInspection(taskID, workerID)
3. Для каждого item:
   - InspectReturnItem(itemID, condition, action)
   - Если defective → AddReturnItemPhoto (×N)
4. CreateQCTask(returnTaskID, "return") (опционально)
5. CompleteReturn(taskID)
6. Если action="restock":
   - PlaceProduct (вернуть на склад)
   - Payment: Refund (возврат денег)
7. Notification: уведомить клиента

**Результат:** Возврат обработан, товар возвращен на склад (или утилизирован).

---

## gRPC Services & Methods (120+ методов)

### 1. WarehouseService (15 методов)

**Управление складами, зонами, локациями.**

- CreateWarehouse, GetWarehouse, UpdateWarehouse, DeleteWarehouse, ListWarehouses
- CreateZone, GetZone, UpdateZone, DeleteZone, ListZones
- CreateLocation, GetLocation, UpdateLocation, DeleteLocation, ListLocations

### 2. InventoryService (12 методов)

**Управление запасами, резервирование, батчи.**

- PlaceProduct, MoveProduct, RemoveProduct
- GetAvailableQuantity, GetProductLocations, GetPickingLocations
- ReserveInventory, ReleaseReservation, CommitReservation
- CreateBatch, GetBatch, ListBatches

### 3. PickingService (10 методов)

**Управление задачами сборки.**

- CreatePickingTask, GetPickingTask, GetPickingTaskByOrderID
- AssignPickingTask, StartPickingTask
- PickItem, GetNextPickingItem
- CompletePickingTask, CancelPickingTask, ListPickingTasks

### 4. PackingService (9 методов)

**Управление задачами упаковки.**

- CreatePackingTask, CreatePackingTaskFromPicking, GetPackingTask
- StartPackingTask, SetPackageDetails
- CompletePackingTask, CancelPackingTask
- PrintLabel, ListPackingTasks

### 5. WorkerService (12 методов)

**Управление работниками, gamification.**

- CreateWorker, GetWorker, UpdateWorker, DeleteWorker, ListWorkers
- AuthenticateWorker, RefreshToken
- GetWorkerPoints, GetPointTransactions, GetLeaderboard
- GetWorkerStats, AddPointTransaction

### 6. OwnerPortalService (15 методов)

**Портал для владельцев складов.**

- GetMyWarehouses, GetWarehouseAccess, GetOwnerDashboard
- AddWarehouseStaff, RemoveWarehouseStaff, UpdateStaffRole, ListWarehouseStaff
- CreateEmailInvitation, CreateLinkInvitation, ListInvitations, RevokeInvitation
- AcceptInvitation, DeclineInvitation, ValidateInviteCode, GetInvitationDetails

### 7-13. Additional Services (60+ методов)

- BatchPickingService (10 методов) - пакетная сборка
- QualityControlService (8 методов) - контроль качества
- ReturnsService (9 методов) - обработка возвратов
- StockAdjustmentService (5 методов) - корректировки
- LocationTransferService (6 методов) - перемещения
- CycleCountService (7 методов) - инвентаризация
- SubstitutionService (7 методов) - замены товаров

**Всего:** 13 gRPC сервисов, 120+ методов

---

## Интеграции

### 1. Listings Service (gRPC)
- GetListingInfo, CheckAvailability, UpdateOrderStatus, DecrementStock

### 2. Delivery Service (gRPC)
- CreateShipment, GetShipment, CancelShipment, GetDeliveryAddressByOrder

### 3. Auth Service (gRPC)
- ValidateToken, CreateWorkerToken, GetUserInfo

### 4. Notification Service (gRPC)
- SendNotification, SendEmail, SendSMS

### 5. Redis Streams (Event-Driven)

**WMS публикует (в WMS Redis 36381):**
- wms:events:picking - picking.created, picking.completed
- wms:events:packing - packing.started, packing.completed
- wms:events:shipment - shipment.created
- wms:events:inventory - inventory.placed, inventory.reserved, inventory.released

**WMS потребляет (из Listings Redis 36380):**
- listings:events:orders - order.confirmed, order.cancelled

---

## Конфигурация

**PostgreSQL:** localhost:35435, wms_db
**Redis WMS:** localhost:36381
**Redis Listings:** localhost:36380
**gRPC Services:** Listings:50053, Delivery:50052, Auth:50051, Notification:50054

---

## Запуск сервиса

```bash
cd /p/github.com/vondi-global/warehouse
docker compose up -d postgres redis
./migrator up
go run cmd/server/main.go

# Health check
curl http://localhost:8085/health
grpcurl -plaintext localhost:50055 list
```

---

## Расположение кода

```
/p/github.com/vondi-global/warehouse/
├── api/proto/warehouse.proto      # gRPC definitions (2200+ lines)
├── cmd/server/main.go             # Точка входа
├── internal/
│   ├── domain/                    # 21+ entity файлов
│   ├── repository/postgres/       # 30+ репозиториев
│   ├── service/                   # 15+ сервисов
│   ├── grpc/handlers/             # 13 gRPC handlers
│   ├── events/                    # Redis Streams pub/sub
│   └── client/                    # Клиенты для других сервисов
└── migrations/                    # 25+ SQL миграций
```

---

## Ссылки

- **Репозиторий:** /p/github.com/vondi-global/warehouse
- **Сервис паспорт:** /p/github.com/vondi-global/.passport/services/warehouse.md
- **Fulfillment flow:** /p/github.com/vondi-global/.passport/flows/fulfillment.md
- **System Passport:** /p/github.com/vondi-global/SYSTEM_PASSPORT.md

---

**Версия:** 1.5.0
**Дата:** 2025-12-21
**Статус:** Production Ready
