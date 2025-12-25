# Authorization System - System Passport

**Версия:** 1.0
**Дата обновления:** 2025-12-21
**Микросервис:** Auth Service (port 28086 HTTP, 20053 gRPC)
**Ответственный:** Malden Simić

---

## 1. Обзор системы авторизации

Vondi использует **Role-Based Access Control (RBAC)** с трехуровневой проверкой доступа:

```
1. Role Check      → Пользователь имеет требуемую роль?
2. Permission Check → Роль имеет требуемое разрешение?
3. Ownership Check  → Пользователь владеет ресурсом? (для *_own операций)
```

### Архитектура

```
┌─────────────┐
│   User      │
│  ID: 123    │
│  Roles: []  │
└──────┬──────┘
       │
       │ has many
       ▼
┌─────────────┐
│    Role     │
│  "admin"    │
│  "seller"   │
└──────┬──────┘
       │
       │ has many
       ▼
┌─────────────────┐
│   Permission    │
│ "listings.edit" │
│ "orders.view"   │
└─────────────────┘
```

### Ключевые принципы

1. **Множественные роли**: Пользователь может иметь несколько ролей одновременно
2. **Иерархия приоритетов**: super_admin (1) → admin (10) → moderator (20) → ... → user (100)
3. **Разделение разрешений**: 67 разрешений в 12 категориях
4. **Protocol-agnostic**: Работает через HTTP и gRPC
5. **Ownership**: Разграничение *_own vs *_any операций

---

## 2. Системные роли (28 ролей)

### 2.1 Core Administrative Roles

| Роль | Константа | Приоритет | Описание |
|------|-----------|-----------|----------|
| **super_admin** | `RoleSuperAdmin` | 1 | Полный доступ к системе, включая критические операции |
| **admin** | `RoleAdmin` | 10 | Административный доступ к платформе |

**Разрешения:**
- super_admin: `system.manage_settings`, `system.manage_roles`, `system.view_audit`, `system.maintenance`, `system.backup`
- admin: `admin.access`, `admin.users`, `admin.categories`, `admin.reports`, `users.view`, `users.list`, `users.edit`

---

### 2.2 Moderation Roles

| Роль | Константа | Приоритет | Специализация |
|------|-----------|-----------|---------------|
| **moderator** | `RoleModerator` | 20 | Общая модерация (объявления + отзывы + чат) |
| **content_moderator** | `RoleContentModerator` | 20 | Модерация только объявлений |
| **review_moderator** | `RoleReviewModerator` | 20 | Модерация только отзывов |
| **chat_moderator** | `RoleChatModerator` | 20 | Модерация только сообщений |
| **dispute_manager** | `RoleDisputeManager` | 30 | Управление спорами и возвратами |

**Разрешения:**
- moderator: `listings.moderate`, `reviews.moderate`, `messaging.moderate`
- content_moderator: `listings.moderate`
- review_moderator: `reviews.moderate`
- chat_moderator: `messaging.moderate`
- dispute_manager: `orders.view_all`, `orders.refund`

---

### 2.3 Business Management Roles

| Роль | Константа | Приоритет | Зона ответственности |
|------|-----------|-----------|----------------------|
| **vendor_manager** | `RoleVendorManager` | 30 | Управление продавцами |
| **category_manager** | `RoleCategoryManager` | 30 | Управление категориями |
| **marketing_manager** | `RoleMarketingManager` | 30 | Маркетинг и промоакции |
| **financial_manager** | `RoleFinancialManager` | 30 | Финансовая отчетность |

**Разрешения:**
- Все: `listings.view_all`, `orders.view_all`
- Дополнительно зависят от специализации (см. раздел 3)

---

### 2.4 Operations Roles (Warehouse & Logistics)

| Роль | Константа | Приоритет | Функционал |
|------|-----------|-----------|------------|
| **warehouse_manager** | `RoleWarehouseManager` | 30 | Управление складом |
| **warehouse_worker** | `RoleWarehouseWorker` | 100 | Складские операции |
| **pickup_manager** | `RolePickupManager` | 30 | Управление пунктами выдачи |
| **pickup_worker** | `RolePickupWorker` | 100 | Работа в ПВЗ |
| **courier** | `RoleCourier` | 100 | Доставка заказов |

