# Domain Passport: Storefronts (B2C Marketplace)

**Domain:** Storefronts
**Type:** B2C E-commerce / Multi-vendor Marketplace
**Status:** ✅ Production (Listings Microservice)
**Last Updated:** 2025-12-21

---

## 📋 Обзор домена

**Storefronts** - это B2C (Business-to-Consumer) модуль платформы Vondi, позволяющий бизнесам создавать интернет-магазины для продажи товаров конечным покупателям.

### Основная концепция

- **Витрина (Storefront)** - это интернет-магазин продавца с собственным брендингом, каталогом товаров, настройками доставки и оплаты
- **Модель:** Multi-vendor marketplace (один маркетплейс, множество продавцов)
- **Архитектура:** gRPC микросервис `listings` + монолит backend (HTTP REST API)

### Отличие от C2C

| Критерий | B2C (Storefronts) | C2C (Listings) |
|----------|-------------------|----------------|
| Продавец | Бизнес (компания) | Физическое лицо |
| Модель | Каталог товаров + варианты | Единичные объявления |
| Инвентарь | Управление остатками, складской учет | Без управления остатками |
| Оплата | Интеграция платежных систем | P2P переводы |
| Доставка | Интеграция служб доставки | Самовывоз или договор |
| Витрина | Кастомизированный интернет-магазин | Профиль пользователя |

---

## 🏗️ Архитектура

### Микросервисная структура

```
┌─────────────────────────────────────────────────────────────┐
│                      Frontend (Next.js)                      │
│                    /[locale]/b2c/[slug]                      │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP REST
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Backend Monolith (Fiber Go)                     │
│         /api/v1/marketplace/storefronts/*                    │
│              Handler: marketplace/handler                    │
└────────┬──────────────────────────────────┬─────────────────┘
         │ gRPC (port 50053)                │ gRPC
         ▼                                   ▼
┌────────────────────────┐         ┌────────────────────────┐
│  Listings Microservice │         │  Auth Microservice     │
│  - Storefronts CRUD    │         │  - JWT validation      │
│  - Products            │         │  - User roles          │
│  - Orders              │         └────────────────────────┘
│  - Cart                │
│  - Invitations         │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│  PostgreSQL            │
│  listings_dev_db       │
│  - storefronts         │
│  - storefront_products │
│  - storefront_orders   │
└────────────────────────┘
```

### Технологический стек

**Backend Monolith:**
- Framework: Fiber (Go)
- Location: `/p/github.com/vondi-global/vondi/backend/internal/proj/marketplace/handler/`
- Endpoints: `/api/v1/marketplace/storefronts/*`, `/api/v1/b2c/*`, `/api/v1/storefronts/*`

**Listings Microservice:**
- Framework: gRPC (Go)
- Location: `/p/github.com/vondi-global/listings/`
- Port: 50053 (gRPC), 8086 (HTTP metrics)
- Database: `listings_dev_db` (PostgreSQL port 35434)
- Redis: port 36380

**Auth Integration:**
- Library: `github.com/vondi-global/auth/pkg/http/fiber/middleware`
- JWT validation + role-based access control

---

## 📊 Entities (Domain Models)

### 1. Storefront (Витрина магазина)

**Location:** `listings/internal/domain/storefront.go`

**Основные поля:**

```go
type Storefront struct {
    ID                   int64      `db:"id"`
    UserID               int64      `db:"user_id"`        // владелец
    Slug                 string     `db:"slug"`           // URL slug (уникальный)
    Name                 string     `db:"name"`
    Description          *string    `db:"description"`

    // Branding
    LogoURL              *string    `db:"logo_url"`
    BannerURL            *string    `db:"banner_url"`
    Theme                JSONB      `db:"theme"`          // цвета, шрифты

    // Contact
    Phone                *string    `db:"phone"`
    Email                *string    `db:"email"`
    Website              *string    `db:"website"`

    // Location
    Address              *string    `db:"address"`
    City                 *string    `db:"city"`
    PostalCode           *string    `db:"postal_code"`
    Country              *string    `db:"country"`
    Latitude             *float64   `db:"latitude"`
    Longitude            *float64   `db:"longitude"`
    FormattedAddress     *string    `db:"formatted_address"`

    // Settings
    Settings             JSONB      `db:"settings"`
    SeoMeta              JSONB      `db:"seo_meta"`

    // Status
    IsActive             bool       `db:"is_active"`
    IsVerified           bool       `db:"is_verified"`

    // Stats
    Rating               float64    `db:"rating"`
    ReviewsCount         int        `db:"reviews_count"`
    ProductsCount        int        `db:"products_count"`
    SalesCount           int        `db:"sales_count"`
    ViewsCount           int        `db:"views_count"`
    FollowersCount       int        `db:"followers_count"`

    // Subscription
    SubscriptionPlan     string     `db:"subscription_plan"`
    CommissionRate       float64    `db:"commission_rate"`
    IsSubscriptionActive bool       `db:"is_subscription_active"`

    // Features
    AIAgentEnabled       bool       `db:"ai_agent_enabled"`
    LiveShoppingEnabled  bool       `db:"live_shopping_enabled"`
    GroupBuyingEnabled   bool       `db:"group_buying_enabled"`

    CreatedAt            time.Time  `db:"created_at"`
    UpdatedAt            time.Time  `db:"updated_at"`
}
```

**Связи:**
- `storefront_staff` - персонал магазина (1:N)
- `storefront_hours` - график работы (1:N)
- `storefront_payment_methods` - способы оплаты (1:N)
- `storefront_delivery_options` - варианты доставки (1:N)
- `storefront_products` - товары (1:N)

### 2. StorefrontProduct (Товар в витрине)

**Location:** `vondi/backend/internal/domain/models/storefront_product.go`

