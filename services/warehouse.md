# Паспорт: Warehouse/WMS Service

> Обновлено: 2025-12-21

## Идентификация

| Параметр | Значение |
|----------|----------|
| Название | Warehouse Management System (WMS) |
| Репозиторий | /p/github.com/vondi-global/warehouse |
| Версия | 1.5.0 |
| Порт HTTP | 8085 |
| Порт gRPC | 50055 |
| Статус | Production Ready |

## Назначение

Микросервис управления складом для платформы Vondi. Обеспечивает:
- Управление складами, зонами, локациями
- Размещение и перемещение товаров
- Picking и Packing workflows
- Возвраты и контроль качества
- Gamification для работников
- Интеграция с Listings/Delivery через Redis Streams

## Технологический стек

| Технология | Версия |
|------------|--------|
| Go | 1.23 |
| PostgreSQL | 15-alpine |
| Redis | 7-alpine |
| gRPC | v1.76.0 |
| Zerolog | v1.33.0 |
| Sqlx | v1.4.0 |

## Архитектура

**DDD (Domain-Driven Design):**
```
cmd/server/main.go           # Точка входа
api/proto/warehouse.proto    # gRPC определения (2200+ строк)
internal/
├── config/                  # Конфигурация
├── domain/                  # Бизнес-логика (21+ entities)
├── repository/postgres/     # Data access (30+ таблиц)
├── service/                 # Application services
├── grpc/handlers/           # gRPC handlers
├── events/                  # Redis Streams integration
└── client/                  # Клиенты для других сервисов
migrations/                  # 25+ SQL миграций
```

## Зависимости

| Сервис | Тип связи | Назначение |
|--------|-----------|------------|
| PostgreSQL (wms_db) | TCP | Хранение данных склада |
| Redis (WMS) | TCP | События, кеширование |
| Redis (Listings) | Redis Streams | Потребление order.confirmed |
| Listings Service | gRPC | Информация о товарах |
| Delivery Service | gRPC | Создание shipments |
| Auth Service | gRPC | Валидация пользователей |
| Notification Service | gRPC | Отправка уведомлений |

## Конфигурация (переменные окружения)

### Основные настройки
| Переменная | Описание | Пример |
|------------|----------|--------|
| WMS_ENV | Окружение | development |
| WMS_GRPC_PORT | gRPC порт | 50055 |
| WMS_HTTP_PORT | HTTP порт (health/metrics) | 8085 |
| LOG_LEVEL | Уровень логирования | debug |
| LOG_FORMAT | Формат логов | json |
| METRICS_ENABLED | Включить метрики | true |

### База данных (PostgreSQL)
| Переменная | Описание | Пример |
|------------|----------|--------|
| WMS_DB_HOST | Хост PostgreSQL | localhost |
| WMS_DB_PORT | Порт PostgreSQL | 35435 |
| WMS_DB_USER | Пользователь | wms_user |
| WMS_DB_PASSWORD | Пароль | wms_secure_pass_2025 |
| WMS_DB_NAME | Имя БД | wms_db |
| WMS_DB_SSL_MODE | SSL режим | disable |
| WMS_DB_MAX_CONNECTIONS | Макс. соединений | 25 |

### Redis (события и кеширование)
| Переменная | Описание | Пример |
|------------|----------|--------|
| WMS_REDIS_HOST | Redis для WMS событий | localhost |
| WMS_REDIS_PORT | Порт | 36381 |
| WMS_REDIS_PASSWORD | Пароль | (пусто) |
| WMS_REDIS_DB | Database номер | 0 |

### Redis (Listings - для order events)
| Переменная | Описание | Пример |
|------------|----------|--------|
| WMS_LISTINGS_REDIS_HOST | Redis Listings | localhost |
| WMS_LISTINGS_REDIS_PORT | Порт | 36380 |
| WMS_LISTINGS_REDIS_PASSWORD | Пароль | redis_password |
| WMS_LISTINGS_REDIS_DB | Database номер | 0 |

