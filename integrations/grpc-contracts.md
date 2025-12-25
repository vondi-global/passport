# gRPC Contracts Passport

**Последнее обновление:** 2025-12-21
**Статус:** Production-ready (Auth, Listings, Delivery, WMS, Payment, Notifications, PVZ)

---

## 1. Архитектура gRPC взаимодействия

### 1.1 Обзор

Vondi Platform использует gRPC для межсервисной коммуникации. Все сервисы предоставляют как HTTP (REST/GraphQL), так и gRPC API.

**Принципы:**
- Protocol Buffers v3 для всех контрактов
- Строгая типизация и валидация
- Обратная совместимость через versioning (v1, v2)
- Circuit breaker и retry логика на клиентах
- OpenTelemetry трассировка для всех вызовов

### 1.2 Карта сервисов

| Сервис | gRPC порт | Package | Директория proto |
|--------|-----------|---------|------------------|
| Auth | 20053 | vondi.auth.v1 | `/auth/api/proto/auth/v1/` |
| Listings | 50053 | listingssvc.v1 | `/listings/api/proto/listings/v1/` |
| Delivery | 50052 | delivery.v1 | `/delivery/proto/delivery/v1/` |
| WMS | 50055 | wms | `/warehouse/api/proto/` |
| Payment | 50051 | payment.v1 | `/payment/proto/payment/v1/` |
| Notifications | 50054 | notify.v1 | `/notifications/api/proto/notify/v1/` |
| PVZ | 50056 | pvz.v1 | `/pvz/api/proto/pvz/v1/` |

### 1.3 Общие типы

**Pagination (общая для всех):**
```protobuf
message PaginationRequest {
  int32 page = 1;        // 1-based
  int32 page_size = 2;   // default: 20, max: 100
  int32 limit = 3;       // альтернатива
  int32 offset = 4;      // альтернатива
}

message PaginationResponse {
  int32 total_count = 1;
  int32 page = 2;
  int32 page_size = 3;
}
```

**Timestamps (google.protobuf.Timestamp):**
- Все даты в RFC3339 формате
- Хранятся в БД как TIMESTAMPTZ (UTC)

**Errors (стандарт gRPC):**
- `INVALID_ARGUMENT` - невалидные данные
- `NOT_FOUND` - ресурс не найден
- `PERMISSION_DENIED` - нет прав доступа
- `ALREADY_EXISTS` - дубликат
- `RESOURCE_EXHAUSTED` - превышен лимит (429)
- `UNAUTHENTICATED` - требуется аутентификация

---

## 2. Auth Service (vondi.auth.v1)

**Порт:** 20053
**Package:** `github.com/vondi-global/auth/pkg/grpc/proto/auth/v1;authv1`

### 2.1 AuthService (1 сервис, 26 методов)

| Метод | Описание | Важность |
|-------|----------|----------|
| **Authentication (6)** |
| `Register` | Регистрация пользователя | HIGH |
| `Login` | Вход пользователя | HIGH |
| `RefreshToken` | Обновление токена | HIGH |
| `ValidateToken` | Валидация JWT | CRITICAL |
| `Logout` | Выход (инвалидация токена) | MEDIUM |
| `LogoutAll` | Выход со всех устройств | LOW |
| **OAuth 2.0 (2)** |
| `GenerateOAuthURL` | URL для OAuth flow (Google) | HIGH |
| `ExchangeOAuthCode` | Обмен code на токены | HIGH |
| **User Management (7)** |
| `GetAllUsers` | Список пользователей (admin) | MEDIUM |
| `GetUser` | Получить пользователя по ID | HIGH |
| `GetUserByEmail` | Получить пользователя по email | HIGH |
| `UpdateUserProfile` | Обновить профиль | MEDIUM |
| `UpdateUserStatus` | Изменить статус (suspend/ban) | MEDIUM |
| `DeleteUser` | Удалить пользователя | LOW |
| `IsUserAdmin` | Проверка прав админа | HIGH |
| **Role Management (5)** |
| `GetUserRoles` | Роли пользователя | HIGH |
| `AddUserRole` | Добавить роль | MEDIUM |
| `RemoveUserRole` | Удалить роль | MEDIUM |
| `GetAllRoles` | Список всех ролей | LOW |
| `GetUsersByRole` | Пользователи с ролью | LOW |
| **Service Accounts (1)** |
| `ExchangeAPIKey` | S2S токен для сервисов | HIGH |
| **Worker Auth (1)** |
| `CreateWorkerToken` | JWT для WMS workers | HIGH |
| **Public Key (1)** |
| `GetPublicKey` | RSA публичный ключ для валидации JWT | CRITICAL |

### 2.2 Ключевые типы