```go
type StorefrontProduct struct {
    ID            int       `db:"id"`
    StorefrontID  int       `db:"storefront_id"`
    Name          string    `db:"name"`
    Description   string    `db:"description"`
    Price         float64   `db:"price"`
    Currency      string    `db:"currency"`
    CategoryID    string    `db:"category_id"`

    // Inventory
    SKU           *string   `db:"sku"`
    Barcode       *string   `db:"barcode"`
    StockQuantity int       `db:"stock_quantity"`
    StockStatus   string    `db:"stock_status"` // in_stock, low_stock, out_of_stock
    IsActive      bool      `db:"is_active"`

    // Attributes
    Attributes    JSONB     `db:"attributes"`

    // Stats
    ViewCount     int       `db:"view_count"`
    SoldCount     int       `db:"sold_count"`

    // Location (индивидуальная для товара)
    HasIndividualLocation bool     `db:"has_individual_location"`
    IndividualAddress     *string  `db:"individual_address"`
    IndividualLatitude    *float64 `db:"individual_latitude"`
    IndividualLongitude   *float64 `db:"individual_longitude"`
    LocationPrivacy       *string  `db:"location_privacy"` // exact, street, district, city
    ShowOnMap             bool     `db:"show_on_map"`

    // Variants
    HasVariants   bool     `db:"has_variants"`

    // Relations
    Images        []StorefrontProductImage   `db:"-"`
    Category      *MarketplaceCategory       `db:"-"`
    Variants      []StorefrontProductVariant `db:"-"`
    Translations  map[string]map[string]string `db:"-"` // i18n
}
```

**Связи:**
- `storefront_product_images` - изображения товара (1:N)
- `storefront_product_variants` - варианты товара (цвет, размер) (1:N)
- `marketplace_categories` - категория (N:1)

### 3. StorefrontProductVariant (Вариант товара)

**Location:** `vondi/backend/internal/domain/models/storefront_product.go`

```go
type StorefrontProductVariant struct {
    ID                int       `db:"id"`
    ProductID         int       `db:"product_id"`

    SKU               *string   `db:"sku"`
    Barcode           *string   `db:"barcode"`
    Price             *float64  `db:"price"`
    CompareAtPrice    *float64  `db:"compare_at_price"` // зачеркнутая цена
    CostPrice         *float64  `db:"cost_price"`       // себестоимость

    StockQuantity     int       `db:"stock_quantity"`
    StockStatus       string    `db:"stock_status"`
    LowStockThreshold *int      `db:"low_stock_threshold"`

    VariantAttributes JSONB     `db:"variant_attributes"` // {"color": "red", "size": "L"}

    // Logistics (для расчета доставки)
    Weight            *float64  `db:"weight"`
    Width             *float64  `db:"width"`
    Height            *float64  `db:"height"`
    Depth             *float64  `db:"depth"`
    DimensionUnit     string    `db:"dimension_unit"` // cm, m, in
    WeightUnit        string    `db:"weight_unit"`    // kg, g, lb

    IsFragile         bool      `db:"is_fragile"`
    IsHazardous       bool      `db:"is_hazardous"`
    RequiresSpecialHandling bool `db:"requires_special_handling"`

    IsActive          bool      `db:"is_active"`
    IsDefault         bool      `db:"is_default"`

    ViewCount         int       `db:"view_count"`
    SoldCount         int       `db:"sold_count"`
}
```

**Use cases:**
- Товар с разными размерами (S, M, L, XL)
- Товар с разными цветами (Red, Blue, Green)
- Товар с комбинациями (Red S, Red M, Blue S, Blue M)
- Разные цены для разных вариантов

### 4. StorefrontOrder (Заказ)

**Location:** `vondi/backend/internal/domain/models/storefront_order.go`

```go
type StorefrontOrder struct {
    ID           int64  `db:"id"`
    OrderNumber  string `db:"order_number"` // уникальный номер
    StorefrontID int    `db:"storefront_id"`
    CustomerID   int    `db:"customer_id"`  // покупатель

    // Financial
    SubtotalAmount   decimal.Decimal `db:"subtotal"`   // сумма товаров
    TaxAmount        decimal.Decimal `db:"tax"`
    ShippingAmount   decimal.Decimal `db:"shipping"`
    Discount         decimal.Decimal `db:"discount"`
    TotalAmount      decimal.Decimal `db:"total"`
    CommissionAmount decimal.Decimal `db:"commission_amount"` // комиссия платформы
    SellerAmount     decimal.Decimal `db:"seller_amount"`     // выплата продавцу
    Currency         string          `db:"currency"`

    // Payment
    PaymentMethod        string  `db:"payment_method"`
    PaymentStatus        string  `db:"payment_status"`
    PaymentTransactionID *string `db:"payment_transaction_id"`

    // Status
    Status            OrderStatus `db:"status"` // pending, confirmed, processing, shipped, delivered, canceled
    EscrowReleaseDate *time.Time  `db:"escrow_release_date"` // защита покупателя
    EscrowDays        int         `db:"escrow_days"`

    // Delivery
    ShippingAddress  JSONB   `db:"shipping_address"`
    BillingAddress   JSONB   `db:"billing_address"`
    PickupAddress    JSONB   `db:"pickup_address"`   // адрес забора у продавца
    ShippingMethod   *string `db:"shipping_method"`
    ShippingProvider *string `db:"shipping_provider"`
    TrackingNumber   *string `db:"tracking_number"`
    ShipmentID       *int64  `db:"shipment_id"`       // ID в delivery microservice
    DeliveryProvider *string `db:"delivery_provider"` // код провайдера (post_express, etc)

    // Notes
    CustomerNotes *string `db:"customer_notes"`
    SellerNotes   *string `db:"seller_notes"`

    // Relations
    Items              []StorefrontOrderItem `db:"-"`
    Storefront         *Storefront           `db:"-"`
    Customer           *User                 `db:"-"`
    PaymentTransaction *PaymentTransaction   `db:"-"`

    // Timestamps
    ConfirmedAt *time.Time `db:"confirmed_at"` // оплачен
    AcceptedAt  *time.Time `db:"accepted_at"`  // продавец подтвердил
    ShippedAt   *time.Time `db:"shipped_at"`
    DeliveredAt *time.Time `db:"delivered_at"`
    CancelledAt *time.Time `db:"canceled_at"`
}
```

**Order Statuses:**
- `pending` - ожидает оплаты
- `confirmed` - оплачен, ожидает подтверждения продавца
- `accepted` - продавец подтвердил заказ
- `processing` - создана отправка, готовится к доставке
- `shipped` - отправлен
- `delivered` - доставлен
- `canceled` - отменен
- `refunded` - возврат

### 5. StorefrontStaff (Персонал)

**Location:** `listings/internal/domain/storefront.go`

```go
type StorefrontStaff struct {
    ID           int64  `db:"id"`
    StorefrontID int64  `db:"storefront_id"`
    UserID       int64  `db:"user_id"`
    Role         string `db:"role"` // owner, manager, staff, cashier
    Permissions  JSONB  `db:"permissions"`
    IsActive     bool   `db:"is_active"`
    CreatedAt    time.Time `db:"created_at"`
}
```