### Микросервисы (gRPC endpoints)
| Переменная | Описание | Пример |
|------------|----------|--------|
| WMS_LISTINGS_GRPC_URL | Listings Service | localhost:50053 |
| WMS_DELIVERY_GRPC_URL | Delivery Service | localhost:50052 |
| WMS_PAYMENT_GRPC_URL | Payment Service | localhost:50056 |
| WMS_AUTH_GRPC_URL | Auth Service | localhost:50051 |
| WMS_NOTIFICATION_GRPC_URL | Notification Service | localhost:50054 |

## Доменная модель

### Warehouse & Zones
```go
// Warehouse - склад
type Warehouse struct {
    ID          int64           // PK
    Name        string          // Название
    Code        string          // Уникальный код
    Type        WarehouseType   // MAIN, REGIONAL, PARTNER, DARK_STORE
    Address     string
    IsActive    bool
    OwnerUserID *int64          // ID владельца
    Zones       []Zone          // Зоны склада
}

// Zone - зона склада
type Zone struct {
    ID          int64
    WarehouseID int64
    Name        string
    Code        string
    Type        ZoneType        // RECEIVING, STORAGE, PICKING, PACKING, SHIPPING, RETURNS
    IsActive    bool
    Locations   []Location
}

// Location - локация (стеллаж, паллета)
type Location struct {
    ID          int64
    ZoneID      int64
    Code        string          // A1-01-01
    Type        LocationType    // SHELF, PALLET, FLOOR, BIN
    Status      LocationStatus  // AVAILABLE, OCCUPIED, RESERVED, MAINTENANCE
    Capacity    int
}
```

### Product Location & Inventory
```go
// ProductLocation - размещение товара
type ProductLocation struct {
    ID          int64
    LocationID  int64
    ListingID   int64
    Quantity    int
    BatchID     *int64          // Опционально: партия
    ExpiryDate  *time.Time
    PlacedAt    time.Time
}

// Reservation - резервирование товара
type Reservation struct {
    ID          int64
    OrderID     int64
    ListingID   int64
    Quantity    int
    WarehouseID int64
    Status      ReservationStatus
    ExpiresAt   time.Time
}
```

### Worker (Работник)
```go
// Worker - работник склада
type Worker struct {
    ID          int64
    UserID      int64           // Связь с Auth Service
    WarehouseID int64
    Name        string
    Email       string
    Phone       string
    Role        string          // picker, packer, manager
    Status      string          // active, inactive, on_break
    IsActive    bool
    HiredAt     time.Time
}
```

### Picking Workflow
```go
// PickingTask - задача на сбор товаров
type PickingTask struct {
    ID          int64
    OrderID     int64           // ID заказа
    WarehouseID int64
    WorkerID    *int64          // Назначенный работник
    Status      PickingStatus   // pending, assigned, in_progress, completed, cancelled
    Priority    int             // 1-10
    Items       []PickingTaskItem
    CreatedAt   time.Time
    StartedAt   *time.Time
    CompletedAt *time.Time
}

// PickingTaskItem - позиция для сбора
type PickingTaskItem struct {
    ID             int64
    PickingTaskID  int64
    ListingID      int64
    Quantity       int
    PickedQuantity int
    LocationID     *int64       // Откуда забрать
    Status         string       // pending, picked, short_pick
}
```

### Packing Workflow
```go
// PackingTask - задача на упаковку
type PackingTask struct {
    ID             int64
    PickingTaskID  int64        // Создаётся после picking
    OrderID        int64
    WarehouseID    int64
    WorkerID       *int64
    Status         PackingStatus // pending, in_progress, completed, cancelled
    PackageWeight  *float64
    PackageDims    *string      // JSON: {"length":30,"width":20,"height":10}
    TrackingNumber *string      // От Delivery Service
    StartedAt      *time.Time
    CompletedAt    *time.Time
}

// Package - упакованная посылка
type Package struct {
    ID             int64
    PackingTaskID  int64
    TrackingNumber string
    Weight         float64      // kg
    Length         int          // cm
    Width          int
    Height         int
    CarrierCode    string       // post_express, others
}
```