**UserInfo:**
```protobuf
message UserInfo {
  int32 id = 1;
  string email = 2;
  string name = 3;
  optional string picture = 4;
  repeated string roles = 5;
  bool is_admin = 6;
  bool email_verified = 7;
  string provider = 8;  // local, google, github
  bool is_active = 9;
}
```

**JWT Claims (ValidateTokenResponse):**
```protobuf
message ValidateTokenResponse {
  bool valid = 1;
  optional int32 user_id = 3;
  optional string email = 4;
  repeated string roles = 5;
  optional int64 expires_at = 6;
  map<string, string> claims = 7;
}
```

**WorkerInfo (для WMS):**
```protobuf
message WorkerInfo {
  int64 id = 1;
  string code = 2;        // WRK-001
  string name = 3;
  int64 warehouse_id = 4;
  string role = 5;        // picker, packer, supervisor, manager
}
```

---

## 3. Listings Service (listingssvc.v1)

**Порт:** 50053
**Package:** `github.com/vondi-global/listings/api/proto/listings/v1;listingssvcv1`

### 3.1 ListingsService (1 сервис, 63+ методов)

**Core Listing Operations (7):**

| Метод | Input | Output | Описание |
|-------|-------|--------|----------|
| `GetListing` | GetListingRequest | GetListingResponse | Получить объявление |
| `CreateListing` | CreateListingRequest | CreateListingResponse | Создать объявление (C2C) |
| `UpdateListing` | UpdateListingRequest | UpdateListingResponse | Обновить объявление |
| `DeleteListing` | DeleteListingRequest | DeleteListingResponse | Удалить объявление |
| `SearchListings` | SearchListingsRequest | SearchListingsResponse | Поиск (OpenSearch) |
| `ListListings` | ListListingsRequest | ListListingsResponse | Фильтры + пагинация |
| `GetSimilarListings` | GetSimilarListingsRequest | GetSimilarListingsResponse | Похожие товары |

**Image Management (6):**

| Метод | Описание |
|-------|----------|
| `GetListingImage` | Получить изображение по ID |
| `DeleteListingImage` | Удалить изображение |
| `AddListingImage` | Добавить изображение |
| `GetListingImages` | Список изображений |
| `ReorderListingImages` | Изменить порядок отображения |
| `UploadListingImages` | Streaming загрузка (Phase 24) |

**B2C Product Operations (12):**

| Метод | Описание |
|-------|----------|
| `GetProduct`, `GetProductsBySKUs`, `GetProductsByIDs` | Чтение продуктов |
| `ListProducts` | Список продуктов storefront |
| `CreateProduct`, `UpdateProduct`, `DeleteProduct` | CRUD продуктов |
| `BulkCreateProducts`, `BulkUpdateProducts`, `BulkDeleteProducts` | Массовые операции |
| `IncrementProductViews` | Аналитика просмотров |
| `GetProductStats` | Статистика продуктов |

**Stock Management (6):**

| Метод | Описание | Критичность |
|-------|----------|-------------|
| `DecrementStock` | Атомарное списание стока | CRITICAL |
| `RollbackStock` | Откат при ошибке заказа | CRITICAL |
| `CheckStockAvailability` | Проверка доступности | HIGH |
| `CreateReservation` | Резервация для transfer | HIGH |
| `ReleaseReservation` | Освобождение резервации | HIGH |
| `CommitReservation` | Фиксация резервации | HIGH |

**Storefront Management (14):**

| Метод | Описание |
|-------|----------|
| `CreateStorefront`, `UpdateStorefront`, `DeleteStorefront` | CRUD витрин |
| `GetStorefront`, `GetStorefrontBySlug`, `ListStorefronts` | Чтение витрин |
| `GetMyStorefronts` | Витрины пользователя |
| `AddStaff`, `UpdateStaff`, `RemoveStaff`, `GetStaff` | Управление персоналом |
| `SetWorkingHours`, `GetWorkingHours`, `IsOpenNow` | Часы работы |
| `SetPaymentMethods`, `GetPaymentMethods` | Способы оплаты |
| `SetDeliveryOptions`, `GetDeliveryOptions` | Доставка |

**Storefront Invitations (8):**

| Метод | Описание |
|-------|----------|
| `CreateStorefrontEmailInvitation` | Email приглашение |
| `CreateStorefrontLinkInvitation` | Ссылка-приглашение |
| `ListStorefrontInvitations` | Список приглашений |
| `RevokeStorefrontInvitation` | Отозвать приглашение |
| `GetMyStorefrontInvitations` | Мои приглашения |
| `AcceptStorefrontInvitation` | Принять приглашение |
| `DeclineStorefrontInvitation` | Отклонить приглашение |
| `ValidateStorefrontInviteCode` | Валидация кода |
| `AcceptStorefrontInviteCode` | Принятие по коду |