**Разрешения:**
- warehouse_*: `warehouse.view_stock`, `warehouse.update_stock`, `warehouse.manage_returns`
- pickup_*: `pickup.manage_points`, `pickup.receive_items`, `pickup.release_items`
- courier: `delivery.assign_routes`, `delivery.update_status`, `delivery.view_routes`

---

### 2.5 Support Roles

| Роль | Константа | Приоритет | Уровень поддержки |
|------|-----------|-----------|-------------------|
| **support_l1** | `RoleSupportL1` | 40 | Первая линия (базовые вопросы) |
| **support_l2** | `RoleSupportL2` | 40 | Вторая линия (технические проблемы) |
| **support_l3** | `RoleSupportL3` | 40 | Третья линия (эскалация) |

**Разрешения:**
- Все уровни: `support.view_tickets`, `support.resolve_tickets`
- L3: дополнительно `support.escalate`

---

### 2.6 Compliance & Legal Roles

| Роль | Константа | Приоритет | Функционал |
|------|-----------|-----------|------------|
| **legal_advisor** | `RoleLegalAdvisor` | 30 | Юридические консультации |
| **compliance_officer** | `RoleComplianceOfficer` | 30 | Соблюдение требований KYC/AML |

**Разрешения:**
- legal_advisor: `users.view`
- compliance_officer: `users.view`, `compliance.view_reports`, `compliance.manage_kyc`

---

### 2.7 Vendor & Seller Roles

| Роль | Константа | Приоритет | Тип продавца |
|------|-----------|-----------|--------------|
| **professional_vendor** | `RoleProfessionalVendor` | 50 | Профессиональный продавец (юрлицо) |
| **vendor** | `RoleVendor` | 50 | Обычный продавец |
| **individual_seller** | `RoleIndividualSeller` | 50 | Частный продавец |
| **storefront_staff** | `RoleStorefrontStaff` | 50 | Персонал магазина |

**Разрешения:**
- vendor/professional_vendor/individual_seller: `listings.create`, `listings.edit_own`, `listings.delete_own`, `orders.view_own`
- storefront_staff: `listings.view_all`, `orders.view_own` (только для своего магазина)

---

### 2.8 Customer Roles

| Роль | Константа | Приоритет | Тип покупателя |
|------|-----------|-----------|----------------|
| **verified_buyer** | `RoleVerifiedBuyer` | 60 | Верифицированный покупатель |
| **vip_customer** | `RoleVIPCustomer` | 60 | VIP клиент (особые условия) |
| **user** | `RoleUser` | 100 | Базовый пользователь |

**Разрешения:**
- Все: `orders.view_own`
- Дополнительные привилегии VIP настраиваются отдельно

---

### 2.9 Service Account Roles

| Роль | Константа | Приоритет | Назначение |
|------|-----------|-----------|------------|
| **service** | `RoleService` | 100 | Базовая роль для service-to-service auth |

**Особенности:**
- Без явных разрешений (авторизация через Service JWT tokens)
- Используется для микросервисов (listings, delivery, payment, etc.)
- Аутентификация через отдельный механизм Service Accounts

---

### 2.10 Analytics Roles

| Роль | Константа | Приоритет | Функционал |
|------|-----------|-----------|------------|
| **data_analyst** | `RoleDataAnalyst` | 30 | Анализ данных |
| **business_analyst** | `RoleBusinessAnalyst` | 30 | Бизнес-аналитика |

**Разрешения:**
- Оба: `analytics.view_reports`, `analytics.view_dashboard`, `analytics.export_data`

---

## 3. Разрешения (67 permissions)

### 3.1 User Management (8 permissions)

| Разрешение | Константа | Описание |
|------------|-----------|----------|
| `users.view` | `PermUsersView` | Просмотр профилей пользователей |
| `users.list` | `PermUsersList` | Список всех пользователей |
| `users.edit` | `PermUsersEdit` | Редактирование профилей |
| `users.delete` | `PermUsersDelete` | Удаление пользователей |
| `users.block` | `PermUsersBlock` | Блокировка пользователей |
| `users.verify` | `PermUsersVerify` | Верификация аккаунтов |
| `users.assign_role` | `PermUsersAssignRole` | Назначение ролей |
| `users.export` | `PermUsersExport` | Экспорт данных пользователей |