### Returns (Возвраты)
```go
// ReturnTask - задача на обработку возврата
type ReturnTask struct {
    ID          int64
    OrderID     int64
    WarehouseID int64
    WorkerID    *int64
    Status      ReturnStatus    // pending, inspection, approved, rejected, restocked
    Reason      string
    Items       []ReturnItem
    CreatedAt   time.Time
    InspectedAt *time.Time
    CompletedAt *time.Time
}

// ReturnItem - позиция возврата
type ReturnItem struct {
    ID           int64
    ReturnTaskID int64
    ListingID    int64
    Quantity     int
    Condition    string         // perfect, good, damaged, defective
    Notes        string
    Action       string         // restock, discard, refurbish
    Photos       []ReturnItemPhoto
}

// ReturnItemPhoto - фото дефектов
type ReturnItemPhoto struct {
    ID           int64
    ReturnItemID int64
    PhotoURL     string
    Notes        string
    UploadedAt   time.Time
}
```

### Quality Control
```go
// QCTask - задача контроля качества
type QCTask struct {
    ID           int64
    ReferenceID  int64          // ID задачи (picking/packing/return)
    ReferenceType string        // "picking", "packing", "return"
    WarehouseID  int64
    WorkerID     *int64
    Status       QCStatus       // pending, in_progress, passed, failed
    Checklist    []QCChecklistItem
    Photos       []QCPhoto
    CreatedAt    time.Time
    CompletedAt  *time.Time
}

// QCChecklistItem - пункт проверки
type QCChecklistItem struct {
    ID        int64
    QCTaskID  int64
    Question  string           // "Товар соответствует описанию?"
    Passed    *bool
    Notes     string
}

// QCPhoto - фото для QC
type QCPhoto struct {
    ID       int64
    QCTaskID int64
    PhotoURL string
    Notes    string
}
```

### Batch Picking (Пакетная сборка)
```go
// BatchPickingTask - сборка нескольких заказов одновременно
type BatchPickingTask struct {
    ID          int64
    WarehouseID int64
    WorkerID    *int64
    OrderIDs    []int64        // Несколько заказов
    Status      BatchStatus    // pending, assigned, in_progress, completed
    Items       []BatchPickingItem
    CreatedAt   time.Time
    StartedAt   *time.Time
    CompletedAt *time.Time
}

// BatchPickingItem - позиция в batch picking
type BatchPickingItem struct {
    ID           int64
    BatchTaskID  int64
    OrderID      int64
    ListingID    int64
    Quantity     int
    LocationID   *int64
    PickedQty    int
    Status       string
}
```

### Gamification (Геймификация)
```go
// WorkerPoints - очки работника
type WorkerPoints struct {
    WorkerID     int64
    TotalPoints  int
    Level        int            // 1-5
    StreakDays   int            // Дни подряд
    LastActivity time.Time
}

// Level - уровень работника
type Level struct {
    LevelNum  int
    Name      string
    MinPoints int
    BadgeURL  string
}

// Уровни:
// 1. Новичок (0 pts)
// 2. Стажёр (100 pts)
// 3. Работник (500 pts)
// 4. Специалист (1500 pts)
// 5. Эксперт (5000 pts)

// PointTransaction - начисление/списание очков
type PointTransaction struct {
    ID       int64
    WorkerID int64
    Points   int            // Может быть отрицательным (штрафы)
    Reason   PointReason    // pick_item, complete_pick_task, speed_bonus, penalty
    TaskID   *int64
    TaskType *string
}

// Причины начисления очков:
// - pick_item: +1 за каждую позицию
// - complete_pick_task: +10 за завершённую задачу
// - complete_pack_task: +10
// - batch_bonus: +25
// - qc_pass: +5
// - streak_bonus: до +50% за streak
// - speed_bonus: до +20% за скорость
```

### Stock Adjustment (Корректировка остатков)
```go
// StockAdjustment - корректировка товарных остатков
type StockAdjustment struct {
    ID           int64
    WarehouseID  int64
    LocationID   int64
    ListingID    int64
    Quantity     int            // Может быть отрицательным
    Reason       string         // damage, loss, found, cycle_count_correction
    Status       string         // pending, approved, rejected
    CreatedBy    int64
    ApprovedBy   *int64
    Notes        string
    CreatedAt    time.Time
    ApprovedAt   *time.Time
}
```