**Other:**
- Categories (4): `GetRootCategories`, `GetAllCategories`, `GetPopularCategories`, `GetCategory`, `GetCategoryTree`
- Favorites (5): `AddToFavorites`, `RemoveFromFavorites`, `GetUserFavorites`, `IsFavorite`, `GetFavoritedUsers`
- Variants (4): `CreateVariants`, `GetVariants`, `UpdateVariant`, `DeleteVariant`
- Reindexing (3): `GetListingsForReindex`, `ResetReindexFlags`, `SyncDiscounts`, `ReindexAll`

### 3.2 OrderService (17 методов)

**Cart Operations (6):**

| Метод | Описание |
|-------|----------|
| `AddToCart` | Добавить в корзину |
| `UpdateCartItem` | Изменить количество/вариант |
| `RemoveFromCart` | Удалить из корзины |
| `GetCart` | Получить корзину |
| `ClearCart` | Очистить корзину |
| `GetUserCarts` | Все корзины пользователя |

**Order Operations (7):**

| Метод | Описание | Важность |
|-------|----------|----------|
| `CreateOrder` | Создать заказ из корзины | CRITICAL |
| `GetOrder` | Получить заказ | HIGH |
| `ListOrders` | Список заказов | HIGH |
| `CancelOrder` | Отменить заказ | HIGH |
| `UpdateOrderStatus` | Обновить статус (admin) | HIGH |
| `GetOrderStats` | Статистика заказов | MEDIUM |
| `UpdateOrderPayment` | Обновить payment info | HIGH |

**Shipment Workflow (4):**

| Метод | Описание |
|-------|----------|
| `AcceptOrder` | Продавец принял заказ |
| `CreateOrderShipment` | Создать отправку через Delivery Service |
| `MarkOrderShipped` | Отметить как отправлено |
| `GetOrderTracking` | Отслеживание доставки |

**Enums:**
```protobuf
enum OrderStatus {
  PENDING = 1;      // Создан, ждет оплаты
  CONFIRMED = 2;    // Оплачен
  ACCEPTED = 9;     // Продавец принял
  PROCESSING = 3;   // В обработке
  SHIPPED = 4;      // Отправлен
  DELIVERED = 5;    // Доставлен
  CANCELLED = 6;    // Отменен
  REFUNDED = 7;     // Возврат
  FAILED = 8;       // Ошибка
}
```

---

## 4. Delivery Service (delivery.v1)

**Порт:** 50052
**Package:** `github.com/vondi-global/delivery/gen/go/delivery/v1;deliveryv1`

### 4.1 DeliveryService (10 методов)

| Метод | Описание | Интеграция |
|-------|----------|------------|
| `CreateShipment` | Создать отправку | Post Express API |
| `GetShipment` | Получить отправку | БД + Provider |
| `TrackShipment` | Отследить отправку | Provider API |
| `CancelShipment` | Отменить отправку | Provider API |
| `CalculateRate` | Расчет стоимости | Provider API |
| `GetSettlements` | Список городов | Provider API |
| `GetStreets` | Список улиц | Provider API |
| `GetParcelLockers` | ПВЗ (BEX Lockers) | Provider API |
| `GetAvailableMethods` | Методы доставки для checkout | Multi-provider |
| `GetPickupPoints` | Точки выдачи | Multi-provider |

**Delivery Address Management (3):**

| Метод | Описание |
|-------|----------|
| `CreateDeliveryAddress` | Создать адрес доставки |
| `GetDeliveryAddress` | Получить адрес по ID |
| `GetDeliveryAddressByOrder` | Адрес по order_id |

**Enums:**
```protobuf
enum DeliveryProvider {
  POST_EXPRESS = 1;   // Post Express Serbia
  BEX_EXPRESS = 2;    // BEX Express
  AKS_EXPRESS = 3;    // AKS Express
  D_EXPRESS = 4;      // D Express
  CITY_EXPRESS = 5;   // City Express
  LOCAL = 6;          // Pickup / Seller delivery
}

enum ShipmentStatus {
  PENDING = 1;
  CONFIRMED = 2;
  IN_TRANSIT = 3;
  OUT_FOR_DELIVERY = 4;
  DELIVERED = 5;
  FAILED = 6;
  CANCELLED = 7;
  RETURNED = 8;
}
```

**DeliveryMethod (unified):**
```protobuf
message DeliveryMethod {
  string provider_id = 1;         // post_express, bex, local
  string method_type = 2;         // courier, pickup_point, standard
  string display_name = 3;
  double price = 4;
  int32 estimated_days_min = 6;
  int32 estimated_days_max = 7;
  bool supports_cod = 12;
  bool supports_insurance = 13;
  CostBreakdown cost_breakdown = 11;
}
```

---

## 5. WMS Service (wms)

**Порт:** 50055
**Package:** `github.com/vondi-global/warehouse/api/gen/wms`

### 5.1 Сервисы (12 сервисов)

**WarehouseService (5 методов):**