**Roles:**
- `owner` - владелец (полный доступ)
- `manager` - менеджер (управление товарами, заказами)
- `staff` - персонал (просмотр заказов, управление товарами)
- `cashier` - кассир (только обработка заказов)

### 6. StorefrontInvitation (Приглашение в персонал)

**Location:** `listings/internal/domain/invitation.go`

```go
type StorefrontInvitation struct {
    ID            int64  `db:"id"`
    StorefrontID  int64  `db:"storefront_id"`
    Type          string `db:"type"` // email, link
    Role          string `db:"role"`
    Status        string `db:"status"` // pending, accepted, declined, revoked, expired

    // Email invitation
    InvitedEmail  *string `db:"invited_email"`
    InvitedUserID *int64  `db:"invited_user_id"`

    // Link invitation
    InviteCode    *string    `db:"invite_code"`
    ExpiresAt     *time.Time `db:"expires_at"`
    MaxUses       *int32     `db:"max_uses"`     // 0 = unlimited
    CurrentUses   int32      `db:"current_uses"`

    InvitedByID   int64      `db:"invited_by_id"`
    Comment       *string    `db:"comment"`
    CreatedAt     time.Time  `db:"created_at"`
}
```

**Use cases:**
- Email invitation: отправка приглашения на email
- Link invitation: генерация пригласительной ссылки (можно использовать N раз)

### 7. StorefrontHours (График работы)

```go
type StorefrontHours struct {
    ID           int64  `db:"id"`
    StorefrontID int64  `db:"storefront_id"`
    DayOfWeek    int    `db:"day_of_week"` // 0 = Sunday, 6 = Saturday
    OpenTime     string `db:"open_time"`   // "09:00"
    CloseTime    string `db:"close_time"`  // "18:00"
    IsClosed     bool   `db:"is_closed"`
}
```

### 8. StorefrontPaymentMethod (Способы оплаты)

```go
type PaymentMethod struct {
    ID               int64  `db:"id"`
    StorefrontID     int64  `db:"storefront_id"`
    Method           string `db:"method"` // cash, card, online, bank_transfer
    IsEnabled        bool   `db:"is_enabled"`
    Settings         JSONB  `db:"settings"` // настройки провайдера
    CommissionRate   float64 `db:"commission_rate"` // комиссия (%)
    ProcessingFee    float64 `db:"processing_fee"`  // фиксированная комиссия
}
```

### 9. StorefrontDeliveryOption (Варианты доставки)

```go
type StorefrontDeliveryOption struct {
    ID           int64   `db:"id"`
    StorefrontID int64   `db:"storefront_id"`
    Method       string  `db:"method"` // pickup, delivery, post_express
    IsEnabled    bool    `db:"is_enabled"`
    Settings     JSONB   `db:"settings"`
    BasePrice    float64 `db:"base_price"`
}
```

---

## 🔄 Use Cases

### UC-1: Создание витрины (Create Storefront)

**Actor:** Seller (verified user)
**Endpoint:** `POST /api/v1/marketplace/storefronts`

**Flow:**
1. Seller заполняет форму создания магазина:
   - Name, Description
   - Logo, Banner
   - Location (адрес, координаты)
   - Contact (phone, email, website)
2. Backend валидирует данные
3. Backend генерирует уникальный slug (из name)
4. Backend вызывает gRPC `listings.CreateStorefront`
5. Listings microservice:
   - Проверяет уникальность slug
   - Создает запись в `storefronts`
   - Возвращает созданную витрину
6. Backend возвращает витрину клиенту

**Validations:**
- Name: 3-100 символов
- Slug: уникальный, lowercase, alphanumeric + hyphens
- Email: валидный email
- Phone: валидный номер телефона

**Handler:** `marketplace/handler.CreateStorefront()`
**Service:** `listings/service/storefront_service.CreateStorefront()`

### UC-2: Обновление витрины (Update Storefront)

**Actor:** Storefront Owner
**Endpoint:** `PUT /api/v1/marketplace/storefronts/:id`

**Authorization:**
- JWT validation
- Check: `userID == storefront.UserID`

**Updatable fields:**
- Name, Description, IsActive
- Logo URL, Banner URL
- Theme (colors, fonts)
- Contact info (phone, email, website)
- Location
- Settings, SEO meta
- Feature flags (AI agent, Live shopping, Group buying)

### UC-3: Добавление товара в витрину (Create Product)

**Actor:** Storefront Owner/Manager
**Endpoint:** `POST /api/v1/marketplace/storefronts/slug/:slug/products`

**Flow:**
1. Owner заполняет форму товара:
   - Name, Description
   - Price, Currency
   - Category
   - Stock Quantity
   - Images
   - Variants (опционально)
2. Backend проверяет права доступа (ownership)
3. Backend вызывает gRPC `listings.CreateProduct`
4. Listings microservice:
   - Создает товар
   - Создает варианты (если есть)
   - Связывает с категорией
5. Backend возвращает созданный товар

**Product Types:**
- **Simple product:** один товар без вариантов (1 SKU)
- **Variable product:** товар с вариантами (N SKUs)

**Handler:** `marketplace/handler.CreateStorefrontProduct()`

### UC-3.1: Редактирование товара (Edit Product) - NEW!

**Actor:** Storefront Owner/Manager
**Endpoint:** `PUT /api/v1/marketplace/storefronts/slug/:slug/products/:id`

**Frontend UX Options:**

Пользователь может выбрать режим редактирования:

**1. Wizard Mode (7 шагов):**
```
EditProductWizard.tsx
├── Step 1: EditInfoStep      - Основная информация (название, описание)
├── Step 2: EditMediaStep     - Изображения и медиа
├── Step 3: EditCategoryStep  - Выбор категории
├── Step 4: EditAttributesStep - Атрибуты категории
├── Step 5: EditPricingStep   - Цены и валюта
├── Step 6: EditVariantsStep  - Варианты товара
└── Step 7: EditReviewStep    - Финальный просмотр
```

**2. Tabbed Mode (6 вкладок):**
```
EditProductTabs.tsx
├── Tab 1: EditProductInfoTab       - Основная информация
├── Tab 2: EditProductMediaTab      - Медиа
├── Tab 3: EditProductCategoryTab   - Категория
├── Tab 4: EditProductAttributesTab - Атрибуты
├── Tab 5: EditProductPricingTab    - Цены
└── Tab 6: EditProductVariantsTab   - Варианты
```