### Location Transfer (Перемещение между локациями)
```go
// LocationTransfer - перемещение товара между локациями
type LocationTransfer struct {
    ID              int64
    WarehouseID     int64
    ListingID       int64
    FromLocationID  int64
    ToLocationID    int64
    Quantity        int
    Status          string    // pending, in_progress, completed, cancelled
    WorkerID        *int64
    Reason          string    // optimization, damage, reallocation
    CreatedAt       time.Time
    CompletedAt     *time.Time
}
```

### Cycle Count (Инвентаризация)
```go
// CycleCount - инвентаризация
type CycleCount struct {
    ID          int64
    WarehouseID int64
    LocationID  *int64         // Null = весь склад
    Status      string         // pending, in_progress, completed, cancelled
    WorkerID    *int64
    Items       []CycleCountItem
    CreatedAt   time.Time
    StartedAt   *time.Time
    CompletedAt *time.Time
}

// CycleCountItem - позиция инвентаризации
type CycleCountItem struct {
    ID            int64
    CycleCountID  int64
    LocationID    int64
    ListingID     int64
    ExpectedQty   int
    CountedQty    *int
    Discrepancy   int           // CountedQty - ExpectedQty
    Notes         string
}
```

### Substitution (Замена товара)
```go
// Substitution - замена товара в заказе
type Substitution struct {
    ID                  int64
    PickingTaskID       int64
    OriginalListingID   int64
    SubstituteListingID *int64
    Quantity            int
    Reason              string    // out_of_stock, damaged, expired
    Status              string    // pending, customer_approved, customer_rejected, selected
    CustomerResponse    *bool
    CreatedAt           time.Time
    RespondedAt         *time.Time
}

// SubstitutionOption - вариант замены
type SubstitutionOption struct {
    ID             int64
    SubstitutionID int64
    ListingID      int64
    PriceDiff      float64       // Разница в цене
    SimilarityScore float64      // 0-1
}
```

### Warehouse Staff & Invitation
```go
// WarehouseStaff - персонал склада
type WarehouseStaff struct {
    ID          int64
    WarehouseID int64
    UserID      int64
    Role        string         // owner, manager, worker
    AddedAt     time.Time
}

// Invitation - приглашение в склад
type Invitation struct {
    ID          int64
    WarehouseID int64
    InviterID   int64
    InviteeID   *int64         // Null для link-based
    Email       *string
    Role        string
    Type        string         // email, link
    Code        string         // Для link-based
    Status      string         // pending, accepted, declined, revoked, expired
    ExpiresAt   time.Time
    CreatedAt   time.Time
    AcceptedAt  *time.Time
}
```

## gRPC API (13 сервисов, 120+ методов)

### Основные сервисы (13 gRPC services)

**1. WarehouseService** - управление складами (15 методов)
- CRUD для warehouses, zones, locations

**2. InventoryService** - управление запасами (12 методов)
- PlaceProduct, MoveProduct, RemoveProduct
- GetAvailableQuantity, GetProductLocations
- ReserveInventory, ReleaseReservation, CommitReservation
- CreateBatch, GetBatch, ListBatches

**3. PickingService** - сборка заказов (10 методов)
- CreatePickingTask, AssignPickingTask, StartPickingTask
- PickItem, CompletePickingTask, GetNextItem
- GetPickingTaskByOrderID, ListPickingTasks

**4. PackingService** - упаковка (9 методов)
- CreatePackingTaskFromPicking, StartPackingTask
- SetPackageDetails, PackItem, CompletePackingTask
- PrintLabel (PDF/ZPL)

**5. WorkerService** - работники (12 методов)
- AuthenticateWorker, RefreshToken
- CRUD workers, GetWorkerStats
- GetWorkerPoints, GetPointTransactions, GetLeaderboard

**6. OwnerPortalService** - портал владельца (15 методов)
- GetMyWarehouses, GetOwnerDashboard
- AddWarehouseStaff, ListWarehouseStaff
- CreateEmailInvitation, CreateLinkInvitation
- AcceptInvitation, ValidateInviteCode

**7. BatchPickingService** - пакетная сборка (10 методов)
- CreateBatchPickingTask, PickBatchItem
- CompleteBatchPicking, GetNextBatchItem

**8. QualityControlService** - контроль качества (8 методов)
- CreateQCTask, UpdateChecklist, AddPhoto
- CompleteQC (passed/failed)