**Кто имеет:** admin, super_admin

---

### 3.2 Admin Panel (7 permissions)

| Разрешение | Константа | Раздел админки |
|------------|-----------|----------------|
| `admin.access` | `PermAdminAccess` | Доступ к админ-панели |
| `admin.users` | `PermAdminUsers` | Управление пользователями |
| `admin.categories` | `PermAdminCategories` | Управление категориями |
| `admin.attributes` | `PermAdminAttributes` | Управление атрибутами |
| `admin.translations` | `PermAdminTranslations` | Управление переводами |
| `admin.reports` | `PermAdminReports` | Просмотр отчетов |
| `admin.settings` | `PermAdminSettings` | Системные настройки |

**Кто имеет:** admin (все кроме settings), super_admin (все)

---

### 3.3 Listing Management (10 permissions)

| Разрешение | Константа | Ownership | Описание |
|------------|-----------|-----------|----------|
| `listings.create` | `PermListingsCreate` | - | Создание объявлений |
| `listings.edit_own` | `PermListingsEditOwn` | ✅ Own | Редактирование своих объявлений |
| `listings.edit_any` | `PermListingsEditAny` | ❌ Any | Редактирование любых объявлений |
| `listings.delete_own` | `PermListingsDeleteOwn` | ✅ Own | Удаление своих объявлений |
| `listings.delete_any` | `PermListingsDeleteAny` | ❌ Any | Удаление любых объявлений |
| `listings.moderate` | `PermListingsModerate` | - | Модерация объявлений |
| `listings.view_all` | `PermListingsViewAll` | - | Просмотр всех объявлений |
| `listings.approve` | `PermListingsApprove` | - | Одобрение объявлений |
| `listings.reject` | `PermListingsReject` | - | Отклонение объявлений |
| `listings.feature` | `PermListingsFeature` | - | Выделение объявлений |

**Кто имеет:**
- *_own: vendor, professional_vendor, individual_seller
- *_any: admin, content_moderator
- moderate: moderator, content_moderator

---

### 3.4 Order Management (7 permissions)

| Разрешение | Константа | Ownership | Описание |
|------------|-----------|-----------|----------|
| `orders.view_all` | `PermOrdersViewAll` | ❌ Any | Просмотр всех заказов |
| `orders.view_own` | `PermOrdersViewOwn` | ✅ Own | Просмотр своих заказов |
| `orders.process` | `PermOrdersProcess` | - | Обработка заказов |
| `orders.cancel` | `PermOrdersCancel` | - | Отмена заказов |
| `orders.refund` | `PermOrdersRefund` | - | Возврат средств |
| `orders.export` | `PermOrdersExport` | - | Экспорт данных заказов |
| `orders.ship` | `PermOrdersShip` | - | Отправка заказов |

**Кто имеет:**
- view_own: все пользователи
- view_all: admin, vendor_manager, financial_manager, dispute_manager

---

### 3.5 Payment Permissions (5 permissions)

| Разрешение | Константа | Описание |
|------------|-----------|----------|
| `payments.view` | `PermPaymentsView` | Просмотр платежей |
| `payments.process` | `PermPaymentsProcess` | Обработка платежей |
| `payments.refund` | `PermPaymentsRefund` | Оформление возвратов |
| `payments.withdraw` | `PermPaymentsWithdraw` | Вывод средств |
| `payments.view_reports` | `PermPaymentsViewReports` | Отчеты по платежам |

**Кто имеет:** financial_manager, admin

---

### 3.6 Messaging Permissions (3 permissions)

| Разрешение | Константа | Описание |
|------------|-----------|----------|
| `messaging.view` | `PermMessagingView` | Просмотр сообщений |
| `messaging.moderate` | `PermMessagingModerate` | Модерация чатов |
| `messaging.broadcast` | `PermMessagingBroadcast` | Массовая рассылка |

**Кто имеет:**
- moderate: moderator, chat_moderator
- broadcast: marketing_manager

---

### 3.7 Review & Rating Permissions (3 permissions)