**Context Provider:** `EditProductContext.tsx`
- Хранит состояние редактируемого товара
- Загружает данные товара из API при инициализации
- Методы: `updateProduct()`, `resetProduct()`, `setCategory()`, `setAttribute()`, `addVariant()`, `removeVariant()`

**Компоненты:**
- Location: `frontend/src/components/products/`
- Steps: `frontend/src/components/products/steps/Edit*.tsx`
- Tabs: `frontend/src/components/products/tabs/Edit*Tab.tsx`
- Context: `frontend/src/contexts/EditProductContext.tsx`

**Атрибуты категории:**
- Загружаются через: `GET /api/v1/marketplace/categories/${categoryId}/attributes`
- Группируются по типу (basic, technical, condition, accessories, dimensions)
- Поддерживаемые типы: text, number, select, boolean, multiselect, range

### UC-4: Загрузка изображений товара

**Actor:** Storefront Owner/Manager
**Endpoint:** `POST /api/v1/storefronts/slug/:slug/products/:id/images`

**Flow:**
1. Owner загружает изображения через форму
2. Backend проверяет:
   - Ownership (slug принадлежит текущему пользователю)
   - Product exists
3. Backend загружает изображения в MinIO (s3.vondi.rs)
4. Backend вызывает gRPC `listings.AddProductImage` для каждого изображения
5. Listings microservice:
   - Создает записи в `storefront_product_images`
   - Первое изображение = primary
6. Backend возвращает список загруженных изображений

**Validations:**
- Max file size: 10MB
- Allowed formats: JPG, PNG, WebP
- Max images per product: 10

### UC-5: AI-анализ товара по фото

**Actor:** Storefront Owner
**Endpoint:** `POST /api/v1/marketplace/storefronts/ai/analyze-product-image`

**Flow:**
1. Owner загружает фото товара
2. Backend вызывает Claude API (Anthropic):
   - Модель: claude-sonnet-4, claude-3-5-sonnet (fallback)
   - Prompt: анализ товара (название, описание, категория, цена)
3. Claude возвращает JSON:
   ```json
   {
     "title": "Смартфон Samsung Galaxy S21",
     "titleVariants": ["Samsung S21", "Galaxy S21"],
     "description": "...",
     "category": "Electronics",
     "categoryHints": {
       "domain": "electronics",
       "productType": "smartphone",
       "keywords": ["samsung", "galaxy", "s21"]
     },
     "price": 50000,
     "priceRange": {"min": 45000, "max": 55000},
     "attributes": {"brand": "Samsung", "color": "Black"},
     "stockEstimate": 1,
     "condition": "new",
     "keywords": ["smartphone", "samsung", "galaxy"]
   }
   ```
4. Frontend pre-fills форму создания товара

**Handler:** `marketplace/handler.AnalyzeProductImageProxy()`

### UC-6: Создание заказа (Checkout)

**Actor:** Customer (Buyer)
**Endpoint:** `POST /api/v1/marketplace/orders`

**Flow:**
1. Customer заполняет корзину:
   - Добавляет товары через `POST /api/v1/cart/:storefrontId/items`
   - Указывает количество, выбирает варианты
2. Customer переходит к checkout:
   - Указывает адрес доставки
   - Выбирает способ доставки
   - Выбирает способ оплаты
3. Backend вызывает gRPC `listings.CreateOrder`:
   - Резервирует товары (inventory reservation)
   - Рассчитывает стоимость доставки (через delivery microservice)
   - Создает заказ в статусе `pending`
4. Customer оплачивает заказ:
   - Backend вызывает payment provider (Stripe, PayPal, etc)
   - При успешной оплате: статус → `confirmed`
5. Seller подтверждает заказ:
   - Статус → `accepted`
6. Seller создает отправку:
   - Backend вызывает delivery microservice
   - Получает tracking number
   - Статус → `processing`
7. Заказ отправлен:
   - Статус → `shipped`
8. Заказ доставлен:
   - Статус → `delivered`
   - После escrow периода: выплата продавцу

**Inventory Reservation:**
- Создается при создании заказа
- Экспирируется через 15 минут (если не оплачен)
- Коммитится при оплате (списание со склада)
- Релизится при отмене

### UC-7: Управление персоналом (Staff Management)

**Actor:** Storefront Owner
**Endpoints:**
- `POST /api/v1/marketplace/storefronts/:id/staff` - добавить персонал
- `PUT /api/v1/marketplace/storefronts/:id/staff/:staff_id` - обновить
- `DELETE /api/v1/marketplace/storefronts/:id/staff/:user_id` - удалить
- `GET /api/v1/marketplace/storefronts/:id/staff` - список

**Flow (Add Staff):**
1. Owner создает приглашение:
   - Email invitation: указывает email, роль
   - Link invitation: генерирует пригласительную ссылку
2. Backend вызывает gRPC `listings.CreateStorefrontInvitation`
3. Listings microservice:
   - Создает приглашение в `storefront_invitations`
   - Генерирует invite_code (для link invitation)
   - Отправляет email (для email invitation)
4. Приглашенный принимает приглашение:
   - `POST /api/v1/storefronts/invitations/:id/accept`
5. Backend вызывает gRPC `listings.AcceptStorefrontInvitation`:
   - Создает запись в `storefront_staff`
   - Обновляет статус приглашения → `accepted`

**Permissions:**
- Owner: все права
- Manager: управление товарами, заказами, персоналом (кроме owner)
- Staff: управление товарами, просмотр заказов
- Cashier: только обработка заказов

### UC-8: Настройки доставки (Delivery Options)

**Actor:** Storefront Owner
**Endpoint:** `POST /api/v1/marketplace/storefronts/:id/delivery-options`

**Flow:**
1. Owner настраивает доставку:
   - Pickup (самовывоз)
   - Delivery (доставка по городу)
   - Post Express (интеграция с почтой)
2. Backend сохраняет настройки в `storefront_delivery_options`
3. При создании заказа:
   - Frontend показывает доступные варианты
   - Backend рассчитывает стоимость доставки (delivery microservice)

**Delivery Methods:**
- `pickup` - самовывоз (бесплатно)
- `delivery` - доставка курьером (фиксированная цена или по расстоянию)
- `post_express` - почта (интеграция через delivery microservice)

### UC-9: Dashboard продавца

**Actor:** Storefront Owner
**Endpoint:** `GET /api/v1/storefronts/:slug/dashboard/stats`