**9. ReturnsService** - возвраты (9 методов)
- CreateReturnTask, StartInspection, InspectReturnItem
- CompleteReturn, AddReturnItemPhoto

**10. StockAdjustmentService** - корректировки (5 методов)
- CreateAdjustment, ApproveAdjustment, RejectAdjustment

**11. LocationTransferService** - перемещения (6 методов)
- CreateTransfer, StartTransfer, CompleteTransfer

**12. CycleCountService** - инвентаризация (7 методов)
- CreateCycleCount, StartCycleCount, CountItem

**13. SubstitutionService** - замены товаров (7 методов)
- GetAvailableSubstitutions, CreateSubstitution
- SelectSubstitution, ApproveSubstitution

## Workflow 1: Picking (Сборка заказов)

### Состояния PickingTask
```
pending → assigned → in_progress → completed
                        ↓
                    cancelled
```

### Процесс
1. **Создание задачи** (event: order.confirmed)
   - Consumer слушает `listings:events:orders`
   - Создаёт PickingTask со статусом `pending`
   - Резервирует товары (ReserveInventory)

2. **Назначение работника**
   - `AssignPickingTask(worker_id)`
   - Статус → `assigned`

3. **Начало сборки**
   - `StartPickingTask(task_id)`
   - Статус → `in_progress`

4. **Сбор позиций**
   - `GetNextItem()` - получить следующую позицию
   - `PickItem(item_id, quantity)` - отметить сборку
   - Gamification: +1 очко за каждую позицию

5. **Завершение**
   - `CompletePickingTask(task_id)`
   - Статус → `completed`
   - Создаёт PackingTask автоматически
   - Gamification: +10 очков + speed bonus
   - Публикует event: `picking.completed`

### Short Pick (Недостача)
- Если товара меньше, чем нужно
- `PickItem(item_id, picked_qty < required_qty)`
- Создаёт Substitution запрос (опционально)
- Уведомление клиенту через Notification Service

## Workflow 2: Packing (Упаковка)

### Состояния PackingTask
```
pending → in_progress → completed
             ↓
         cancelled
```

### Процесс
1. **Создание** (автоматически после Picking)
   - `CreatePackingTaskFromPicking(picking_task_id)`
   - Статус → `pending`

2. **Начало упаковки**
   - `StartPackingTask(task_id)`
   - Статус → `in_progress`

3. **Установка параметров посылки**
   - `SetPackageDetails(weight, dims)`
   - Вызов Delivery Service → создание shipment
   - Получение tracking number

4. **Упаковка позиций**
   - `PackItem(item_id)` - отметить упакованную позицию

5. **Завершение**
   - `CompletePackingTask(task_id)`
   - Статус → `completed`
   - Gamification: +10 очков
   - Публикует event: `packing.completed`

6. **Печать ярлыка**
   - `PrintLabel(task_id)` → возвращает PDF/ZPL

## Workflow 3: Returns (Возвраты)

### Состояния ReturnTask
```
pending → inspection → (approved | rejected) → restocked
                           ↓
                       cancelled
```

### Процесс
1. **Создание задачи возврата**
   - `CreateReturnTask(order_id, reason, items[])`
   - Статус → `pending`

2. **Начало инспекции**
   - `StartInspection(task_id, worker_id)`
   - Статус → `inspection`

3. **Проверка каждой позиции**
   - `InspectReturnItem(item_id, condition, notes)`
   - Condition: perfect, good, damaged, defective
   - Action: restock, discard, refurbish
   - `AddReturnItemPhoto(item_id, photo_url)` (опционально)

4. **QC проверка** (опционально)
   - `CreateQCTask(return_task_id, "return")`
   - QC проверяет состояние товара
   - Может добавить фото и чеклист

5. **Завершение**
   - `CompleteReturn(task_id)`
   - Статус → `approved` или `rejected`
   - Если approved + restock → возвращает на склад (PlaceProduct)
   - Уведомление клиенту + Payment Service (возврат денег)

## Quality Control (Контроль качества)

### Когда запускается QC
- После Picking (случайные проверки)
- После Packing (перед отправкой)
- При Returns (проверка состояния)
- Периодические проверки товаров на складе

### Процесс
1. **Создание задачи**
   - `CreateQCTask(reference_id, reference_type)`
   - reference_type: "picking", "packing", "return", "inventory"