| Разрешение | Константа | Описание |
|------------|-----------|----------|
| `reviews.moderate` | `PermReviewsModerate` | Модерация отзывов |
| `reviews.delete` | `PermReviewsDelete` | Удаление отзывов |
| `reviews.respond` | `PermReviewsRespond` | Ответы на отзывы |

**Кто имеет:**
- moderate: moderator, review_moderator
- respond: vendor (для своих отзывов)

---

### 3.8 Storefront Management (5 permissions)

| Разрешение | Константа | Описание |
|------------|-----------|----------|
| `storefront.create` | `PermStorefrontCreate` | Создание магазина |
| `storefront.edit` | `PermStorefrontEdit` | Редактирование магазина |
| `storefront.delete` | `PermStorefrontDelete` | Удаление магазина |
| `storefront.verify` | `PermStorefrontVerify` | Верификация магазина |
| `storefront.suspend` | `PermStorefrontSuspend` | Приостановка магазина |

**Кто имеет:**
- create/edit: vendor, professional_vendor
- verify/suspend: admin, vendor_manager

---

### 3.9 Warehouse Management (4 permissions)

| Разрешение | Константа | Описание |
|------------|-----------|----------|
| `warehouse.view_stock` | `PermWarehouseViewStock` | Просмотр остатков |
| `warehouse.update_stock` | `PermWarehouseUpdateStock` | Обновление остатков |
| `warehouse.manage_returns` | `PermWarehouseManageReturns` | Управление возвратами |
| `warehouse.ship_orders` | `PermWarehouseShipOrders` | Отгрузка заказов |

**Кто имеет:** warehouse_manager, warehouse_worker

---

### 3.10 Pickup Point Permissions (3 permissions)

| Разрешение | Константа | Описание |
|------------|-----------|----------|
| `pickup.manage_points` | `PermPickupManagePoints` | Управление ПВЗ |
| `pickup.receive_items` | `PermPickupReceiveItems` | Прием товаров |
| `pickup.release_items` | `PermPickupReleaseItems` | Выдача товаров |

**Кто имеет:** pickup_manager, pickup_worker

---

### 3.11 Delivery Permissions (3 permissions)

| Разрешение | Константа | Описание |
|------------|-----------|----------|
| `delivery.assign_routes` | `PermDeliveryAssignRoutes` | Назначение маршрутов |
| `delivery.update_status` | `PermDeliveryUpdateStatus` | Обновление статуса доставки |
| `delivery.view_routes` | `PermDeliveryViewRoutes` | Просмотр маршрутов |

**Кто имеет:** courier

---

### 3.12 System Permissions (6 permissions)

| Разрешение | Константа | Критичность | Описание |
|------------|-----------|-------------|----------|
| `system.view_logs` | `PermSystemViewLogs` | 🟡 Medium | Просмотр логов |
| `system.manage_settings` | `PermSystemManageSettings` | 🔴 High | Системные настройки |
| `system.manage_roles` | `PermSystemManageRoles` | 🔴 High | Управление RBAC |
| `system.view_audit` | `PermSystemViewAudit` | 🟡 Medium | Аудит действий |
| `system.maintenance` | `PermSystemMaintenance` | 🔴 Critical | Режим обслуживания |
| `system.backup` | `PermSystemBackup` | 🔴 Critical | Резервное копирование |

**Кто имеет:** ТОЛЬКО super_admin

---

## 4. Role-Permission Mapping

### Полная матрица разрешений

```go
// Из pkg/entity/roles_example.go: GetPermissionsByRole()

RoleSuperAdmin → [
  system.manage_settings, system.manage_roles, system.view_audit,
  system.maintenance, system.backup
]

RoleAdmin → [
  admin.access, admin.users, admin.categories, admin.reports,
  users.view, users.list, users.edit
]

RoleModerator → [
  listings.moderate, reviews.moderate, messaging.moderate
]

RoleVendor → [
  listings.create, listings.edit_own, listings.delete_own,
  orders.view_own
]

// ... и так далее для всех 28 ролей
```

**ВАЖНО:** Функция `GetPermissionsByRole()` возвращает базовый набор разрешений. В production можно расширить через database (`role_permissions` таблица).

---

## 5. Ownership Проверки

### Механизм проверки владения ресурсом