**Metrics:**
- Продажи за период (день, неделя, месяц)
- Топ товары (по продажам, просмотрам)
- Заказы (новые, в обработке, доставленные)
- Низкие остатки (low stock products)
- Уведомления (новые заказы, отзывы)

**Handler:** `marketplace/handler.GetStorefrontDashboardStats()`

---

## 🔌 API Endpoints

### Public Endpoints (без авторизации)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/b2c` | Список витрин |
| GET | `/api/v1/b2c/:slug` | Витрина по slug |
| GET | `/api/v1/b2c/:id/products` | Товары витрины |
| GET | `/api/v1/marketplace/storefronts/slug/:slug/products` | Товары витрины (альтернатива) |
| GET | `/api/v1/storefronts/products/:id` | Товар по ID |
| GET | `/api/v1/storefronts/slug/:slug/products/:id` | Товар по slug и ID |
| GET | `/api/v1/marketplace/storefronts/:id/hours` | График работы |
| GET | `/api/v1/marketplace/storefronts/:id/is-open` | Открыта ли витрина сейчас |
| GET | `/api/v1/marketplace/storefronts/:id/payment-methods` | Способы оплаты |
| GET | `/api/v1/marketplace/storefronts/:id/delivery-options` | Варианты доставки |

### Protected Endpoints (JWT required)

**Storefront CRUD:**

| Method | Endpoint | Description | Role |
|--------|----------|-------------|------|
| GET | `/api/v1/marketplace/storefronts/my` | Мои витрины | User |
| POST | `/api/v1/marketplace/storefronts` | Создать витрину | User |
| GET | `/api/v1/marketplace/storefronts/:id` | Витрина по ID | User |
| PUT | `/api/v1/marketplace/storefronts/:id` | Обновить витрину | Owner |
| POST | `/api/v1/marketplace/storefronts/:id/logo` | Загрузить лого | Owner |
| POST | `/api/v1/marketplace/storefronts/:id/banner` | Загрузить баннер | Owner |

**Product Management:**

| Method | Endpoint | Description | Role |
|--------|----------|-------------|------|
| POST | `/api/v1/marketplace/storefronts/slug/:slug/products` | Создать товар | Owner/Manager |
| POST | `/api/v1/storefronts/slug/:slug/products/:id/images` | Загрузить изображения | Owner/Manager |
| DELETE | `/api/v1/storefronts/slug/:slug/products/bulk/delete` | Удалить несколько товаров | Owner/Manager |

**Staff Management:**

| Method | Endpoint | Description | Role |
|--------|----------|-------------|------|
| POST | `/api/v1/marketplace/storefronts/:id/staff` | Добавить персонал | Owner |
| PUT | `/api/v1/marketplace/storefronts/:id/staff/:staff_id` | Обновить персонал | Owner |
| DELETE | `/api/v1/marketplace/storefronts/:id/staff/:user_id` | Удалить персонал | Owner |
| GET | `/api/v1/marketplace/storefronts/:id/staff` | Список персонала | Owner/Manager |

**Invitations:**

| Method | Endpoint | Description | Role |
|--------|----------|-------------|------|
| POST | `/api/v1/storefronts/:id/invitations` | Создать приглашение | Owner |
| GET | `/api/v1/storefronts/:id/invitations` | Список приглашений | Owner |
| DELETE | `/api/v1/storefronts/:id/invitations/:invitation_id` | Отозвать приглашение | Owner |
| GET | `/api/v1/storefronts/invitations/my` | Мои приглашения | User |
| POST | `/api/v1/storefronts/invitations/:invitation_id/accept` | Принять приглашение | User |
| POST | `/api/v1/storefronts/invitations/:invitation_id/decline` | Отклонить приглашение | User |

**Settings:**

| Method | Endpoint | Description | Role |
|--------|----------|-------------|------|
| POST | `/api/v1/marketplace/storefronts/:id/hours` | Настроить график работы | Owner |
| POST | `/api/v1/marketplace/storefronts/:id/payment-methods` | Настроить оплату | Owner |
| POST | `/api/v1/marketplace/storefronts/:id/delivery-options` | Настроить доставку | Owner |

**Dashboard:**

| Method | Endpoint | Description | Role |
|--------|----------|-------------|------|
| GET | `/api/v1/storefronts/:slug/dashboard/stats` | Статистика | Owner/Manager |
| GET | `/api/v1/storefronts/:slug/dashboard/recent-orders` | Недавние заказы | Owner/Manager |
| GET | `/api/v1/storefronts/:slug/dashboard/low-stock` | Низкие остатки | Owner/Manager |
| GET | `/api/v1/storefronts/:slug/dashboard/notifications` | Уведомления | Owner/Manager |

**AI Tools:**

| Method | Endpoint | Description | Role |
|--------|----------|-------------|------|
| POST | `/api/v1/marketplace/storefronts/ai/analyze-product-image` | AI анализ фото | Owner/Manager |
| POST | `/api/v1/marketplace/storefronts/ai/ab-test-titles` | A/B тест названий | Owner/Manager |
| POST | `/api/v1/marketplace/storefronts/ai/translate-content` | Перевод контента | Owner/Manager |

### Admin Endpoints (admin role)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/admin/b2c` | Все витрины (admin) |
| DELETE | `/api/v1/marketplace/storefronts/:id` | Удалить витрину (hard delete) |

---

## 🔐 Authorization

### Middleware Stack

```go
// JWT Parser (извлекает user из токена, не требует авторизации)
jwtParserMW := authMiddleware.JWTParser(authServiceInstance)

// Require Auth (требует валидный JWT)
authMiddleware.RequireAuth()

// Require Auth with Role (требует роль)
authMiddleware.RequireAuthString("admin")
```

### Role-Based Access Control

**Ownership Check:**
```go
// В хендлере
userID, ok := authMiddleware.GetUserID(c)
if !ok {
    return utils.ErrorResponse(c, fiber.StatusUnauthorized, "auth.unauthorized")
}

// Проверка ownership
storefront, err := h.listingsClient.GetStorefrontBySlug(ctx, slug)
if storefront.UserId != int64(userID) {
    return utils.ErrorResponse(c, fiber.StatusForbidden, "storefronts.forbidden")
}
```

**Staff Permission Check:**
```go
// Проверка что пользователь - персонал магазина
staff, err := h.listingsClient.GetStaffByUserID(ctx, storefrontID, userID)
if err != nil {
    return utils.ErrorResponse(c, fiber.StatusForbidden, "storefronts.not_staff")
}

// Проверка роли
if staff.Role != "owner" && staff.Role != "manager" {
    return utils.ErrorResponse(c, fiber.StatusForbidden, "storefronts.insufficient_permissions")
}
```