2. **Чеклист**
   - Заполняется список вопросов (checklist)
   - `UpdateChecklist(item_id, passed, notes)`

3. **Фотографии**
   - `AddPhoto(task_id, photo_url, notes)`

4. **Завершение**
   - `CompleteQC(task_id, result)`
   - result: "passed" или "failed"
   - Если failed → создаёт StockAdjustment или ReturnTask
   - Gamification: +5 очков за QC pass

## Gamification (Геймификация для работников)

### Уровни
| Уровень | Название | Очки | Badge |
|---------|----------|------|-------|
| 1 | Новичок | 0 | /badges/level1.png |
| 2 | Стажёр | 100 | /badges/level2.png |
| 3 | Работник | 500 | /badges/level3.png |
| 4 | Специалист | 1500 | /badges/level4.png |
| 5 | Эксперт | 5000 | /badges/level5.png |

### Начисление очков
| Действие | Очки |
|----------|------|
| pick_item | +1 |
| complete_pick_task | +10 |
| complete_pack_task | +10 |
| batch_bonus | +25 |
| qc_pass | +5 |
| streak_bonus | до +50% |
| speed_bonus | до +20% |

### Speed Bonus (Бонус за скорость)
- Сравнивается с средним временем выполнения задачи
- Если выполнил быстрее на 40% → +20% бонуса к очкам

### Streak Bonus (Серия дней подряд)
- За каждый день работы подряд: +10% к очкам
- Максимум: +50% (5 дней подряд)
- Если пропустил день → streak сбрасывается

### Leaderboard (Таблица лидеров)
- `GetLeaderboard(warehouse_id, period)` → топ работников
- Period: day, week, month, all_time
- Сортировка по total_points

### API методы
```protobuf
rpc GetWorkerPoints(GetWorkerPointsRequest) returns (WorkerPointsResponse);
rpc GetPointTransactions(GetPointTransactionsRequest) returns (PointTransactionsResponse);
rpc GetLeaderboard(GetLeaderboardRequest) returns (LeaderboardResponse);
```

## Redis Streams (События)

### WMS публикует (в WMS Redis)
| Stream | Event | Данные |
|--------|-------|--------|
| wms:events:picking | picking.created | task_id, order_id, warehouse_id |
| wms:events:picking | picking.assigned | task_id, worker_id |
| wms:events:picking | picking.completed | task_id, order_id, worker_id |
| wms:events:packing | packing.started | task_id, order_id, worker_id |
| wms:events:packing | packing.completed | task_id, tracking_number, shipment_id |
| wms:events:shipment | shipment.created | shipment_id, tracking_number, carrier_code |
| wms:events:inventory | inventory.placed | listing_id, location_id, quantity |
| wms:events:inventory | inventory.moved | listing_id, from_location, to_location, quantity |
| wms:events:inventory | inventory.reserved | order_id, listing_id, quantity |
| wms:events:inventory | inventory.released | order_id, listing_id, quantity |

### WMS потребляет (из Listings Redis)
| Stream | Event | Действие |
|--------|-------|----------|
| listings:events:orders | order.confirmed | Создать PickingTask + резервировать товары |
| listings:events:orders | order.cancelled | Отменить PickingTask + освободить резервации |

### Consumer Group
- Группа: `wms-consumers`
- XREADGROUP для гарантии обработки
- ACK после успешного выполнения

## Миграции (25+ файлов)