```go
// Пример: редактирование объявления

1. Role Check: user has "vendor" role? ✅
2. Permission Check: "vendor" has "listings.edit_own"? ✅
3. Ownership Check: listing.seller_id == user.id? ✅

// Если все 3 проверки пройдены → доступ разрешен
```

### Разрешения с ownership

| Операция | Permission (own) | Permission (any) |
|----------|------------------|------------------|
| Редактирование объявлений | `listings.edit_own` | `listings.edit_any` |
| Удаление объявлений | `listings.delete_own` | `listings.delete_any` |
| Просмотр заказов | `orders.view_own` | `orders.view_all` |

### Middleware для ownership

```go
// internal/transport/http/middleware/auth.go

// RequireSelfOrAdmin - проверка ownership через URL параметр
func RequireSelfOrAdmin() fiber.Handler {
    return func(c *fiber.Ctx) error {
        authenticatedUserID := c.Locals("user_id").(int)
        targetUserID := c.Params("id") // из /users/:id

        // Разрешить если:
        // 1. Пользователь редактирует себя (ownership)
        if authenticatedUserID == targetUserID {
            return c.Next()
        }

        // 2. Пользователь - админ (bypass ownership)
        if IsAdmin(c) {
            return c.Next()
        }

        // Логирование IDOR попытки
        logger.Warn().Msg("IDOR attempt blocked")
        return c.Status(403).JSON(fiber.Map{"error": "forbidden"})
    }
}
```

**IDOR Protection:** Все попытки доступа к чужим ресурсам логируются для security audit.

---

## 6. Middleware интеграция

### 6.1 HTTP Middleware (Fiber Framework)

#### Публичная библиотека (для внешних приложений)

```go
// Из pkg/http/fiber/middleware.go

import (
    "github.com/vondi-global/auth/pkg/service"
    fibermw "github.com/vondi-global/auth/pkg/http/fiber"
)

authService := service.NewAuthService(&service.Config{
    HTTPURL: "http://auth-service:28086",
    GRPCURL: "auth-service:20053",
})

// 1. ExtractUser (опциональная авторизация)
app.Use(fibermw.ExtractUser(authService))

// 2. RequireAuth (обязательная авторизация)
protected := app.Group("/api/protected")
protected.Use(fibermw.RequireAuth(authService))

// 3. RequireRole (проверка ролей)
admin := app.Group("/api/admin")
admin.Use(fibermw.RequireAuth(authService))
admin.Use(fibermw.RequireRole("admin", "super_admin"))

// Получение userInfo в хендлере
func handler(c *fiber.Ctx) error {
    userInfo := fibermw.GetUserInfo(c)
    return c.JSON(fiber.Map{
        "user_id": userInfo.ID,
        "email":   userInfo.Email,
        "roles":   userInfo.Roles,
    })
}
```

#### Внутренний middleware (для Auth Service)

```go
// Из internal/transport/http/middleware/auth.go

import (
    "github.com/vondi-global/auth/internal/transport/http/middleware"
)

// 1. JWTAuth - валидация токена (internal)
app.Use(middleware.JWTAuth(jwtService))

// 2. RequireAuth - обязательная аутентификация
app.Use(middleware.RequireAuth())

// 3. RequireRole - проверка ролей
app.Use(middleware.RequireRole(entity.RoleAdmin))

// 4. RequireSelfOrAdmin - ownership проверка
app.Put("/users/:id",
    middleware.RequireAuth(),
    middleware.RequireSelfOrAdmin(),
    updateUserHandler,
)
```

---

### 6.2 gRPC Interceptors

```go
// Из internal/transport/grpc/server.go

import (
    grpc_middleware "github.com/grpc-ecosystem/go-grpc-middleware"
    "github.com/vondi-global/auth/internal/transport/grpc/interceptors"
)

// Service-to-Service Authentication Interceptor
authInterceptor := interceptors.NewServiceAuthInterceptor(authService)

grpcServer := grpc.NewServer(
    grpc.UnaryInterceptor(
        grpc_middleware.ChainUnaryServer(
            authInterceptor.Unary(),
            // другие interceptors
        ),
    ),
)
```

**Функционал:**
- Проверка service JWT token в metadata
- Извлечение service_name и user_token
- Логирование service-to-service вызовов
- Audit trail для compliance

---