---

## 📦 Database Schema

### Core Tables (в listings_dev_db)

**storefronts:**
```sql
CREATE TABLE storefronts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    logo_url TEXT,
    banner_url TEXT,
    theme JSONB,
    phone VARCHAR(50),
    email VARCHAR(255),
    website VARCHAR(255),
    address TEXT,
    city VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(100),
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    formatted_address TEXT,
    geo_strategy VARCHAR(50),
    default_privacy_level VARCHAR(20),
    address_verified BOOLEAN DEFAULT FALSE,
    settings JSONB,
    seo_meta JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    is_verified BOOLEAN DEFAULT FALSE,
    rating DECIMAL(3, 2) DEFAULT 0,
    reviews_count INTEGER DEFAULT 0,
    products_count INTEGER DEFAULT 0,
    sales_count INTEGER DEFAULT 0,
    views_count INTEGER DEFAULT 0,
    followers_count INTEGER DEFAULT 0,
    subscription_plan VARCHAR(50) DEFAULT 'starter',
    commission_rate DECIMAL(5, 2) DEFAULT 3.00,
    is_subscription_active BOOLEAN DEFAULT TRUE,
    ai_agent_enabled BOOLEAN DEFAULT FALSE,
    live_shopping_enabled BOOLEAN DEFAULT FALSE,
    group_buying_enabled BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_storefronts_user_id ON storefronts(user_id);
CREATE INDEX idx_storefronts_slug ON storefronts(slug);
CREATE INDEX idx_storefronts_is_active ON storefronts(is_active);
```

**storefront_products:**
```sql
CREATE TABLE storefront_products (
    id BIGSERIAL PRIMARY KEY,
    storefront_id BIGINT NOT NULL REFERENCES storefronts(id) ON DELETE CASCADE,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    price DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'RSD',
    category_id VARCHAR(255),
    sku VARCHAR(100),
    barcode VARCHAR(100),
    stock_quantity INTEGER DEFAULT 0,
    stock_status VARCHAR(50) DEFAULT 'in_stock',
    is_active BOOLEAN DEFAULT TRUE,
    attributes JSONB,
    view_count INTEGER DEFAULT 0,
    sold_count INTEGER DEFAULT 0,
    has_individual_location BOOLEAN DEFAULT FALSE,
    individual_address TEXT,
    individual_latitude DOUBLE PRECISION,
    individual_longitude DOUBLE PRECISION,
    location_privacy VARCHAR(20),
    show_on_map BOOLEAN DEFAULT FALSE,
    has_variants BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_storefront_products_storefront_id ON storefront_products(storefront_id);
CREATE INDEX idx_storefront_products_category_id ON storefront_products(category_id);
CREATE INDEX idx_storefront_products_is_active ON storefront_products(is_active);
CREATE INDEX idx_storefront_products_sku ON storefront_products(sku);
```

**storefront_product_variants:**
```sql
CREATE TABLE storefront_product_variants (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES storefront_products(id) ON DELETE CASCADE,
    sku VARCHAR(100),
    barcode VARCHAR(100),
    price DECIMAL(15, 2),
    compare_at_price DECIMAL(15, 2),
    cost_price DECIMAL(15, 2),
    stock_quantity INTEGER DEFAULT 0,
    stock_status VARCHAR(50) DEFAULT 'in_stock',
    low_stock_threshold INTEGER,
    variant_attributes JSONB NOT NULL, -- {"color": "red", "size": "L"}
    weight DECIMAL(10, 3),
    width DECIMAL(10, 2),
    height DECIMAL(10, 2),
    depth DECIMAL(10, 2),
    dimension_unit VARCHAR(10) DEFAULT 'cm',
    weight_unit VARCHAR(10) DEFAULT 'kg',
    is_fragile BOOLEAN DEFAULT FALSE,
    is_hazardous BOOLEAN DEFAULT FALSE,
    requires_special_handling BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    is_default BOOLEAN DEFAULT FALSE,
    view_count INTEGER DEFAULT 0,
    sold_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_product_variants_product_id ON storefront_product_variants(product_id);
CREATE INDEX idx_product_variants_sku ON storefront_product_variants(sku);
```

**storefront_staff:**
```sql
CREATE TABLE storefront_staff (
    id BIGSERIAL PRIMARY KEY,
    storefront_id BIGINT NOT NULL REFERENCES storefronts(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL,
    role VARCHAR(50) NOT NULL, -- owner, manager, staff, cashier
    permissions JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),

    UNIQUE(storefront_id, user_id)
);

CREATE INDEX idx_storefront_staff_storefront_id ON storefront_staff(storefront_id);
CREATE INDEX idx_storefront_staff_user_id ON storefront_staff(user_id);
```

**storefront_invitations:**
```sql
CREATE TABLE storefront_invitations (
    id BIGSERIAL PRIMARY KEY,
    storefront_id BIGINT NOT NULL REFERENCES storefronts(id) ON DELETE CASCADE,
    type VARCHAR(20) NOT NULL, -- email, link
    role VARCHAR(50) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending', -- pending, accepted, declined, revoked, expired
    invited_email VARCHAR(255),
    invited_user_id BIGINT,
    invite_code VARCHAR(100) UNIQUE,
    expires_at TIMESTAMP,
    max_uses INTEGER DEFAULT 0, -- 0 = unlimited
    current_uses INTEGER DEFAULT 0,
    invited_by_id BIGINT NOT NULL,
    comment TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_invitations_storefront_id ON storefront_invitations(storefront_id);
CREATE INDEX idx_invitations_invite_code ON storefront_invitations(invite_code);
CREATE INDEX idx_invitations_status ON storefront_invitations(status);
```