| Миграция | Описание |
|----------|----------|
| 000_create_functions.up.sql | Функции: set_updated_at(), gen_code() |
| 001_create_warehouses.up.sql | Таблица warehouses |
| 002_create_zones.up.sql | Таблица warehouse_zones |
| 003_create_locations.up.sql | Таблица storage_locations |
| 004_create_product_batches.up.sql | Таблица product_batches |
| 005_create_product_locations.up.sql | Таблица product_locations |
| 006_create_wms_reservations.up.sql | Таблица wms_reservations |
| 007_create_inventory_movements.up.sql | Таблица inventory_movements |
| 008_picking_tasks.up.sql | Таблица picking_tasks |
| 009_picking_task_items.up.sql | Таблица picking_task_items |
| 010_packing_tasks.up.sql | Таблица packing_tasks |
| 011_packages.up.sql | Таблица packages |
| 012_workers.up.sql | Таблица workers |
| 013_warehouse_ownership.up.sql | owner_user_id → warehouses |
| 014_warehouse_staff.up.sql | Таблица warehouse_staff |
| 015_idempotency_key.up.sql | idempotency_key → tasks |
| 016_warehouse_invitations.up.sql | Таблица warehouse_invitations |
| 017_quality_control.up.sql | qc_tasks, qc_checklist_items, qc_photos |
| 018_gamification.up.sql | worker_points, point_transactions |
| 019_batch_picking.up.sql | batch_picking_tasks, batch_picking_items |
| 020_returns.up.sql | return_tasks, return_items |
| 021_cycle_counts.up.sql | cycle_counts, cycle_count_items |
| 021_stock_adjustments.up.sql | stock_adjustments |
| 022_location_transfers.up.sql | location_transfers |
| 024_substitutions.up.sql | substitutions, substitution_options |
| 025_return_item_photos.up.sql | return_item_photos |

### Применение миграций
```bash
cd /p/github.com/vondi-global/warehouse
./migrator up
```

## Связи с другими сервисами

| Сервис | Методы | Назначение |
|--------|--------|------------|
| **Listings** (gRPC) | GetListingInfo, CheckAvailability | Информация о товарах |
| **Delivery** (gRPC) | CreateShipment | Создание отправлений, tracking number |
| **Auth** (gRPC) | ValidateToken | Валидация JWT, user_id, roles |
| **Notification** (gRPC) | SendNotification | Уведомления (email, SMS) |

## Запуск и проверка

### Локальный запуск
```bash
cd /p/github.com/vondi-global/warehouse

# 1. Запустить PostgreSQL и Redis
docker compose up -d postgres redis

# 2. Применить миграции
./migrator up

# 3. Запустить сервис
go run cmd/server/main.go

# Или через screen
screen -dmS wms-service-50055 bash -c 'cd /p/github.com/vondi-global/warehouse && go run cmd/server/main.go 2>&1 | tee /tmp/wms.log'
```

### Health Check
```bash
curl http://localhost:8085/health
# {"status":"ok","timestamp":"2025-12-21T10:00:00Z"}

curl http://localhost:8085/health/ready
# Проверяет: PostgreSQL, Redis, gRPC listeners
```

### Проверка gRPC (grpcurl)
```bash
# Список сервисов
grpcurl -plaintext localhost:50055 list

# Создать warehouse
grpcurl -plaintext -d '{"name":"Main Warehouse","code":"WH001","type":"WAREHOUSE_TYPE_MAIN","address":"Belgrade, Serbia"}' \
  localhost:50055 wms.WarehouseService/CreateWarehouse

# Получить warehouse
grpcurl -plaintext -d '{"id":1}' localhost:50055 wms.WarehouseService/GetWarehouse
```

### Проверка через Redis CLI
```bash
# Подключиться к WMS Redis
docker exec -it warehouse_redis redis-cli

# Посмотреть стримы
XINFO STREAM wms:events:picking

# Прочитать события
XREAD COUNT 10 STREAMS wms:events:picking 0
```

## Мониторинг

### Логи
- **Формат:** JSON (zerolog)
- **Уровни:** debug, info, warn, error, fatal
- **Файл:** stdout (в production → journald или file)

```json
{
  "level": "info",
  "timestamp": "2025-12-21T10:00:00Z",
  "service": "wms",
  "message": "Picking task created",
  "task_id": 123,
  "order_id": 456,
  "warehouse_id": 1
}
```

### Метрики (Prometheus)
- **Endpoint:** `http://localhost:8085/metrics`
- **Метрики:**
  - `wms_picking_tasks_total` - количество задач
  - `wms_picking_duration_seconds` - время сборки
  - `wms_packing_tasks_total`
  - `wms_inventory_movements_total`
  - `wms_worker_points_total`
  - `wms_grpc_requests_total`
  - `wms_grpc_errors_total`

## Диаграмма архитектуры