| Метод | Описание |
|-------|----------|
| `CreateWarehouse`, `GetWarehouse`, `UpdateWarehouse`, `DeleteWarehouse` | CRUD складов |
| `ListWarehouses` | Список складов |
| `CreateZone`, `GetZone`, `UpdateZone`, `DeleteZone`, `ListZones` | Управление зонами |
| `CreateLocation`, `GetLocation`, `UpdateLocation`, `DeleteLocation`, `ListLocations` | Ячейки хранения |

**InventoryService (11 методов):**

| Метод | Описание |
|-------|----------|
| `PlaceProduct` | Разместить товар в ячейке |
| `MoveProduct` | Переместить между ячейками |
| `RemoveProduct` | Удалить из ячейки |
| `GetAvailableQuantity` | Доступное количество |
| `GetProductLocations` | Где находится товар |
| `GetPickingLocations` | Откуда забирать (FIFO/FEFO) |
| `ReserveInventory`, `ReleaseReservation`, `CommitReservation` | Резервация |
| `CreateBatch`, `GetBatch`, `ListBatches` | Управление партиями (lot tracking) |

**PickingService (9 методов):**

| Метод | Описание |
|-------|----------|
| `CreatePickingTask` | Создать задачу на комплектацию |
| `GetPickingTask`, `GetPickingTaskByOrderID` | Получить задачу |
| `AssignPickingTask` | Назначить работнику |
| `StartPickingTask` | Начать выполнение |
| `PickItem` | Отметить позицию собранной |
| `CompletePickingTask` | Завершить |
| `CancelPickingTask` | Отменить |
| `ListPickingTasks` | Список задач |
| `GetNextItem` | Следующая позиция для сбора |

**PackingService (7 методов):**

| Метод | Описание |
|-------|----------|
| `CreatePackingTaskFromPicking` | Создать задачу упаковки |
| `GetPackingTask` | Получить задачу |
| `StartPackingTask` | Начать упаковку |
| `SetPackageDetails` | Указать вес/габариты |
| `PackItem` | Упаковать позицию |
| `CompletePackingTask` | Завершить |
| `CancelPackingTask`, `ListPackingTasks`, `PrintLabel` | Управление |

**WorkerService (9 методов + Gamification 3):**

| Метод | Описание |
|-------|----------|
| `AuthenticateWorker` | Вход работника (PIN) |
| `RefreshToken` | Обновить токен |
| `GetWorker`, `ListWorkers` | Информация о работниках |
| `CreateWorker`, `UpdateWorker`, `DeleteWorker` | CRUD |
| `GetWorkerStats` | Статистика работника |
| `UpdateWorkerStatus` | Обновить статус (active/on_break) |
| **Gamification** |
| `GetWorkerPoints` | Баллы работника |
| `GetPointTransactions` | История баллов |
| `GetLeaderboard` | Таблица лидеров |

**OwnerPortalService (14 методов + Invitations 10):**

| Метод | Описание |
|-------|----------|
| `GetMyWarehouses` | Склады владельца |
| `GetWarehouseAccess` | Проверка доступа |
| `GetOwnerDashboard` | Дашборд владельца |
| `AddWarehouseStaff`, `RemoveWarehouseStaff`, `ListWarehouseStaff`, `UpdateWarehouseStaffRole` | Персонал |
| **Invitations** |
| `CreateEmailInvitation`, `CreateLinkInvitation` | Создать приглашение |
| `ListInvitations`, `GetInvitation` | Список приглашений |
| `RevokeInvitation` | Отозвать приглашение |
| `GetMyInvitations` | Мои приглашения |
| `AcceptInvitation`, `DeclineInvitation` | Принять/отклонить |
| `ValidateInviteCode`, `AcceptInviteCode` | Приглашение по ссылке |

**Other Services:**
- `BatchPickingService` (10 методов) - Пакетная комплектация
- `QualityControlService` (7 методов) - Контроль качества
- `ReturnsService` (7 методов) - Обработка возвратов
- `StockAdjustmentService` (4 методов) - Корректировки остатков
- `LocationTransferService` (6 методов) - Перемещения между ячейками
- `CycleCountService` (7 методов) - Инвентаризация
- `SubstitutionService` (6 методов) - Замены товаров

**Enums (ключевые):**
```protobuf
enum PickingTaskStatus {
  PENDING = 1;
  ASSIGNED = 2;
  IN_PROGRESS = 3;
  COMPLETED = 4;
  CANCELLED = 5;
}

enum WorkerRole {
  PICKER = 1;
  PACKER = 2;
  SUPERVISOR = 3;
  MANAGER = 4;
}

enum LocationStatus {
  AVAILABLE = 1;
  OCCUPIED = 2;
  RESERVED = 3;
  MAINTENANCE = 4;
}
```

---

## 6. Payment Service (payment.v1)