## 7. gRPC Authorization

### 7.1 ValidateToken RPC

```protobuf
// api/proto/auth/v1/auth.proto

service AuthService {
  rpc ValidateToken(ValidateTokenRequest) returns (ValidateTokenResponse);
}

message ValidateTokenRequest {
  string token = 1;
}

message ValidateTokenResponse {
  bool valid = 1;
  int32 user_id = 2;
  string email = 3;
  repeated string roles = 4;
  map<string, string> claims = 5;
}
```

**Использование:**

```go
// Микросервис вызывает ValidateToken для проверки user token

resp, err := authClient.ValidateToken(ctx, &auth.ValidateTokenRequest{
    Token: userToken,
})

if !resp.Valid {
    return status.Error(codes.Unauthenticated, "invalid token")
}

// Проверка ролей
hasAdminRole := false
for _, role := range resp.Roles {
    if role == "admin" {
        hasAdminRole = true
    }
}
```

---

### 7.2 Service-to-Service Auth

```go
// Service Account создается заранее:

// 1. Создание service account через Admin API
POST /api/v1/admin/service-accounts
{
  "service_name": "listings-service",
  "permissions": ["listings.manage", "orders.view"]
}

// 2. Получение service JWT token (auto-refresh)
tokenManager := service.NewServiceTokenManager(&service.ServiceTokenConfig{
    AuthServiceURL: "http://auth-service:28086",
    ServiceName:    "listings-service",
    ClientID:       "service-123",
    ClientSecret:   "secret-xyz",
})

// 3. Автоматическое добавление токена в gRPC metadata
conn, err := grpc.Dial("delivery-service:50051",
    grpc.WithUnaryInterceptor(tokenManager.UnaryClientInterceptor()),
)
```

**Dual Token Pattern:**
```
gRPC Metadata:
├─ authorization: "Bearer <service_jwt>"    # Listings Service identity
└─ x-user-token: "Bearer <user_jwt>"        # Original user context
```

---

## 8. Примеры использования

### 8.1 Монолит Vondi (Fiber Framework)

```go
// backend/cmd/api/main.go

import (
    authMiddleware "github.com/vondi-global/auth/pkg/http/fiber"
    "github.com/vondi-global/auth/pkg/service"
)

// Инициализация Auth Service клиента
authService, _ := service.NewAuthService(&service.Config{
    HTTPURL: os.Getenv("AUTH_SERVICE_URL"),
    Timeout: 30 * time.Second,
})

// Публичные маршруты (опциональная аутентификация)
app.Use(authMiddleware.ExtractUser(authService))

// Защищенные маршруты
api := app.Group("/api/v1")
api.Use(authMiddleware.RequireAuth(authService))

// Админские маршруты
admin := api.Group("/admin")
admin.Use(authMiddleware.RequireRole("admin", "super_admin"))

// Ownership проверка (пользователь редактирует только свои данные)
api.Put("/listings/:id",
    authMiddleware.RequireAuth(authService),
    checkListingOwnership, // custom middleware
    updateListingHandler,
)
```

---

### 8.2 Listings Microservice (gRPC)

```go
// listings/internal/grpc/handlers/listing_handler.go

func (h *ListingHandler) UpdateListing(ctx context.Context, req *pb.UpdateListingRequest) (*pb.ListingResponse, error) {
    // 1. Извлечь user token из metadata
    md, _ := metadata.FromIncomingContext(ctx)
    userToken := md.Get("x-user-token")[0]

    // 2. Валидация через Auth Service
    validation, err := h.authClient.ValidateToken(ctx, &auth.ValidateTokenRequest{
        Token: userToken,
    })
    if err != nil || !validation.Valid {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }

    // 3. Role check
    hasVendorRole := false
    for _, role := range validation.Roles {
        if role == "vendor" || role == "admin" {
            hasVendorRole = true
        }
    }
    if !hasVendorRole {
        return nil, status.Error(codes.PermissionDenied, "insufficient permissions")
    }

    // 4. Ownership check (если не admin)
    listing := h.repo.GetListingByID(req.ListingId)
    isAdmin := contains(validation.Roles, "admin")
    if !isAdmin && listing.SellerID != validation.UserId {
        return nil, status.Error(codes.PermissionDenied, "not listing owner")
    }

    // 5. Update listing
    return h.listingService.Update(ctx, req)
}
```

