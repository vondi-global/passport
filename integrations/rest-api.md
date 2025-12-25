# REST API Passport

> **Version:** 1.0
> **Base URL:** `/api/v1`
> **Status:** Production (монолит)
> **Architecture:** Fiber (Go HTTP framework)

---

## 1. Обзор

REST API монолита Vondi предоставляет HTTP endpoints для:
- Аутентификации и управления пользователями
- Маркетплейса (C2C listings, B2C storefronts)
- Заказов и корзины
- Доставки и логистики
- Платежей и балансов
- Административных операций

**Стратегия миграции:**
Постепенный переход на микросервисы. Монолит выступает как BFF (Backend-for-Frontend) и проксирует запросы к gRPC микросервисам (Auth, Listings, Orders, Delivery).

---

## 2. Base URL и версионирование

| Endpoint | Описание |
|----------|----------|
| `/api/v1/*` | Основная версия API |
| `/api/v2/*` | BFF proxy (Next.js → Backend) |
| `/api/public/*` | Публичные endpoints без авторизации |

**Примеры:**
```bash
# Production
https://devapi.vondi.rs/api/v1/auth/me

# Development
http://localhost:3000/api/v1/marketplace/listings
```

**Версионирование:**
- `v1` - текущая стабильная версия
- Endpoints могут иметь вложенные версии (например, `/categories/v2/*` для UUID-based API)

---

## 3. Аутентификация

### 3.1 JWT Bearer Token

```http
Authorization: Bearer <JWT_TOKEN>
```

**Получение токена:**
```bash
POST /api/v1/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGc...",
  "refresh_token": "eyJhbGc...",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "roles": ["user"]
  }
}
```

### 3.2 Middleware уровни

| Middleware | Описание | Применение |
|------------|----------|-----------|
| `JWTParser` | Парсинг JWT (опционально) | Публичные endpoints с опциональным auth |
| `RequireAuth()` | Требует авторизацию | Защищенные endpoints |
| `RequireAuth("admin")` | Требует admin роль | Административные endpoints |

### 3.3 OAuth 2.0

| Provider | Authorization URL | Callback URL |
|----------|-------------------|--------------|
| Google | `/api/v1/auth/oauth/google/url` | `/api/v1/auth/oauth/google/callback` |

**Примечание:** OAuth endpoints регистрируются через `fiberauth.Setup()` из библиотеки Auth Service.

---

## 4. Endpoints по доменам

### 4.1 Auth (`/api/v1/auth/*`)