**Порт:** 50051
**Package:** `github.com/vondi-global/payment/pkg/pb/payment/v1;paymentv1`

### 6.1 Сервисы (7 сервисов)

**PaymentService (4 методов):**

| Метод | Описание |
|-------|----------|
| `CreateCheckoutSession` | Создать платежную сессию |
| `GetPayment` | Получить платеж |
| `GetUserPayments` | Платежи пользователя |
| `ProcessWebhookEvent` | Обработка webhook от gateway |

**EscrowService (6 методов):**

| Метод | Описание |
|-------|----------|
| `GetEscrowHold`, `GetEscrowByPaymentID` | Получить escrow |
| `ListEscrows` | Список escrow |
| `ReleaseEscrowHold` | Освободить средства продавцу |
| `CancelEscrowHold` | Отменить escrow |
| `ProcessExpiredEscrows` | Batch обработка истекших |

**RefundService (3 методов):**

| Метод | Описание |
|-------|----------|
| `CreateRefund` | Создать возврат |
| `GetRefund` | Получить возврат |
| `GetPaymentRefunds` | Возвраты по платежу |

**ConnectedAccountService (4 методов):**

| Метод | Описание |
|-------|----------|
| `CreateConnectedAccount` | Stripe Connect аккаунт |
| `GetConnectedAccount` | Получить аккаунт |
| `CreateAccountLink` | Onboarding ссылка |
| `ListConnectedAccounts` | Список аккаунтов |

**TransferService (4 методов):**

| Метод | Описание |
|-------|----------|
| `CreateTransfer` | Перевод продавцу |
| `GetTransfer` | Получить перевод |
| `ListTransfers` | Список переводов |
| `ReverseTransfer` | Отменить перевод (полностью/частично) |

**BalanceService (2 методов):**

| Метод | Описание |
|-------|----------|
| `GetBalance` | Текущий баланс продавца |
| `GetBalanceHistory` | История баланса |

**PayoutService (4 методов):**

| Метод | Описание |
|-------|----------|
| `CreatePayout` | Вывод средств на банк |
| `GetPayout` | Получить payout |
| `ListPayouts` | Список payouts |
| `CancelPayout` | Отменить pending payout |

**SplitService (6 методов):**

| Метод | Описание |
|-------|----------|
| `CreateSplitRule`, `GetSplitRule`, `UpdateSplitRule`, `DeleteSplitRule` | CRUD правил комиссии |
| `ListSplitRules` | Список правил |
| `CalculateFee` | Расчет комиссии |

**Enums:**
```protobuf
enum PaymentStatus {
  PENDING = 1;
  PROCESSING = 2;
  SUCCEEDED = 3;
  FAILED = 4;
  CANCELLED = 5;
  REFUNDED = 6;
}

enum EscrowStatus {
  HOLDING = 1;     // Средства удерживаются
  RELEASED = 2;    // Отпущены продавцу
  CANCELLED = 3;   // Отменено (возврат покупателю)
}

enum PaymentType {
  PLATFORM_SERVICE = 1;   // Подписка, комиссия
  PRODUCT_B2C = 2;        // B2C покупка
  PRODUCT_C2C = 3;        // C2C покупка
  BALANCE_DEPOSIT = 4;    // Пополнение баланса
}
```

**Payment Flow:**
```protobuf
message Payment {
  string id = 1;
  string user_id = 2;
  string amount = 3;
  string currency = 4;
  PaymentStatus status = 5;
  PaymentType payment_type = 6;
  string gateway_name = 7;          // stripe, allsecure
  string gateway_session_id = 8;    // Checkout session
  string gateway_payment_intent_id = 9;
  string merchant_id = 10;          // Продавец
  string buyer_id = 11;             // Покупатель
  bool escrow_enabled = 12;
  int32 escrow_hold_period_days = 13;
}
```

---

## 7. Notifications Service (notify.v1)

**Порт:** 50054
**Package:** `github.com/vondi-global/notifications/api/gen/notify/v1;notifyv1`

### 7.1 Сервисы (3 сервиса)

**NotificationService (8 методов):**

| Метод | Описание |
|-------|----------|
| `SendNotification` | Отправить уведомление |
| `GetNotifications` | Список уведомлений |
| `GetUnreadCount` | Количество непрочитанных |
| `MarkAsRead` | Отметить прочитанным |
| `MarkAllAsRead` | Отметить все прочитанными |
| `DeleteNotification` | Удалить уведомление |

**SettingsService (6 методов):**

| Метод | Описание |
|-------|----------|
| `GetSettings` | Настройки пользователя |
| `UpdateSettings` | Обновить настройки |
| `GetTypeSetting` | Настройки для типа события |
| `UpdateTypeSetting` | Обновить настройки типа |
| `DeleteTypeSetting` | Сбросить к дефолту |