---

### 8.3 Frontend (Next.js BFF Pattern)

```typescript
// frontend/src/services/api-client.ts

// Frontend НЕ знает про Auth Service!
// Вся аутентификация через BFF (Backend-for-Frontend)

// ❌ НЕПРАВИЛЬНО: прямой вызов auth-service
fetch('http://auth-service:28086/api/v1/auth/login')

// ✅ ПРАВИЛЬНО: через Next.js API route
export async function login(email: string, password: string) {
  const response = await fetch('/api/v2/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include', // httpOnly cookies
    body: JSON.stringify({ email, password }),
  });
  return response.json();
}

// Next.js API Route (BFF)
// frontend/src/app/api/v2/auth/login/route.ts

import { NextRequest, NextResponse } from 'next/server';

export async function POST(req: NextRequest) {
  const { email, password } = await req.json();

  // Проксирование в backend
  const response = await fetch('http://backend:3000/api/v1/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password }),
  });

  const data = await response.json();

  // Установка httpOnly cookie (безопасность)
  const res = NextResponse.json(data);
  res.cookies.set('access_token', data.access_token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60, // 1 hour
  });

  return res;
}
```

**Архитектура:**
```
Browser → Next.js (/api/v2/auth/*) → Backend (/api/v1/auth/*) → Auth Service (gRPC/HTTP)
         └─ httpOnly cookies       └─ Bearer JWT              └─ Internal API
```

---

### 8.4 Проверка разрешений в коде

```go
// pkg/entity/roles_example.go

import "github.com/vondi-global/auth/pkg/entity"

// 1. Проверка роли
hasRole := entity.CheckRole(userRole, entity.RoleAdmin)

// 2. Проверка разрешения
hasPermission := entity.CheckPermission(permission, entity.PermListingsEdit)

// 3. Получение приоритета роли
priority := entity.GetRolePriority(entity.RoleAdmin) // 10

// 4. Сравнение приоритетов
isHigher := entity.IsHigherPriority(entity.RoleAdmin, entity.RoleModerator) // true

// 5. Получение разрешений по роли
permissions := entity.GetPermissionsByRole(entity.RoleVendor)
// → [listings.create, listings.edit_own, listings.delete_own, orders.view_own]
```

---

## 9. Security Best Practices

### 9.1 Token Management

1. **JWT Expiration:**
   - Access Token: 1 hour
   - Refresh Token: 7 days
   - Service Token: 24 hours (auto-refresh)

2. **Token Storage:**
   - Frontend: httpOnly cookies (XSS protection)
   - Backend: Memory (не в localStorage!)
   - Service: Environment variables + auto-rotation

3. **Token Validation:**
   - ВСЕГДА валидировать через Auth Service
   - НЕ доверять client-side декодированию JWT
   - Проверять expiration и signature

---

### 9.2 RBAC Best Practices

1. **Principle of Least Privilege:**
   - Назначать минимально необходимые роли
   - Использовать *_own разрешения где возможно
   - Регулярный audit назначенных ролей

2. **Role Hierarchy:**
   - super_admin (Priority 1) → может все
   - admin (Priority 10) → административные функции
   - Остальные роли → специализированные разрешения

3. **Ownership Checks:**
   - ВСЕГДА проверять ownership для *_own операций
   - Логировать IDOR попытки
   - Использовать `RequireSelfOrAdmin()` middleware

---

### 9.3 Audit Logging

```go
// Логирование критических действий

logger.Warn().
    Int("user_id", userID).
    Str("action", "role_assignment").
    Str("target_user_id", targetUserID).
    Strs("roles_added", rolesAdded).
    Msg("Admin assigned roles to user")

logger.Error().
    Int("authenticated_user_id", authUserID).
    Int("target_user_id", targetUserID).
    Str("path", c.Path()).
    Str("ip", c.IP()).
    Msg("IDOR attempt blocked")
```

**Что логировать:**
- Все изменения ролей/разрешений
- IDOR попытки (попытки доступа к чужим ресурсам)
- Неудачные проверки авторизации
- Service-to-service вызовы (dual token pattern)

---

## 10. Migration & Testing