```
┌───────────────────┐
│  Listings Service │ (gRPC)
│   PORT: 50053     │
└─────────┬─────────┘
          │ Redis Streams
          ▼ (order.confirmed)
┌──────────────────────┐      gRPC      ┌────────────────┐
│   WMS Service        │◄───────────────►│ Delivery Svc   │
│   HTTP: 8085         │   CreateShipment│  PORT: 50052   │
│   gRPC: 50055        │                 └────────────────┘
└──────┬────┬──────────┘
       │    │
       │    └───────────► Redis (WMS)
       │                  PORT: 36381
       │                  Streams: picking, packing, inventory
       │
       ▼
┌──────────────────┐
│  PostgreSQL      │
│  PORT: 35435     │
│  DB: wms_db      │
│  Tables: 30+     │
└──────────────────┘

Workflows:
1. order.confirmed → PickingTask → PackingTask → Shipment
2. Returns → QC → Restock/Discard
3. Worker actions → Gamification points → Leaderboard
```

## Changelog

### v1.5.0 (2025-12-21)
- ✅ Gamification система (levels, points, leaderboard)
- ✅ Batch Picking (сборка нескольких заказов)
- ✅ Quality Control workflow
- ✅ Returns с фотографиями
- ✅ Stock Adjustments с approval
- ✅ Location Transfers
- ✅ Cycle Counts (инвентаризация)
- ✅ Substitutions (замены товаров)

### v1.4.0
- ✅ Owner Portal API
- ✅ Warehouse Staff & Invitations
- ✅ Redis Streams integration

### v1.3.0
- ✅ Packing workflow
- ✅ Integration с Delivery Service

### v1.2.0
- ✅ Picking workflow
- ✅ Worker authentication

### v1.1.0
- ✅ Inventory management (placement, reservations)

### v1.0.0
- ✅ Базовые CRUD для warehouses, zones, locations

---

## Phase 4: Neighborhood Express (2025-12-23)

### Обзор

2-часовая экспресс-доставка в пределах города (аналог Amazon Prime Now / Yandex Lavka).

### Новые компоненты

**Dark Stores:**
- Мини-склады в жилых районах
- 500-1000 топовых SKU на store
- Geo-location с PostGIS
- Покрытие: радиус 3km на store

**Express Orders:**
- SLA: 2 часа с момента заказа
- Auto-refund при опоздании
- Real-time tracking
- Background SLA monitoring

**Courier Tracking:**
- Real-time GPS positions
- ETA calculation (Haversine)
- Redis Pub/Sub для live updates
- TimescaleDB для time-series

**Geo-fencing:**
- Delivery zones (polygons)
- Priority-based routing
- Coverage optimization

### Database (PostGIS)

```sql
-- Dark Stores
dark_stores (
  id, code, name,
  latitude, longitude,
  location GEOGRAPHY(POINT, 4326),
  coverage_polygon GEOGRAPHY(POLYGON, 4326),
  capacity_orders_per_hour
)

-- Express Orders
express_orders (
  id, order_id, dark_store_id, courier_id,
  delivery_location GEOGRAPHY(POINT, 4326),
  sla_deadline, status
)

-- Courier Positions (TimescaleDB)
courier_positions (
  id, courier_id,
  location GEOGRAPHY(POINT, 4326),
  speed_kmh, timestamp
)
```

### gRPC API

```protobuf
service ExpressService {
  rpc CreateExpressOrder(...) returns (...);
  rpc GetOrderTracking(...) returns (...);
  rpc UpdateCourierPosition(...) returns (...);
  rpc GetSLAMetrics(...) returns (...);
}
```

### Metrics

- 95% заказов < 2 часа
- ETA точность: ±5 минут
- Dark store capacity: 20 orders/hour
- Coverage: 85% населения Belgrade

### Status

✅ PRODUCTION READY (реализовано 2025-12-23)

---

## Ссылки

- **Репозиторий:** /p/github.com/vondi-global/warehouse
- **Proto файл:** /p/github.com/vondi-global/warehouse/api/proto/warehouse.proto
- **Документация:**
  - EVENTS_IMPLEMENTATION.md
  - PROGRESS.md
  - HTTP_API_ENDPOINTS.md
  - BATCH_PICKING_IMPLEMENTATION.md
- **System Passport:** /p/github.com/vondi-global/SYSTEM_PASSPORT.md