**TelegramService (4 методов):**

| Метод | Описание |
|-------|----------|
| `ConnectTelegram` | Инициировать подключение |
| `VerifyTelegramConnection` | Подтвердить подключение |
| `DisconnectTelegram` | Отключить Telegram |
| `GetTelegramConnection` | Статус подключения |

**Enums:**
```protobuf
enum Channel {
  EMAIL = 1;
  TELEGRAM = 2;
  SMS = 3;
  PUSH = 4;
  IN_APP = 5;
}

enum Priority {
  LOW = 1;
  MEDIUM = 2;
  HIGH = 3;
  CRITICAL = 4;
}

enum DeliveryStatus {
  PENDING = 1;
  QUEUED = 2;
  SENT = 3;
  DELIVERED = 4;
  FAILED = 5;
  SKIPPED = 6;
}

enum Locale {
  SR = 1;  // Serbian
  EN = 2;  // English
  RU = 3;  // Russian
}
```

**SendNotificationRequest:**
```protobuf
message SendNotificationRequest {
  int64 user_id = 1;
  string event_type = 2;          // order.created, order.shipped, etc.
  Locale locale = 3;
  map<string, string> variables = 4; // Template variables
  repeated Channel channels = 5;
  Priority priority = 6;
  map<string, string> metadata = 7;
  string email = 8;               // Override email
}
```

**Event Types (примеры):**
- `order.created` - Заказ создан
- `order.confirmed` - Заказ подтвержден
- `order.shipped` - Заказ отправлен
- `order.delivered` - Заказ доставлен
- `payment.success` - Оплата успешна
- `payment.failed` - Оплата не прошла
- `listing.approved` - Объявление одобрено
- `listing.rejected` - Объявление отклонено

---

## 8. PVZ Service (pvz.v1)

**Порт:** 50056
**Package:** `github.com/vondi-global/pvz/api/proto/pvz/v1;pvzv1`
**Назначение:** Управление сетью пунктов выдачи заказов (ПВЗ), партнёрская программа

### 8.1 PVZService (9 методов)

| Метод | Описание | Важность |
|-------|----------|----------|
| **PVZ Management (3)** |
| `GetPVZ` | Информация о ПВЗ | HIGH |
| `ListPVZ` | Список ПВЗ с фильтрацией | HIGH |
| `FindNearestPVZ` | Поиск ближайших ПВЗ (geo) | CRITICAL |
| **Capacity (2)** |
| `GetPVZCapacity` | Текущая загрузка ПВЗ | HIGH |
| `ReserveSlot` | Резервирование места | HIGH |
| **Parcels (4)** |
| `GetParcelAtPVZ` | Информация о посылке в ПВЗ | HIGH |
| `ListParcelsAtPVZ` | Посылки в конкретном ПВЗ | MEDIUM |
| `ValidatePickupCode` | Проверка кода выдачи | CRITICAL |
| `GetPickupStatus` | Статус посылки в ПВЗ | MEDIUM |

**Примеры запросов:**

```protobuf
// FindNearestPVZ - геопоиск
message FindNearestPVZRequest {
  double latitude = 1;   // 44.8125
  double longitude = 2;  // 20.4612
  int32 limit = 3;       // default: 5
  bool supports_pickup = 4;
  bool supports_return = 5;
}

message FindNearestPVZResponse {
  repeated PVZLocation pvz_list = 1;
}

message PVZLocation {
  int64 id = 1;
  string code = 2;        // PVZ-BG-001
  string name = 3;
  string type = 4;        // full | mikro
  string address = 5;
  double latitude = 6;
  double longitude = 7;
  double distance_km = 8;
  int32 current_parcels = 9;
  int32 max_capacity = 10;
  bool is_available = 11;
}

// ValidatePickupCode - критический метод для выдачи
message ValidatePickupCodeRequest {
  int64 pvz_id = 1;
  string pickup_code = 2;  // 123456 (6 цифр)
  int64 customer_id = 3;   // опционально для доп. проверки
}

message ValidatePickupCodeResponse {
  bool is_valid = 1;
  string parcel_id = 2;
  int64 order_id = 3;
  int32 attempts_left = 4;  // осталось попыток (max 5)
  google.protobuf.Timestamp expiry = 5;
}
```

### 8.2 PVZOperatorService (14 методов)