### 10.1 Database Schema

```sql
-- auth.users таблица
CREATE TABLE auth.users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  roles TEXT[] DEFAULT '{"user"}', -- PostgreSQL array
  created_at TIMESTAMP DEFAULT NOW()
);

-- Пример: пользователь с несколькими ролями
INSERT INTO auth.users (email, roles) VALUES
('seller@example.com', '{"vendor", "verified_buyer"}'),
('admin@vondi.rs', '{"admin", "moderator"}');

-- Индекс для быстрого поиска по ролям
CREATE INDEX idx_users_roles ON auth.users USING GIN(roles);
```

---

### 10.2 Testing Authorization

```go
// Тестирование RBAC

func TestRequireRole_Success(t *testing.T) {
    app := fiber.New()

    // Mock user с admin ролью
    app.Use(func(c *fiber.Ctx) error {
        c.Locals("user_id", 123)
        c.Locals("roles", []string{"admin"})
        return c.Next()
    })

    app.Use(RequireRole(entity.RoleAdmin))
    app.Get("/test", func(c *fiber.Ctx) error {
        return c.SendString("OK")
    })

    req := httptest.NewRequest("GET", "/test", nil)
    resp, _ := app.Test(req)

    assert.Equal(t, 200, resp.StatusCode)
}

func TestRequireRole_Forbidden(t *testing.T) {
    app := fiber.New()

    // Mock user БЕЗ admin роли
    app.Use(func(c *fiber.Ctx) error {
        c.Locals("user_id", 123)
        c.Locals("roles", []string{"user"})
        return c.Next()
    })

    app.Use(RequireRole(entity.RoleAdmin))
    app.Get("/test", func(c *fiber.Ctx) error {
        return c.SendString("OK")
    })

    req := httptest.NewRequest("GET", "/test", nil)
    resp, _ := app.Test(req)

    assert.Equal(t, 403, resp.StatusCode) // Forbidden
}
```

---

## 11. Troubleshooting

### Частые ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `401 Unauthorized` | Токен невалиден/истек | Обновить через refresh token |
| `403 Forbidden` | Нет требуемой роли/разрешения | Проверить назначенные роли пользователю |
| `IDOR attempt blocked` | Попытка доступа к чужому ресурсу | Проверить ownership logic |
| `Service token expired` | Service JWT истек | Использовать `ServiceTokenManager` для auto-refresh |

---

### Проверка ролей пользователя

```sql
-- Узнать роли пользователя
SELECT id, email, roles FROM auth.users WHERE email = 'user@example.com';

-- Добавить роль вручную
UPDATE auth.users
SET roles = array_append(roles, 'admin')
WHERE email = 'user@example.com';

-- Удалить роль
UPDATE auth.users
SET roles = array_remove(roles, 'moderator')
WHERE email = 'user@example.com';
```

---

### Debug logging

```bash
# Включить debug логи в Auth Service
export VONDIAUTH_LOG_LEVEL=debug

# Смотреть логи авторизации
docker logs -f auth-service-container | grep "authorization\|role\|permission"

# Проверить JWT токен (decode без валидации)
echo $TOKEN | jwt decode -
```

---

## 12. Roadmap

### Планируемые улучшения

1. **Dynamic Permissions (Q1 2026)**
   - Database-driven permissions (не хардкод в коде)
   - UI для управления role-permission mapping
   - Custom permissions для tenant-based multitenancy

2. **Attribute-Based Access Control (ABAC) (Q2 2026)**
   - Проверка на основе атрибутов (location, time, device)
   - Conditional permissions (например, "edit_own только в рабочее время")

3. **Audit Dashboard (Q3 2026)**
   - Web UI для просмотра audit logs
   - Alerts на подозрительную активность
   - Compliance reports (GDPR, KYC)

---

## 13. Контакты и поддержка

- **Документация:** `/p/github.com/vondi-global/.passport/security/authorization.md`
- **Auth Service Repo:** `github.com/vondi-global/auth`
- **Примеры интеграции:** `/p/github.com/vondi-global/auth/examples/`
- **Ответственный:** Malden Simić, Generalni direktor

---

**Версия документа:** 1.0
**Последнее обновление:** 2025-12-21
**Статус:** Production Ready ✅