**storefront_orders:**
```sql
CREATE TABLE storefront_orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(100) UNIQUE NOT NULL,
    storefront_id BIGINT NOT NULL REFERENCES storefronts(id),
    customer_id BIGINT NOT NULL,
    subtotal DECIMAL(15, 2) NOT NULL,
    tax DECIMAL(15, 2) DEFAULT 0,
    shipping DECIMAL(15, 2) DEFAULT 0,
    discount DECIMAL(15, 2) DEFAULT 0,
    total DECIMAL(15, 2) NOT NULL,
    commission_amount DECIMAL(15, 2) DEFAULT 0,
    seller_amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'RSD',
    payment_method VARCHAR(50),
    payment_status VARCHAR(50),
    payment_transaction_id VARCHAR(255),
    status VARCHAR(50) DEFAULT 'pending',
    escrow_release_date TIMESTAMP,
    escrow_days INTEGER DEFAULT 7,
    shipping_address JSONB,
    billing_address JSONB,
    pickup_address JSONB,
    shipping_method VARCHAR(50),
    shipping_provider VARCHAR(50),
    tracking_number VARCHAR(255),
    shipment_id BIGINT,
    delivery_provider VARCHAR(50),
    notes TEXT,
    customer_notes TEXT,
    seller_notes TEXT,
    metadata JSONB,
    confirmed_at TIMESTAMP,
    accepted_at TIMESTAMP,
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP,
    canceled_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_orders_storefront_id ON storefront_orders(storefront_id);
CREATE INDEX idx_orders_customer_id ON storefront_orders(customer_id);
CREATE INDEX idx_orders_status ON storefront_orders(status);
CREATE INDEX idx_orders_order_number ON storefront_orders(order_number);
```

---

## 🔗 Integration Points

### 1. Listings Microservice (gRPC)

**Location:** `/p/github.com/vondi-global/listings/`
**Port:** 50053
**Protocol:** gRPC

**Main Services:**
- `CreateStorefront` - создание витрины
- `GetStorefront` - получение витрины (по ID или slug)
- `UpdateStorefront` - обновление витрины
- `DeleteStorefront` - удаление витрины
- `ListStorefronts` - список витрин с фильтрами
- `AddStaff`, `UpdateStaff`, `RemoveStaff`, `GetStaff` - управление персоналом
- `CreateProduct`, `GetProduct`, `UpdateProduct`, `DeleteProduct`, `ListProducts` - управление товарами
- `AddProductImage` - добавление изображений
- `CreateStorefrontInvitation`, `AcceptStorefrontInvitation` - приглашения

**Client Usage:**
```go
// В backend monolith
import pb "github.com/vondi-global/listings/api/proto/listings/v1"

// Create storefront
resp, err := h.listingsClient.CreateStorefront(ctx, &pb.CreateStorefrontRequest{
    UserId: userID,
    Name: "My Store",
    Slug: "my-store",
    // ...
})

// Get storefront by slug
storefront, err := h.listingsClient.GetStorefrontBySlug(ctx, slug)
```

### 2. Auth Service (HTTP + JWT)

**Location:** `/p/github.com/vondi-global/auth/`
**Port:** 28086 (HTTP), 20053 (gRPC)
**Protocol:** HTTP REST + gRPC

**Integration:**
```go
import authMiddleware "github.com/vondi-global/auth/pkg/http/fiber/middleware"

// JWT Parser middleware
jwtParserMW := authMiddleware.JWTParser(authServiceInstance)

// Require Auth middleware
protected := app.Use(authMiddleware.RequireAuth())

// In handler
userID, ok := authMiddleware.GetUserID(c)
email, ok := authMiddleware.GetEmail(c)
roles, ok := authMiddleware.GetRoles(c)
```

### 3. Delivery Service (gRPC)

**Port:** 50052
**Integration:** Расчет стоимости доставки, создание отправок, трекинг

**Usage:**
```go
// Calculate shipping cost
cost, err := h.deliveryClient.CalculateShippingCost(ctx, &delivery.CalculateRequest{
    FromAddress: storefrontAddress,
    ToAddress: customerAddress,
    Weight: totalWeight,
    // ...
})

// Create shipment
shipment, err := h.deliveryClient.CreateShipment(ctx, &delivery.CreateShipmentRequest{
    OrderId: orderID,
    Provider: "post_express",
    // ...
})
```

### 4. OpenSearch (Search Engine)

**Port:** 9200
**Index:** `marketplace_listings`

**Indexing:**
- При создании/обновлении товара → индексация в OpenSearch
- Поля: name, description, category, price, location
- Используется для поиска товаров

### 5. MinIO (Object Storage)

**URL:** s3.vondi.rs
**Bucket:** `vondi-images`

**Usage:**
- Загрузка изображений товаров
- Логотипов и баннеров витрин
- Путь: `storefronts/{storefront_id}/products/{product_id}/{timestamp}.jpg`

### 6. Claude AI (Anthropic API)

**Usage:** AI-анализ фото товаров

**Models:**
- claude-sonnet-4-20250514 (primary)
- claude-3-5-sonnet-20241022 (fallback)
- claude-3-5-haiku-20241022 (fallback)

**Features:**
- Analyze product image → title, description, category, price
- A/B test titles → best variant
- Translate content → multiple languages

---

## 🔄 Data Flow Examples

### Example 1: Create Product with AI Analysis

```
┌─────────────┐
│  Frontend   │
└──────┬──────┘
       │ 1. Upload photo
       ▼
┌──────────────────────┐
│  Backend Monolith    │
└──────┬───────────────┘
       │ 2. POST /api/v1/marketplace/storefronts/ai/analyze-product-image
       ▼
┌──────────────────────┐
│  Claude API          │
│  (Anthropic)         │
└──────┬───────────────┘
       │ 3. Return analysis JSON
       ▼
┌──────────────────────┐
│  Frontend            │
│  (Pre-fill form)     │
└──────┬───────────────┘
       │ 4. POST /api/v1/marketplace/storefronts/slug/:slug/products
       ▼
┌──────────────────────┐
│  Backend Monolith    │
└──────┬───────────────┘
       │ 5. gRPC CreateProduct
       ▼
┌──────────────────────┐
│  Listings Microservice│
└──────┬───────────────┘
       │ 6. Insert into storefront_products
       ▼
┌──────────────────────┐
│  PostgreSQL          │
│  listings_dev_db     │
└──────────────────────┘
```

### Example 2: Customer Checkout

```
┌─────────────┐
│  Customer   │
└──────┬──────┘
       │ 1. Add to cart
       ▼
┌──────────────────────┐
│  Backend Monolith    │
│  POST /api/v1/cart   │
└──────┬───────────────┘
       │ 2. gRPC AddCartItem
       ▼
┌──────────────────────┐
│  Listings Microservice│
│  cart_items table    │
└──────────────────────┘
       │ 3. Checkout
       ▼
┌──────────────────────┐
│  Backend Monolith    │
│  POST /api/v1/orders │
└──────┬───────────────┘
       │ 4. gRPC CreateOrder
       ▼
┌──────────────────────┐
│  Listings Microservice│
└──────┬───────────────┘
       │ 5. Reserve inventory
       │ 6. Calculate shipping (Delivery microservice)
       │ 7. Create order
       ▼
┌──────────────────────┐
│  PostgreSQL          │
│  - storefront_orders │
│  - inventory_reservations │
└──────────────────────┘
       │ 8. Payment
       ▼
┌──────────────────────┐
│  Payment Provider    │
│  (Stripe/PayPal)     │
└──────┬───────────────┘
       │ 9. Payment success
       ▼
┌──────────────────────┐
│  Backend Monolith    │
└──────┬───────────────┘
       │ 10. Confirm order (commit inventory)
       ▼
┌──────────────────────┐
│  Listings Microservice│
│  status: confirmed   │
└──────────────────────┘
```