| Метод | Описание | Важность |
|-------|----------|----------|
| **Shifts (2)** |
| `StartShift` | Начать смену оператора | HIGH |
| `EndShift` | Завершить смену | HIGH |
| **Handover (4)** |
| `ListPendingHandovers` | Ожидающие приёмки от курьеров | HIGH |
| `AcceptHandover` | Принять посылку (QR scan) | CRITICAL |
| `RejectHandover` | Отклонить посылку | MEDIUM |
| `GetHandoverDetails` | Детали передачи | LOW |
| **Pickup (4)** |
| `ValidateAndPickup` | Проверка кода + выдача | CRITICAL |
| `CompletePickup` | Завершение выдачи | HIGH |
| `ListTodayPickups` | Выдачи за сегодня | MEDIUM |
| `GetShiftStats` | Статистика смены | MEDIUM |
| **Returns (2)** |
| `InitiateReturn` | Инициировать возврат | HIGH |
| `ProcessReturn` | Обработать возврат | HIGH |
| **Inventory (2)** |
| `GetPVZInventory` | Текущие посылки в ПВЗ | MEDIUM |
| `SearchParcel` | Поиск посылки по ID/код | MEDIUM |

**Примеры запросов:**

```protobuf
// AcceptHandover - критический метод
message AcceptHandoverRequest {
  int64 pvz_id = 1;
  int64 operator_id = 2;
  string qr_code = 3;           // отсканированный QR
  repeated string photo_urls = 4; // фото посылки
  string notes = 5;
}

message AcceptHandoverResponse {
  int64 handover_id = 1;
  string parcel_id = 2;
  int64 order_id = 3;
  string pickup_code = 4;       // сгенерированный код выдачи
  google.protobuf.Timestamp pickup_code_expiry = 5;
}

// CompletePickup - завершение выдачи
message CompletePickupRequest {
  int64 pvz_id = 1;
  int64 operator_id = 2;
  string parcel_id = 3;
  string pickup_code = 4;
  int64 customer_id = 5;
  string payment_method = 6;    // cash | card | wallet | paid_online
  double cod_amount = 7;        // если COD
}

message CompletePickupResponse {
  int64 pickup_event_id = 1;
  google.protobuf.Timestamp pickup_time = 2;
  bool payment_collected = 3;
}
```

### 8.3 PartnerService (10 методов)

| Метод | Описание | Важность |
|-------|----------|----------|
| **Registration (3)** |
| `RegisterPartner` | Регистрация партнёра | HIGH |
| `GetPartnerApplication` | Статус заявки | MEDIUM |
| `UpdatePartnerProfile` | Обновить профиль | MEDIUM |
| **Financials (4)** |
| `GetPartnerEarnings` | Заработки партнёра | HIGH |
| `GetPayoutHistory` | История выплат | MEDIUM |
| `RequestPayout` | Запросить выплату | HIGH |
| `GetCommissions` | Комиссии за период | MEDIUM |
| **PVZ Management (3)** |
| `GetPartnerPVZ` | ПВЗ партнёра | HIGH |
| `UpdatePVZInfo` | Обновить инфо о ПВЗ | MEDIUM |
| `SetPVZStatus` | Установить статус (closed/active) | MEDIUM |

**Примеры запросов:**

```protobuf
message RegisterPartnerRequest {
  int64 user_id = 1;
  string business_name = 2;
  string business_type = 3;     // cafe | shop | salon | pharmacy | gas_station
  string contact_name = 4;
  string phone = 5;
  string email = 6;
  string address = 7;
  double latitude = 8;
  double longitude = 9;
  string tax_id = 10;
  string bank_account = 11;
}

message GetPartnerEarningsResponse {
  double total_earnings = 1;
  double pending_earnings = 2;
  double paid_earnings = 3;
  int32 total_pickups = 4;
  double commission_rate = 5;   // 65.00 = 65%
}
```

### 8.4 Интеграции с другими сервисами

**PVZ → Delivery:**
- Подписка на `delivery.parcel.routed_to_pvz` (Redis Stream)
- Создание записи в `parcels_at_pvz`
- Генерация pickup кода

**PVZ → Payment:**
- `ProcessCODPayment` - обработка оплаты при получении
- `CalculateStorageFee` - расчёт платы за хранение
- `ProcessPartnerPayout` - выплата комиссий партнёрам

**PVZ → Notifications:**
- Отправка SMS с pickup кодом
- Email уведомления о прибытии посылки
- Telegram уведомления операторам

**PVZ → Auth:**
- `ValidateToken` - проверка JWT оператора
- `GetUser` - получение данных пользователя
- `CheckPermission` - проверка прав доступа

---

## 9. Интеграционные сценарии

### 9.1 Создание заказа (End-to-End)

```
1. Frontend → Listings.CreateOrder(cart_id)
   ↓
2. Listings Service:
   - Валидация корзины
   - Резервация стока (CreateReservation)
   - Создание Order (status=PENDING)
   ↓
3. Listings → Payment.CreateCheckoutSession(order_id)
   ↓
4. Payment Service:
   - Создание Payment (status=PENDING)
   - Создание Stripe session
   - Возврат checkout_url
   ↓
5. User оплачивает → Stripe webhook
   ↓
6. Payment.ProcessWebhookEvent
   - Payment.status = SUCCEEDED
   - Order.payment_status = COMPLETED
   ↓
7. Payment → Listings.UpdateOrderStatus(CONFIRMED)
   ↓
8. Listings Service:
   - CommitReservation (списание стока)
   - Order.status = CONFIRMED
   ↓
9. Listings → Notifications.SendNotification(order.created)
   ↓
10. Notifications → Email + Telegram
```