**Регистрация через Auth Service библиотеку (`fiberauth.Setup()`):**

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/auth/register` | Регистрация пользователя | Public |
| POST | `/auth/login` | Вход в систему | Public |
| POST | `/auth/logout` | Выход из системы | Bearer |
| POST | `/auth/refresh` | Обновление токена | Refresh Token |
| POST | `/auth/validate` | Валидация токена | Public |
| GET | `/auth/me` | Получить текущего пользователя | Bearer |
| GET | `/auth/oauth/google/url` | OAuth Google URL | Public |
| GET | `/auth/oauth/google/callback` | OAuth Google callback | Public |

**Источник:** Auth Service библиотека (`github.com/vondi-global/auth/pkg/http/fiber`)

---

### 4.2 Users (`/api/v1/users/*`, `/api/v1/admin/users/*`)

#### User Profile

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/users/profile` | Получить свой профиль | Bearer |
| PUT | `/users/profile` | Обновить свой профиль | Bearer |
| GET | `/users/:id/profile` | Получить профиль по ID | Bearer |
| GET | `/users/privacy-settings` | Получить настройки приватности | Bearer |
| PUT | `/users/privacy-settings` | Обновить настройки приватности | Bearer |
| GET | `/users/chat-settings` | Получить настройки чата | Bearer |
| PUT | `/users/chat-settings` | Обновить настройки чата | Bearer |
| POST | `/users/phone/verify` | Верификация телефона (Firebase) | Bearer |

**Источник:** `backend/internal/proj/users/handler/routes.go`

#### Admin - User Management

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/admin/users/` | Получить всех пользователей | Admin |
| GET | `/admin/users/:id` | Получить пользователя по ID | Admin |
| PUT | `/admin/users/:id` | Обновить пользователя | Admin |
| PUT | `/admin/users/:id/status` | Обновить статус пользователя | Admin |
| PUT | `/admin/users/:id/role` | Обновить роль пользователя | Admin |
| DELETE | `/admin/users/:id` | Удалить пользователя | Admin |
| GET | `/admin/users/:id/balance` | Получить баланс пользователя | Admin |
| GET | `/admin/users/:id/transactions` | Получить транзакции пользователя | Admin |

#### Admin - Roles & Admins

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/admin/roles/` | Получить все роли | Admin |
| GET | `/admin/admins/` | Получить всех администраторов | Admin |
| POST | `/admin/admins/` | Добавить администратора | Admin |
| DELETE | `/admin/admins/:email` | Удалить администратора | Admin |
| GET | `/admin/admins/check/:email` | Проверить является ли админом | Admin |

---

### 4.3 Marketplace (`/api/v1/marketplace/*`)

#### Categories

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/categories` | Получить категории (V2, paginated) | Public |
| GET | `/categories/tree` | Получить дерево категорий (V2) | Public |
| GET | `/categories/:id` | Получить категорию по ID | Public |
| GET | `/categories/slug/:slug` | Получить категорию по slug | Public |
| GET | `/categories/v2/tree` | Дерево категорий V3 (UUID, i18n) | Public |
| GET | `/categories/v2/slug/:slug` | Категория по slug V3 | Public |
| GET | `/categories/v2/:id/breadcrumb` | Breadcrumb V3 | Public |
| POST | `/categories/detect` | AI определение категории | Public |
| POST | `/categories/detect/confirm` | Подтверждение выбора категории | Public |

**Legacy:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/categories` | Категории (legacy) | Public |
| GET | `/marketplace/popular-categories` | Популярные категории | Public |
| GET | `/marketplace/categories/:id/attributes` | Атрибуты категории | Public |
| GET | `/marketplace/categories/:slug/variant-attributes` | Variant атрибуты | Public |

**Admin:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/admin/categories` | Создать категорию | Admin |
| PUT | `/admin/categories/:id` | Обновить категорию | Admin |
| DELETE | `/admin/categories/:id` | Удалить категорию | Admin |

**Источник:** `backend/internal/proj/marketplace/handler/routes.go`

---

#### Search

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/search` | Поиск листингов | Public |
| GET | `/marketplace/search/facets` | Получить фасеты для фильтров | Public |
| GET | `/marketplace/search/filters` | Поиск с фильтрами | Public |
| GET | `/marketplace/search/suggestions` | Подсказки поиска | Public |
| GET | `/marketplace/search/popular` | Популярные запросы | Public |

**Источник:** OpenSearch integration via `backend/internal/proj/marketplace/handler/`

---

#### Listings (C2C)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/listings` | Получить все листинги | Admin |
| GET | `/marketplace/listings/my` | Мои листинги | Bearer |
| POST | `/marketplace/listings` | Создать листинг | Bearer |
| GET | `/marketplace/listings/:id` | Получить листинг по ID | Public |
| DELETE | `/marketplace/listings/:id` | Удалить листинг | Bearer (Owner) |
| GET | `/marketplace/listings/:id/similar` | Похожие листинги | Public |
| POST | `/marketplace/listings/:id/images` | Загрузить изображения | Bearer (Owner) |
| DELETE | `/marketplace/listings/:id/images/:imageId` | Удалить изображение | Bearer (Owner) |
| PUT | `/marketplace/listings/:id/images/reorder` | Изменить порядок изображений | Bearer (Owner) |

**Admin:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/admin/listings/statistics` | Статистика листингов | Admin |

**Источник:** `backend/internal/proj/marketplace/handler/routes.go`

---

#### Storefronts (B2C)

**Публичные:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/b2c` | Получить все витрины | Public |
| GET | `/b2c/:slug` | Получить витрину по slug | Public |
| GET | `/b2c/:id/products` | Получить товары витрины | Public |
| GET | `/marketplace/storefronts/slug/:slug` | Получить витрину по slug | Public |
| GET | `/marketplace/storefronts/slug/:slug/products` | Товары витрины по slug | Public (optional auth) |
| GET | `/marketplace/storefronts/:id` | Получить витрину по ID | Public |
| GET | `/marketplace/storefronts/:id/hours` | Часы работы | Public |
| GET | `/marketplace/storefronts/:id/is-open` | Открыто ли сейчас | Public |
| GET | `/marketplace/storefronts/:id/payment-methods` | Способы оплаты | Public |
| GET | `/marketplace/storefronts/:id/delivery-options` | Опции доставки | Public |
| GET | `/marketplace/storefronts/:id/staff` | Сотрудники витрины | Public |

**Защищенные (владелец/персонал):**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/storefronts/my` | Мои витрины | Bearer |
| POST | `/marketplace/storefronts` | Создать витрину | Bearer |
| PUT | `/marketplace/storefronts/:id` | Обновить витрину | Bearer (Owner) |
| POST | `/marketplace/storefronts/:id/logo` | Загрузить логотип | Bearer (Owner) |
| POST | `/marketplace/storefronts/:id/banner` | Загрузить баннер | Bearer (Owner) |
| POST | `/marketplace/storefronts/:id/hours` | Установить часы работы | Bearer (Owner) |
| POST | `/marketplace/storefronts/:id/payment-methods` | Установить способы оплаты | Bearer (Owner) |
| POST | `/marketplace/storefronts/:id/delivery-options` | Установить опции доставки | Bearer (Owner) |

**Admin:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/admin/b2c` | Все витрины (админ) | Admin |
| DELETE | `/marketplace/storefronts/:id` | Удалить витрину | Admin |

**Источник:** `backend/internal/proj/marketplace/handler/routes.go`

---

#### Storefront Products (B2C)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/storefronts/slug/:slug/products` | Список товаров | Public (optional auth) |
| POST | `/storefronts/slug/:slug/products` | Создать товар | Bearer (Owner/Staff) |
| GET | `/storefronts/products/:id` | Получить товар по ID | Public (optional auth) |
| GET | `/storefronts/slug/:slug/products/:id` | Товар по slug + ID | Public (optional auth) |
| POST | `/storefronts/:slug/products/:id/images` | Загрузить изображения товара | Bearer (Owner/Staff) |
| DELETE | `/storefronts/slug/:slug/products/bulk/delete` | Массовое удаление товаров | Bearer (Owner/Staff) |

**AI Endpoints (проксирование к AI module):**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/marketplace/storefronts/ai/analyze-product-image` | Анализ изображения товара | Bearer |
| POST | `/marketplace/storefronts/ai/ab-test-titles` | A/B тестирование заголовков | Bearer |
| POST | `/marketplace/storefronts/ai/translate-content` | Перевод контента | Bearer |

**Источник:** `backend/internal/proj/marketplace/handler/routes.go`

---

#### Storefront Staff Management

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/marketplace/storefronts/:id/staff` | Добавить сотрудника | Bearer (Owner/Admin) |
| PUT | `/marketplace/storefronts/:id/staff/:staff_id` | Обновить сотрудника | Bearer (Owner/Admin) |
| DELETE | `/marketplace/storefronts/:id/staff/:user_id` | Удалить сотрудника | Bearer (Owner/Admin) |

**Invitations:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/storefronts/invitations/my` | Мои приглашения | Bearer |
| POST | `/storefronts/invitations/:invitation_id/accept` | Принять приглашение | Bearer |
| POST | `/storefronts/invitations/:invitation_id/decline` | Отклонить приглашение | Bearer |
| POST | `/storefronts/:id/invitations` | Создать приглашение | Bearer (Owner/Admin) |
| GET | `/storefronts/:id/invitations` | Список приглашений | Bearer (Owner/Admin) |
| DELETE | `/storefronts/:id/invitations/:invitation_id` | Отозвать приглашение | Bearer (Owner/Admin) |

---

#### Variants API (товарные варианты)

**Public:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/listings/:id/variants` | Все варианты товара | Public |
| GET | `/marketplace/variants/:id` | Получить вариант по ID | Public |
| GET | `/variants` | Список вариантов (с фильтрами) | Public |
| GET | `/variants/:id` | Получить вариант | Public |
| GET | `/variants/sku/:sku` | Вариант по SKU | Public |
| POST | `/variants/find` | Найти вариант по атрибутам | Public |

**Protected:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/marketplace/listings/:id/variants` | Создать вариант | Bearer |
| PUT | `/marketplace/variants/:id` | Обновить вариант | Bearer |
| DELETE | `/marketplace/variants/:id` | Удалить вариант | Bearer |
| POST | `/variants/:id/reserve-stock` | Зарезервировать stock | Bearer |

**Источник:** Phase 27.4 + Phase 3 Integration
**Microservice:** Listings gRPC (проксирование через монолит)

---

#### Favorites

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/favorites` | Мои избранные | Bearer |
| POST | `/marketplace/favorites/:id` | Добавить в избранное | Bearer |
| DELETE | `/marketplace/favorites/:id` | Удалить из избранного | Bearer |

---

#### Chat (TEMPORARY - будет перенесено в микросервис)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/chat` | Получить чаты | Bearer |
| POST | `/marketplace/chat` | Создать/получить чат | Bearer |
| GET | `/marketplace/chat/:id` | Получить чат по ID | Bearer |
| POST | `/marketplace/chat/:id/messages` | Отправить сообщение | Bearer |
| GET | `/marketplace/chat/:id/messages` | Получить сообщения | Bearer |
| PUT | `/marketplace/chat/:id/messages/read` | Отметить как прочитанное | Bearer |

**WebSocket:**
```
ws://localhost:3000/ws/chat
Authorization: Bearer <token>
```

**Источник:** `backend/internal/proj/marketplace/handler/routes.go`

---

#### Storefront Dashboard (для владельцев)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/storefronts/:slug/dashboard/stats` | Статистика витрины | Bearer (Owner/Staff) |
| GET | `/storefronts/:slug/dashboard/recent-orders` | Последние заказы | Bearer (Owner/Staff) |
| GET | `/storefronts/:slug/dashboard/low-stock` | Товары с низким остатком | Bearer (Owner/Staff) |
| GET | `/storefronts/:slug/dashboard/notifications` | Уведомления | Bearer (Owner/Staff) |

---

#### Storefront Inventory (WMS интеграция)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/storefronts/:slug/inventory` | Получить инвентарь | Bearer (Owner/Staff) |
| PUT | `/storefronts/:slug/inventory` | Обновить инвентарь | Bearer (Owner/Staff) |
| GET | `/storefronts/:slug/fulfillment` | Статус фулфилмента | Bearer (Owner/Staff) |

**Microservice:** WMS gRPC (`localhost:50055`)

---

#### Transfer Requests (P9-transfers: storefront → WMS)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/storefronts/:slug/transfers/stats` | Статистика переводов | Bearer (Owner/Staff) |
| GET | `/storefronts/:slug/transfers` | Список запросов на перевод | Bearer (Owner/Staff) |
| POST | `/storefronts/:slug/transfers` | Создать запрос на перевод | Bearer (Owner/Staff) |
| GET | `/storefronts/:slug/transfers/:id` | Получить запрос на перевод | Bearer (Owner/Staff) |
| POST | `/storefronts/:slug/transfers/:id/ship` | Отправить перевод | Bearer (Owner/Staff) |
| POST | `/storefronts/:slug/transfers/:id/cancel` | Отменить перевод | Bearer (Owner/Staff) |
| GET | `/storefronts/:slug/transfers/:id/history` | История перевода | Bearer (Owner/Staff) |

**Источник:** `backend/internal/proj/marketplace/service/transfer_service.go`

---

### 4.4 Orders (`/api/v1/orders/*`, `/api/v1/marketplace/orders/*`)

#### Cart (корзина)

**Опциональная авторизация (поддержка анонимных корзин):**

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/storefronts/:storefront_id/cart` | Получить корзину | Optional |
| POST | `/marketplace/storefronts/:storefront_id/cart/items` | Добавить в корзину | Optional |
| PUT | `/marketplace/storefronts/:storefront_id/cart/items/:item_id` | Обновить товар в корзине | Optional |
| DELETE | `/marketplace/storefronts/:storefront_id/cart/items/:item_id` | Удалить из корзины | Optional |
| DELETE | `/marketplace/storefronts/:storefront_id/cart` | Очистить корзину | Optional |

**Требуют авторизацию:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/user/carts` | Все корзины пользователя | Bearer |

**Источник:** `backend/internal/proj/orders/handler/routes.go`
**Microservice:** Orders gRPC (проксирование через монолит)

---

#### Orders (заказы)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/orders` | Создать заказ | Bearer |
| GET | `/orders` | Мои заказы | Bearer |
| GET | `/orders/:id` | Получить заказ по ID | Bearer (Owner/Seller) |
| PUT | `/orders/:id/cancel` | Отменить заказ | Bearer (Owner) |

**C2C Marketplace:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/orders/my/purchases` | Мои покупки | Bearer |
| GET | `/marketplace/orders/my/sales` | Мои продажи | Bearer |
| GET | `/marketplace/orders/:id` | Получить заказ | Bearer (Buyer/Seller) |

**Storefront Orders (для продавцов B2C):**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/storefronts/:storefront_id/orders` | Заказы витрины | Bearer (Owner/Staff) |
| PUT | `/marketplace/storefronts/:storefront_id/orders/:order_id/status` | Обновить статус заказа | Bearer (Owner/Staff) |

---

#### Shipment Workflow (отправка заказов)

**B2C (через storefront_id):**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/marketplace/storefronts/:storefront_id/orders/:order_id/accept` | Подтвердить заказ (confirmed → accepted) | Bearer (Seller) |
| POST | `/marketplace/storefronts/:storefront_id/orders/:order_id/create-shipment` | Создать отправку (accepted → processing) | Bearer (Seller) |
| POST | `/marketplace/storefronts/:storefront_id/orders/:order_id/mark-shipped` | Отметить как отправлено (processing → shipped) | Bearer (Seller) |
| GET | `/marketplace/storefronts/:storefront_id/orders/:order_id/tracking` | Получить трекинг | Bearer (Seller) |

**Unified (C2C + B2C):**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/orders/:order_id/accept` | Подтвердить заказ | Bearer (Seller) |
| POST | `/orders/:order_id/shipment` | Создать отправку | Bearer (Seller) |
| POST | `/orders/:order_id/shipped` | Отметить как отправлено | Bearer (Seller) |
| GET | `/orders/:order_id/tracking` | Получить трекинг | Bearer (Buyer/Seller) |

**Источник:** `backend/internal/proj/orders/handler/routes.go`
**Workflow:** `TZ_UNIFIED_ORDER_WORKFLOW.md`, `TZ_SHIPMENT_WORKFLOW.md`

---

### 4.5 Delivery (`/api/v1/delivery/*`)

#### Публичные endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/delivery/calculate-universal` | DEPRECATED (moved to microservice) | Public |
| POST | `/delivery/calculate-cart` | DEPRECATED (moved to microservice) | Public |
| GET | `/delivery/providers` | Список провайдеров доставки | Public |

#### Checkout Flow

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/delivery/checkout/methods` | Доступные методы доставки | Public |
| GET | `/delivery/checkout/pickup-points` | Пункты самовывоза | Public |

#### Product Delivery Attributes

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/products/:id/delivery-attributes` | Атрибуты доставки товара | Public |
| PUT | `/products/:id/delivery-attributes` | Обновить атрибуты доставки | Bearer (Owner) |

#### Category Defaults

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/categories/:id/delivery-defaults` | Дефолтные атрибуты категории | Public |
| PUT | `/categories/:id/delivery-defaults` | Обновить дефолтные атрибуты | Admin |
| POST | `/categories/:id/apply-defaults` | Применить дефолты к товарам | Admin |

#### Shipments

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/shipments/` | Создать отправление | Bearer |
| GET | `/shipments/:id` | Получить отправление | Bearer |
| DELETE | `/shipments/:id` | Отменить отправление | Bearer |
| GET | `/shipments/track/:tracking` | Трекинг отправления | Public |

#### Admin

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/admin/delivery/providers` | Провайдеры (админ) | Admin |
| PUT | `/admin/delivery/providers/:id` | Обновить провайдера | Admin |
| POST | `/admin/delivery/pricing-rules` | Создать правило ценообразования | Admin |
| GET | `/admin/delivery/analytics` | Аналитика доставки | Admin |

#### Test Endpoints (публичные)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/api/public/delivery/test/shipment` | Создать тестовое отправление | Public |
| GET | `/api/public/delivery/test/tracking/:tracking_number` | Трекинг тестового отправления | Public |
| POST | `/api/public/delivery/test/cancel/:id` | Отменить тестовое отправление | Public |
| POST | `/api/public/delivery/test/calculate` | Расчет тестового тарифа | Public |
| GET | `/api/public/delivery/test/settlements` | Тестовые населенные пункты | Public |
| GET | `/api/public/delivery/test/streets` | Тестовые улицы | Public |
| GET | `/api/public/delivery/test/parcel-lockers` | Тестовые постаматы | Public |
| GET | `/api/public/delivery/test/delivery-services` | Тестовые службы доставки | Public |
| POST | `/api/public/delivery/test/validate-address` | Валидация адреса | Public |
| GET | `/api/public/delivery/test/providers` | Тестовые провайдеры | Public |
| GET | `/api/public/delivery/test/config` | Тестовая конфигурация | Public |
| GET | `/api/public/delivery/test/history` | История тестов | Public |

**Источник:** `backend/internal/proj/delivery/handler/handler.go`
**Microservice:** Delivery gRPC (`localhost:50052`)

#### Webhooks (без авторизации)

```http
POST /webhooks/delivery/:provider/tracking
Content-Type: application/json
```

**Источник:** `backend/internal/proj/delivery/handler/handler.go:82`

---

### 4.6 Payments (`/api/v1/payments/*`, `/api/v1/admin/payments/*`)

#### Payment Settings (глобальные комиссии маркетплейса)

**Admin:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/admin/payments/settings` | Все настройки платежей | Admin |
| PUT | `/admin/payments/settings` | Обновить все настройки | Admin |
| GET | `/admin/payments/settings/:method` | Настройка по методу | Admin |
| PUT | `/admin/payments/settings/:method` | Обновить настройку метода | Admin |

**Public (read-only):**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/marketplace/payment-settings` | Публичные настройки платежей | Public |

**Источник:** `backend/internal/proj/marketplace/handler/routes.go:183-189`

---

### 4.7 Balance (`/api/v1/balance/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/balance/` | Получить баланс | Bearer |
| GET | `/balance/transactions` | История транзакций | Bearer |
| POST | `/balance/withdraw` | Вывод средств | Bearer |
| POST | `/balance/transfer` | Перевод средств | Bearer |

**Источник:** `backend/internal/proj/balance/handler/routes.go`

---

### 4.8 Notifications (`/api/v1/notifications/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/notifications/` | Получить уведомления | Bearer |
| GET | `/notifications/:id` | Получить уведомление по ID | Bearer |
| PUT | `/notifications/:id/read` | Отметить как прочитанное | Bearer |
| PUT | `/notifications/read-all` | Отметить все как прочитанные | Bearer |
| DELETE | `/notifications/:id` | Удалить уведомление | Bearer |
| GET | `/notifications/settings` | Настройки уведомлений | Bearer |
| PUT | `/notifications/settings` | Обновить настройки уведомлений | Bearer |

**Telegram Integration:**
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/notifications/telegram/connect` | Подключить Telegram | Bearer |
| DELETE | `/notifications/telegram/disconnect` | Отключить Telegram | Bearer |

**Webhooks:**
```http
POST /webhooks/telegram
Content-Type: application/json
X-Telegram-Bot-Api-Secret-Token: <secret>
```

**Источник:** `backend/internal/proj/notifications/handler/routes.go`

---

### 4.9 Analytics (`/api/v1/analytics/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/analytics/storefronts/:id` | Статистика витрины | Bearer (Owner OR Admin) |
| GET | `/analytics/trending` | Трендовая статистика | Admin |
| POST | `/analytics/event` | Отправить событие аналитики | Public |

**Источник:** `backend/internal/proj/analytics/routes/routes.go`
**Microservice:** Analytics gRPC (использует тот же порт что Orders microservice)

---

### 4.10 Reviews (`/api/v1/reviews/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/reviews/listing/:id` | Отзывы на листинг | Public |
| POST | `/reviews/` | Создать отзыв | Bearer |
| PUT | `/reviews/:id` | Обновить отзыв | Bearer (Owner) |
| DELETE | `/reviews/:id` | Удалить отзыв | Bearer (Owner OR Admin) |
| POST | `/reviews/:id/helpful` | Отметить отзыв полезным | Bearer |

**Источник:** `backend/internal/proj/reviews/handler/routes.go`

---

### 4.11 Admin

#### Admin - Testing (для E2E тестов)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/admin/testing/run` | Запустить тесты | Admin |
| GET | `/admin/testing/results` | Результаты тестов | Admin |
| GET | `/admin/testing/results/:id` | Результат теста по ID | Admin |

**Примечание:** Доступно только при наличии `TEST_ADMIN_EMAIL` и `TEST_ADMIN_PASSWORD` в env.

**Источник:** `backend/internal/proj/admin/testing/handler/handler.go`

---

#### Admin - Warehouse (WMS)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/admin/warehouse/inventory` | Инвентарь склада | Admin |
| PUT | `/admin/warehouse/inventory/:id` | Обновить инвентарь | Admin |
| GET | `/admin/warehouse/transfers` | Трансферы склада | Admin |
| POST | `/admin/warehouse/transfers` | Создать трансфер | Admin |

**Источник:** `backend/internal/proj/admin/warehouse/`

---

#### Admin - Logistics

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/admin/logistics/deliveries` | Все доставки | Admin |
| GET | `/admin/logistics/deliveries/:id` | Доставка по ID | Admin |
| PUT | `/admin/logistics/deliveries/:id/status` | Обновить статус доставки | Admin |

**Источник:** `backend/internal/proj/admin/logistics/`

---

### 4.12 Config (`/api/v1/config/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/config/storage` | Конфигурация хранилища (MinIO) | Public |

**Response:**
```json
{
  "endpoint": "s3.vondi.rs",
  "bucket": "listings",
  "region": "us-east-1"
}
```

**Источник:** `backend/internal/server/server.go:774`

---

### 4.13 AI (`/api/ai/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/ai/analyze` | Анализ изображения товара | Bearer |
| POST | `/ai/ab-test` | A/B тестирование заголовков | Bearer |
| POST | `/ai/translate` | Перевод контента | Bearer |
| POST | `/ai/detect-category` | DEPRECATED (используй `/categories/detect`) | Bearer |

**Источник:** `backend/internal/proj/ai/handler/handler.go`
**Интеграция:** Replicate API (flux-redux-schnell, flux-1.1-pro)

---

### 4.14 Search Admin (`/api/v1/admin/search/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/admin/search/reindex` | Переиндексация OpenSearch | Admin |
| GET | `/admin/search/stats` | Статистика поиска | Admin |

**Источник:** `backend/internal/proj/search_admin/handler/routes.go`

---

### 4.15 Addresses (`/api/v1/addresses/*`)

**HTTP Proxy to Auth Service:**

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/addresses/` | Список адресов пользователя | Bearer |
| POST | `/addresses/` | Создать адрес | Bearer |
| PUT | `/addresses/:id` | Обновить адрес | Bearer |
| DELETE | `/addresses/:id` | Удалить адрес | Bearer |

**Источник:** `backend/internal/proj/addresses/handler/routes.go`
**Proxy to:** Auth Service HTTP API

---

### 4.16 Attributes (`/api/v1/attributes/*`)

**gRPC Proxy to Attributes Microservice:**

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/attributes/categories/:category_id` | Атрибуты категории | Public |
| POST | `/attributes/` | Создать атрибут | Admin |
| PUT | `/attributes/:id` | Обновить атрибут | Admin |
| DELETE | `/attributes/:id` | Удалить атрибут | Admin |

**Источник:** `backend/internal/proj/attributes/handler/routes.go`
**Microservice:** Attributes gRPC (тот же порт что Orders microservice)

---

### 4.17 Contacts (`/api/v1/contacts/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/contacts/send` | Отправить контактную форму | Public |

**Источник:** `backend/internal/proj/contacts/handler/routes.go`

---

### 4.18 Geocode (`/api/v1/geocode/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/geocode/address` | Геокодирование адреса | Public |
| GET | `/geocode/reverse` | Обратное геокодирование | Public |

**Источник:** `backend/internal/proj/geocode/handler/routes.go`

---

### 4.19 GIS (`/api/v1/gis/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/gis/boundaries` | Получить границы | Public |
| POST | `/gis/point-in-polygon` | Проверка точки в полигоне | Public |

**Источник:** `backend/internal/proj/gis/handler/routes.go`

---

### 4.20 Subscriptions (`/api/v1/subscriptions/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/subscriptions/` | Список подписок | Bearer |
| POST | `/subscriptions/` | Создать подписку | Bearer |
| DELETE | `/subscriptions/:id` | Удалить подписку | Bearer |

**Источник:** `backend/internal/proj/subscriptions/handler/routes.go`

---

### 4.21 Tracking (`/api/v1/tracking/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/tracking/:token` | Публичный трекинг по токену | Public |

**WebSocket:**
```
ws://localhost:3000/ws/tracking/:token
```

**Источник:** `backend/internal/server/server.go:726-771`

---

### 4.22 VIN Decoder (`/api/v1/vin/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/vin/decode/:vin` | Декодировать VIN номер | Public |

**Источник:** `backend/internal/proj/vin/`

---

### 4.23 Franchise (`/api/v1/franchise/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/franchise/register` | Регистрация франчайзи | Bearer |
| GET | `/franchise/my` | Мои франшизы | Bearer |
| PUT | `/franchise/:id` | Обновить франшизу | Bearer (Owner) |

**Источник:** `backend/internal/proj/franchise/`

---

### 4.24 Warehouse Owner Portal (`/api/v1/my-warehouse/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/my-warehouse/` | Список моих складов | Bearer (Owner) |
| GET | `/my-warehouse/:id/staff` | Сотрудники склада | Bearer (Owner) |
| POST | `/my-warehouse/:id/staff` | Добавить сотрудника | Bearer (Owner) |
| GET | `/my-warehouse/:id/transfers` | Трансферы склада | Bearer (Owner) |

**Источник:** `backend/internal/proj/warehouse_owner/`

---

### 4.25 Public (`/api/v1/join/*`)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/join/request` | Запрос на присоединение к витрине/складу | Public |

**Источник:** `backend/internal/proj/public/`

---

## 5. Pagination

### 5.1 Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | 1 | Номер страницы (начинается с 1) |
| `page_size` | integer | 20 | Количество элементов на странице |
| `limit` | integer | 20 | Альтернатива для page_size |
| `offset` | integer | 0 | Смещение (альтернатива pagination) |

### 5.2 Response Format

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### 5.3 Примеры

```bash
# Страница 2, по 50 элементов
GET /api/v1/marketplace/listings?page=2&page_size=50

# Offset-based pagination
GET /api/v1/marketplace/listings?limit=20&offset=40
```

---

## 6. Error Responses

### 6.1 Стандартный формат

```json
{
  "error": "error.placeholder",
  "status": 400,
  "details": {
    "field": "email",
    "reason": "invalid_format"
  }
}
```

### 6.2 HTTP Status Codes

| Status | Description | Use Case |
|--------|-------------|----------|
| 200 | OK | Успешный запрос |
| 201 | Created | Ресурс создан |
| 204 | No Content | Успешное удаление |
| 400 | Bad Request | Невалидные данные |
| 401 | Unauthorized | Нет авторизации |
| 403 | Forbidden | Нет прав доступа |
| 404 | Not Found | Ресурс не найден |
| 409 | Conflict | Конфликт (например, дубликат) |
| 429 | Too Many Requests | Rate limit превышен |
| 500 | Internal Server Error | Серверная ошибка |
| 501 | Not Implemented | Функционал не реализован (DEPRECATED endpoints) |
| 503 | Service Unavailable | Сервис временно недоступен |

### 6.3 Error Placeholders

Backend возвращает **placeholders** (не реальные ошибки), frontend переводит через `t()`:

```javascript
// Backend response:
{ "error": "storefronts.no_image_file" }

// Frontend translation:
t('storefronts.no_image_file') // → "Файл изображения не найден"
```

**Файлы переводов:**
- `frontend/svetu/src/messages/en/{module}.json`
- `frontend/svetu/src/messages/ru/{module}.json`
- `frontend/svetu/src/messages/sr/{module}.json`

---

## 7. Rate Limiting

### 7.1 Middleware

```go
// Пример в routes.go
app.Post("/api/v1/marketplace/listings/:id/images",
  h.jwtParserMW,
  authMiddleware.RequireAuth(),
  mw.RateLimitByUser(10, time.Minute),  // 10 запросов в минуту
  h.UploadListingImages
)
```

### 7.2 Response Headers

```http
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 7
X-RateLimit-Reset: 1635782400
```

### 7.3 Error Response (429)

```json
{
  "error": "rate_limit.exceeded",
  "status": 429,
  "retry_after": 42
}
```

---

## 8. WebSocket Endpoints

### 8.1 Chat WebSocket

```
ws://localhost:3000/ws/chat
Authorization: Bearer <JWT_TOKEN>
```

**Message Format:**
```json
{
  "type": "message",
  "chat_id": 123,
  "content": "Hello!",
  "timestamp": "2025-12-21T10:30:00Z"
}
```

**Источник:** `backend/internal/proj/chat/`
**Примечание:** TEMPORARY - будет перенесено в микросервис

---

### 8.2 Tracking WebSocket

```
ws://localhost:3000/ws/tracking/:token
```

**Public endpoint (по токену):**

```json
{
  "type": "connected",
  "delivery_id": 456
}
```

**Updates:**
```json
{
  "type": "status_update",
  "delivery_id": 456,
  "status": "in_transit",
  "location": {
    "lat": 44.8125,
    "lon": 20.4612
  },
  "timestamp": "2025-12-21T10:35:00Z"
}
```

**Источник:** `backend/internal/server/server.go:726-771`

---

## 9. Swagger Documentation

### 9.1 Доступ

| URL | Description |
|-----|-------------|
| `/swagger/doc.json` | JSON спецификация |
| `/swagger/` | Swagger UI (auto-redirect) |
| `/docs/` | Альтернативный Swagger UI |

### 9.2 Генерация

```bash
cd backend
make generate-types  # Только если изменял swagger аннотации
```

**Источник:** `backend/docs/swagger.json`
**Примечание:** Используй `pkg/utils/utils.go` для структур в swagger аннотациях

---

## 10. Health & Metrics

### 10.1 Health Check

```bash
GET /health
```

**Response:**
```json
{
  "status": "healthy",
  "version": "0.2.1",
  "services": {
    "database": "ok",
    "redis": "ok",
    "opensearch": "ok",
    "minio": "ok"
  },
  "uptime": 3600
}
```

**Источник:** `backend/internal/proj/health/handler.go`

---

### 10.2 Prometheus Metrics

```bash
GET /metrics
```

**Metrics:**
- `http_requests_total` - Всего HTTP запросов
- `http_request_duration_seconds` - Длительность запросов
- `vondi_feature_flag_enabled` - Feature flags статус
- `vondi_storefront_*` - Метрики витрин

**Источник:** `backend/internal/middleware/prometheus.go`

---

## 11. Static Files Proxy (MinIO)

| Path | Description | Bucket |
|------|-------------|--------|
| `/listings/*` | Изображения листингов | `listings` |
| `/chat-files/*` | Файлы чата | `chat-files` |
| `/storefront-products/*` | Товары витрин | `storefront-products` |

**Примеры:**
```bash
GET /listings/12345/image1.jpg
GET /storefront-products/abc123/photo.png
```

**Источник:** `backend/internal/server/server.go:786-788`
**Хранилище:** MinIO (`s3.vondi.rs`)

---

## 12. CORS & Security Headers

### 12.1 CORS

**Разрешенные origins:**
```
http://localhost:3001
https://dev.vondi.rs
https://vondi.rs
```

**Разрешенные методы:**
```
GET, POST, PUT, DELETE, PATCH, OPTIONS
```

**Разрешенные headers:**
```
Authorization, Content-Type, Accept, X-Requested-With
```

**Источник:** `backend/internal/middleware/cors.go`

---

### 12.2 Security Headers

```http
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'
```

**Источник:** `backend/internal/middleware/security_headers.go`

---

## 13. Microservices Integration

### 13.1 gRPC Clients

| Service | gRPC URL | Монолит интеграция |
|---------|----------|-------------------|
| Auth | `localhost:50051` | HTTP fallback (SetPhoneWithVerification требует HTTP) |
| Listings | `localhost:50053` | gRPC client |
| Orders | `localhost:50050` | gRPC client (ЕДИНСТВЕННЫЙ источник истины для cart/orders) |
| Delivery | `localhost:50052` | gRPC client |
| WMS | `localhost:50055` | gRPC client (опционально) |
| Attributes | Same as Orders | gRPC client |
| Categories | Same as Orders | gRPC client |
| Analytics | Same as Orders | gRPC client |
| Search | `localhost:50054` | gRPC client (опционально, fallback на OpenSearch) |

**Источник:** `backend/internal/server/server.go`

---

### 13.2 HTTP Proxying

| Route | Target Service | Description |
|-------|---------------|-------------|
| `/api/v1/addresses/*` | Auth Service HTTP | Proxy to Auth Service |
| `/api/v2/*` | Монолит | BFF proxy (Next.js → Backend) |

**Примечание:** BFF proxy использует httpOnly cookies для JWT, обеспечивая безопасность.

---

## 14. Environment Variables

### 14.1 Backend

```bash
# Server
PORT=3000
BACKEND_URL=http://localhost:3000

# Database
DATABASE_URL=postgres://postgres:password@localhost:5433/vondi_db

# Redis
REDIS_URL=localhost:6379
REDIS_PASSWORD=
REDIS_DB=0
REDIS_POOL_SIZE=10

# OpenSearch
OPENSEARCH_URL=http://localhost:9200
OPENSEARCH_USERNAME=admin
OPENSEARCH_PASSWORD=admin
OPENSEARCH_UNIFIED_INDEX=marketplace_listings

# MinIO (S3)
MINIO_ENDPOINT=s3.vondi.rs
MINIO_ACCESS_KEY_ID=...
MINIO_SECRET_ACCESS_KEY=...
MINIO_BUCKET=listings

# Microservices
AUTH_SERVICE_URL=http://localhost:28086
ORDERS_GRPC_URL=localhost:50050
LISTINGS_GRPC_URL=localhost:50053
DELIVERY_GRPC_URL=localhost:50052
SEARCH_GRPC_URL=localhost:50054

# Features
REINDEX_ON_API=  # "true" для автоматической переиндексации при старте
MIGRATIONS_ON_API=schema  # "schema", "full", or "off"
```

**Источник:** `backend/.env.example`

---

## 15. Version & Build Info

```bash
GET /
```

**Response:**
```
Vondi API 0.2.1
```

**Build информация:**
```go
// Set via ldflags
var (
  gitCommit = "abc123"
  buildTime = "2025-12-21T10:00:00Z"
)
```

**Источник:** `backend/cmd/api/main.go`

---

## 16. Deprecated Endpoints

| Endpoint | Status | Migration Path |
|----------|--------|----------------|
| `POST /api/v1/delivery/calculate-universal` | 501 Not Implemented | Delivery Microservice gRPC `CalculateRate` |
| `POST /api/v1/delivery/calculate-cart` | 501 Not Implemented | Delivery Microservice gRPC `CalculateRate` |
| `POST /api/v1/marketplace/storefronts/ai/detect-category` | Deprecated | `POST /api/v1/categories/detect` |

**Источник:** Migration to microservices in progress

---

## 17. Testing Endpoints

**Admin Testing (только при `TEST_ADMIN_EMAIL` и `TEST_ADMIN_PASSWORD`):**

```bash
POST /api/v1/admin/testing/run
GET /api/v1/admin/testing/results
GET /api/v1/admin/testing/results/:id
```

**Public Delivery Testing:**

```bash
# Полный список в разделе 4.5 Delivery
POST /api/public/delivery/test/*
```

**Test route (для проверки middleware порядка):**

```bash
POST /api/v1/auth/test
```

**Источник:** `backend/internal/server/server.go:780-782`

---

## 18. Common Request/Response Examples

### 18.1 Создание листинга

```bash
POST /api/v1/marketplace/listings
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "iPhone 15 Pro",
  "description": "Новый iPhone в отличном состоянии",
  "price": 120000,
  "category_id": "uuid-here",
  "location": {
    "lat": 44.8125,
    "lon": 20.4612
  }
}
```

**Response:**
```json
{
  "id": 12345,
  "title": "iPhone 15 Pro",
  "slug": "iphone-15-pro-12345",
  "status": "draft",
  "created_at": "2025-12-21T10:00:00Z"
}
```

---

### 18.2 Добавление в корзину

```bash
POST /api/v1/marketplace/storefronts/abc123/cart/items
Authorization: Bearer <token>  # Опционально (анонимные корзины)
Content-Type: application/json

{
  "product_id": 456,
  "variant_id": 789,
  "quantity": 2
}
```

**Response:**
```json
{
  "cart_id": "uuid",
  "item_id": 123,
  "product_id": 456,
  "variant_id": 789,
  "quantity": 2,
  "price": 5000,
  "subtotal": 10000
}
```

---

### 18.3 Создание заказа

```bash
POST /api/v1/orders
Authorization: Bearer <token>
Content-Type: application/json

{
  "cart_id": "uuid",
  "delivery_address": {
    "street": "Kneza Miloša 10",
    "city": "Beograd",
    "postal_code": "11000"
  },
  "delivery_method": "courier",
  "payment_method": "cash_on_delivery"
}
```

**Response:**
```json
{
  "order_id": "uuid",
  "status": "pending",
  "total": 15500,
  "created_at": "2025-12-21T10:00:00Z"
}
```

---

## 19. Best Practices

### 19.1 Frontend Integration

**ВСЕГДА используй BFF proxy:**
```typescript
// ✅ ПРАВИЛЬНО
import { apiClient } from '@/services/api-client';
await apiClient.get('/admin/categories');

// ❌ НЕПРАВИЛЬНО
fetch('http://localhost:3000/api/v1/...')
```

### 19.2 Error Handling

**Backend:**
```go
return utils.SendErrorResponse(c,
  fiber.StatusBadRequest,
  "listings.invalid_price",
  fiber.Map{"min": 0}
)
```

**Frontend:**
```typescript
const errorMessage = t(error.error, error.details);
```

### 19.3 Миграции вместо прямого SQL

```bash
# ✅ ПРАВИЛЬНО
cd backend && ./migrator up

# ❌ НЕПРАВИЛЬНО
psql -c "ALTER TABLE listings ADD COLUMN ..."
```

---

## 20. Related Documentation

| Document | Path | Description |
|----------|------|-------------|
| SYSTEM_PASSPORT.md | `/p/github.com/vondi-global/SYSTEM_PASSPORT.md` | Полный паспорт микросервисов |
| Auth Integration | `/p/github.com/vondi-global/auth/docs/MARKETPLACE_INTEGRATION_SPEC.md` | Auth Service спецификация |
| BFF Proxy | `/p/github.com/vondi-global/vondi/docs/BFF_PROXY_ARCHITECTURE.md` | BFF архитектура |
| Database Guidelines | `/p/github.com/vondi-global/vondi/docs/CLAUDE_DATABASE_GUIDELINES.md` | Работа с БД |
| Troubleshooting | `/p/github.com/vondi-global/vondi/docs/CLAUDE_TROUBLESHOOTING.md` | Решение проблем |

---

**Последнее обновление:** 2025-12-21
**Версия API:** v1.0
**Статус:** Production-ready (монолит), Active migration (микросервисы)