---

## 🚀 Deployment

### Development Environment

**Backend Monolith:**
```bash
cd /p/github.com/vondi-global/vondi/backend
go run ./cmd/api/main.go
# Port: 3000
```

**Listings Microservice:**
```bash
/home/dim/.local/bin/start-listings-microservice.sh
# Port: 50053 (gRPC), 8086 (HTTP)
```

**Database:**
```bash
# Docker PostgreSQL
docker ps | grep listings_postgres
# Port: 35434
# DB: listings_dev_db
```

### Production Environment (dev.vondi.rs)

**Services:**
- Backend: https://devapi.vondi.rs
- Frontend: https://dev.vondi.rs
- Listings microservice: grpc://dev.vondi.rs:50053

**Deployment Script:**
```bash
cd /p/github.com/vondi-global/vondi
./deploy-to-dev.sh --all
```

---

## 📚 Related Documentation

- [Listings Microservice Architecture](/p/github.com/vondi-global/listings/docs/DATABASE_ARCHITECTURE.md)
- [Auth Service Integration](/p/github.com/vondi-global/auth/docs/MARKETPLACE_INTEGRATION_SPEC.md)
- [Migration Plan (Monolith → Microservices)](/p/github.com/vondi-global/vondi/docs/migration/MIGRATION_PLAN_TO_MICROSERVICE.md)
- [Post Express Integration](/p/github.com/vondi-global/vondi/docs/POST_EXPRESS_INTEGRATION_COMPLETE.md)
- [Image Upload Guide](/p/github.com/vondi-global/vondi/docs/IMAGE_UPLOAD_TESTING_GUIDE.md)

---

## 🔧 Development Tools

### JWT Token Generation

```bash
cd /home/dim/jwtgen && ./jwtgen > /tmp/token
TOKEN=$(cat /tmp/token)
```

### Testing API

```bash
# Get storefronts
curl -s http://localhost:3000/api/v1/b2c | jq '.'

# Get storefront by slug
curl -s http://localhost:3000/api/v1/b2c/my-store | jq '.'

# Create storefront (requires JWT)
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Store","slug":"test-store"}' \
  http://localhost:3000/api/v1/marketplace/storefronts | jq '.'

# Get my storefronts
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:3000/api/v1/marketplace/storefronts/my | jq '.'
```

### Database Queries

```bash
# Connect to listings DB
psql "postgres://listings_user:listings_secret@localhost:35434/listings_dev_db"

# Count storefronts
SELECT COUNT(*) FROM storefronts;

# List active storefronts
SELECT id, slug, name, is_active, products_count
FROM storefronts
WHERE is_active = true;

# Products by storefront
SELECT sp.id, sp.name, sp.price, sp.stock_quantity, sp.stock_status
FROM storefront_products sp
WHERE sp.storefront_id = 1
ORDER BY sp.created_at DESC;

# Staff members
SELECT ss.id, ss.role, ss.is_active
FROM storefront_staff ss
WHERE ss.storefront_id = 1;
```

---

## 📊 Metrics & Monitoring

### Health Checks

```bash
# Backend monolith
curl http://localhost:3000/

# Listings microservice
curl http://localhost:8086/health

# Metrics
curl http://localhost:8086/metrics
```

### Key Metrics

- **Storefronts:** Total count, active count, new per day
- **Products:** Total count, active count, out of stock count
- **Orders:** Total count, by status, revenue
- **Performance:** API response time, gRPC latency, database query time

---

## ⚠️ Common Issues

### Issue: "storefront not found"

**Причина:** Микросервис подключен к монолитной БД вместо `listings_dev_db`

**Решение:**
```bash
# Проверить .env
cat /p/github.com/vondi-global/listings/.env | grep DB_PORT
# Должно быть: VONDILISTINGS_DB_PORT=35434

# Перезапустить микросервис
/home/dim/.local/bin/stop-listings-microservice.sh
/home/dim/.local/bin/start-listings-microservice.sh
```

### Issue: "forbidden" при создании товара

**Причина:** Проверка ownership не прошла

**Решение:**
- Убедиться что JWT токен валидный
- Проверить что `storefront.user_id == JWT.user_id`
- Проверить что пользователь - владелец или менеджер

### Issue: AI анализ возвращает fallback

**Причина:** Claude API недоступен или превышен лимит

**Решение:**
- Проверить CLAUDE_API_KEY в .env
- Проверить лимиты Anthropic API
- Fallback response используется автоматически

---

## 🎯 Future Enhancements

### Planned Features

- [ ] **Reviews & Ratings:** Отзывы покупателей о товарах и магазинах
- [ ] **Wishlist:** Список желаний покупателей
- [ ] **Promotions:** Скидки, купоны, распродажи
- [ ] **Bundles:** Комплекты товаров (bundle offers)
- [ ] **Pre-orders:** Предзаказы товаров
- [ ] **Digital Products:** Продажа цифровых товаров (ebooks, software)
- [ ] **Subscriptions:** Подписки на товары (recurring payments)
- [ ] **Multi-location:** Несколько точек продаж (филиалы)
- [ ] **Analytics Dashboard:** Расширенная аналитика продаж
- [ ] **Email Marketing:** Email рассылки покупателям
- [ ] **SMS Notifications:** SMS уведомления о заказах

### Tech Improvements

- [ ] **GraphQL API:** Добавить GraphQL для фронтенда
- [ ] **Webhooks:** Уведомления о событиях (новый заказ, изменение статуса)
- [ ] **Event Sourcing:** Переход на event-driven архитектуру
- [ ] **CQRS:** Разделение команд и запросов
- [ ] **CDC (Change Data Capture):** Синхронизация данных через Debezium

---

**Last Updated:** 2025-12-21
**Version:** 1.0.0
**Status:** ✅ Production Ready