### 8.2 Создание отправки (Seller workflow)

```
1. Seller → Listings.AcceptOrder(order_id)
   - Order.status = ACCEPTED
   ↓
2. Seller → Listings.CreateOrderShipment(order_id, provider, package_info)
   ↓
3. Listings → Delivery.CreateShipment(...)
   ↓
4. Delivery Service:
   - Provider API (Post Express / BEX)
   - Создание Shipment
   - Получение tracking_number + label_url
   ↓
5. Delivery → Listings (return)
   - Order.shipment_id = ...
   - Order.tracking_number = ...
   - Order.label_url = ...
   - Order.status = PROCESSING
   ↓
6. Seller → Listings.MarkOrderShipped(order_id)
   - Order.status = SHIPPED
   ↓
7. Listings → Notifications.SendNotification(order.shipped)
```

### 8.3 WMS Picking + Packing

```
1. Order.status = CONFIRMED
   ↓
2. WMS.CreatePickingTask(order_id)
   - Получение picking locations (FIFO)
   - Создание PickingTask + PickingTaskItems
   ↓
3. Worker → WMS.AssignPickingTask(task_id, worker_id)
   ↓
4. Worker → WMS.StartPickingTask(task_id)
   ↓
5. Worker → WMS.PickItem(task_id, item_id, quantity)
   - Списание с location
   - Gamification: +10 points
   ↓
6. Worker → WMS.CompletePickingTask(task_id)
   - Auto-create PackingTask
   ↓
7. Worker → WMS.StartPackingTask(task_id)
   ↓
8. Worker → WMS.SetPackageDetails(weight, dimensions)
   ↓
9. Worker → WMS.CompletePackingTask(task_id, tracking_number)
   - Gamification: +50 points
   ↓
10. WMS → Listings.UpdateOrderStatus(PROCESSING)
```

---

## 9. Версионирование и совместимость

### 9.1 Правила версионирования

- **Пакеты версионируются:** `package_name.v1`, `package_name.v2`
- **Обратная совместимость:** Добавление полей (не удаление)
- **Breaking changes:** Новая мажорная версия (v2)
- **Deprecation:** Минимум 3 месяца warning

### 9.2 Текущие версии

| Сервис | Версия | Статус |
|--------|--------|--------|
| Auth | v1 | Stable |
| Listings | v1 | Stable |
| Delivery | v1 | Stable |
| WMS | v1 | Stable |
| Payment | v1 | Stable |
| Notifications | v1 | Beta |

---

## 10. Производительность и лимиты

| Сервис | Timeout | Max request size | Rate limit |
|--------|---------|------------------|------------|
| Auth | 5s | 1 MB | 100 req/s |
| Listings | 10s | 10 MB | 200 req/s |
| Delivery | 15s | 5 MB | 50 req/s |
| WMS | 10s | 5 MB | 100 req/s |
| Payment | 30s | 2 MB | 50 req/s |
| Notifications | 5s | 1 MB | 100 req/s |

**Retry логика (все сервисы):**
- Exponential backoff: 100ms, 200ms, 400ms, 800ms
- Max retries: 3
- Idempotency keys для POST/PUT/DELETE

**Circuit Breaker:**
- Failure threshold: 5 errors in 10s
- Timeout: 30s
- Half-open after: 60s

---

## 11. Мониторинг

**Метрики (Prometheus):**
- `grpc_server_handled_total{service, method, code}` - Счетчик вызовов
- `grpc_server_handling_seconds{service, method}` - Время ответа
- `grpc_server_msg_received_total` - Входящие сообщения
- `grpc_server_msg_sent_total` - Исходящие сообщения

**Трассировка (OpenTelemetry):**
- Все gRPC вызовы автоматически трассируются
- Trace ID передается через metadata
- Экспорт в Jaeger

**Логирование:**
- Structured JSON (zerolog)
- Уровни: DEBUG, INFO, WARN, ERROR
- Поля: trace_id, user_id, request_id, method

---

**Конец документа**

Для получения подробной информации о конкретном сервисе см. соответствующие proto файлы:
- Auth: `/auth/api/proto/auth/v1/auth.proto`
- Listings: `/listings/api/proto/listings/v1/*.proto`
- Delivery: `/delivery/proto/delivery/v1/delivery.proto`
- WMS: `/warehouse/api/proto/warehouse.proto`
- Payment: `/payment/proto/payment/v1/payment.proto`
- Notifications: `/notifications/api/proto/notify/v1/notification.proto`
