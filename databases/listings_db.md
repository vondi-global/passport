# Паспорт: База данных listings_dev_db

> Обновлено: 2025-12-21

## Идентификация

| Параметр | Значение |
|----------|----------|
| Тип | PostgreSQL 15 Alpine |
| Image | postgres:15-alpine |
| Хост | localhost |
| Порт | 35434 |
| Имя БД | listings_dev_db |
| Пользователь | listings_user |
| Пароль | listings_secret |
| Контейнер | listings_postgres |

## КРИТИЧНО

**Это ОТДЕЛЬНАЯ база данных от монолита!**
- НЕ используй порт 5433 (это монолит vondi_db)
- НЕ используй БД vondi_db
- Listings микросервис имеет СВОЮ независимую БД
- **Расширения:** cube, earthdistance (для геопоиска)

## Подключение

```bash
# Через localhost
psql "postgres://listings_user:listings_secret@localhost:35434/listings_dev_db?sslmode=disable"

# Через Docker
docker exec -it listings_postgres psql -U listings_user -d listings_dev_db
```

## Статистика

| Параметр | Значение |
|----------|----------|
| Всего таблиц | 37 |
| Бизнес-таблиц | 37 |
| Миграций | 89 |
| Путь к миграциям | /p/github.com/vondi-global/listings/migrations/ |

**⚠️ КРИТИЧНО:** Эта БД является **Source of Truth** для:
- Категорий (categories - 663 записи)
- Атрибутов (attributes - 30 записей)
- Связей категория-атрибут (category_attributes - 600 связей)
- Значений атрибутов (attribute_values, variant_attribute_values)

## Таблицы (полный список - 37 таблиц)

### 1. Категории (Categories V2 - JSONB многоязычность)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| categories | Категории с JSONB (sr/en/ru), иерархия 3 уровня | UUID |
| category_attributes | Связи категорий и атрибутов | SERIAL |
| category_detections | AI-детекция категорий из изображений | BIGSERIAL |

**Особенности categories:**
- UUID первичный ключ (вместо INT)
- JSONB для i18n: `name`, `description`, `meta_title`, `meta_description`
- Иерархия: `level` (1-3), `parent_id`, `path` (slug-based)
- SEO: `google_category_id`, `facebook_category_id`
- GIN индексы для полнотекстового поиска по JSONB

### 2. Атрибуты (Unified Attributes System - SOURCE OF TRUTH)

**⚠️ КРИТИЧНО:** Listings Microservice - единственный источник истины для атрибутов!

| Таблица | Назначение | Тип ID | Статус |
|---------|------------|--------|--------|
| attributes | Мета-данные атрибутов (JSONB i18n) | SERIAL | **PRODUCTION** |
| attribute_values | Значения атрибутов листингов | SERIAL | **PRODUCTION** |
| variant_attribute_values | Значения атрибутов вариантов | SERIAL | **PRODUCTION** |
| category_attributes | Связи категорий и атрибутов | SERIAL | **PRODUCTION** |

**Особенности attributes:**
- JSONB: `name`, `display_name`, `options`, `validation_rules`, `ui_settings`
- Типы: text, textarea, number, boolean, select, multiselect, date, color, size
- Purpose: regular (атрибут), variant (вариант), both
- Флаги: `is_searchable`, `is_filterable`, `is_required`, `affects_stock`, `affects_price`
- Full-text search: `search_vector` (TSVECTOR)

**Удалённые legacy таблицы из монолита (vondi_db):**
- ❌ `unified_attributes` - удалена 2025-12-21
- ❌ `unified_category_attributes` - удалена 2025-12-21
- ❌ `unified_attribute_values` - удалена 2025-12-21
- ❌ `variant_attribute_mappings` - удалена 2025-12-21

### 3. Варианты товаров (Product Variants V2)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| listing_variants | Варианты листингов (цвет, размер) | BIGSERIAL |
| product_variants | Legacy варианты (будет удалена) | SERIAL |

**Особенности listing_variants:**
- JSONB атрибуты: `{"color": "red", "size": "M"}`
- Цена вариантов: `price` (NULL = базовая цена листинга)
- Управление стоком: `stock` (INT)
- Медиа: `image_url` (специфичное изображение варианта)
- Уникальность: `listing_id + md5(attributes::text)`

### 4. Инвентарь (Inventory Management)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| inventory_reservations | Резервации товара (корзина, заказы) | BIGSERIAL |
| inventory_movements | История движения стока | BIGSERIAL |
| stock_reservations | Legacy резервации (будет удалена) | BIGSERIAL |

**Особенности inventory_reservations:**
- Типы: `reference_type` (cart, order, manual)
- TTL: `expires_at` (автоочистка старых резерваций)
- Статус: `status` (reserved, released, expired)
- Трекинг: `reserved_at`, `released_at`

### 5. Листинги (Unified Listings - C2C + B2C)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| listings | Объединенная таблица объявлений | BIGSERIAL |
| listing_images | Изображения листингов | BIGSERIAL |
| listing_locations | Геолокации листингов | BIGSERIAL |
| listing_attributes | Атрибуты листингов (key-value) | BIGSERIAL |
| listing_tags | Теги листингов | BIGSERIAL |
| listing_stats | Статистика (просмотры, избранное) | BIGSERIAL |
| listing_favorites | Избранные объявления | BIGSERIAL |

**Особенности listings:**
- Унифицированная таблица для C2C и B2C
- Связь: `user_id` (продавец), `storefront_id` (магазин)
- Статусы: draft, active, inactive, sold, archived
- Цена: `price` (DECIMAL 15,2), `currency` (VARCHAR 3)
- Стоки: `quantity` (INT), `sku` (VARCHAR 100)
- Мультиязычность: `title` (VARCHAR 255), `description` (TEXT)
- Индексы: gin для full-text search, composite для фильтров

### 6. Витрины магазинов (Storefronts - B2C)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| storefronts | Витрины B2C магазинов | SERIAL |
| storefront_staff | Сотрудники магазинов | BIGSERIAL |
| storefront_hours | Часы работы | BIGSERIAL |
| storefront_payment_methods | Методы оплаты | BIGSERIAL |
| storefront_delivery_options | Опции доставки | BIGSERIAL |
| storefront_invitations | Приглашения сотрудников | BIGSERIAL |

**Особенности storefronts:**
- Брендинг: `logo_url`, `banner_url`, `theme` (JSONB)
- Геолокация: `latitude`, `longitude`, geo indexes (cube + earthdistance)
- Статистика: `rating`, `reviews_count`, `products_count`, `sales_count`
- Подписки: `subscription_plan`, `subscription_expires_at`, `commission_rate`
- AI: `ai_agent_enabled`, `ai_agent_config` (JSONB)
- Фичи: `live_shopping_enabled`, `group_buying_enabled`

### 7. Корзина (Shopping Cart)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| shopping_carts | Корзины (auth + anonymous) | BIGSERIAL |
| cart_items | Товары в корзине | BIGSERIAL |

**Особенности shopping_carts:**
- Идентификация: `user_id` XOR `session_id` (constraint)
- Связь: `storefront_id` (одна корзина на магазин)
- Уникальность: `(user_id, storefront_id)` OR `(session_id, storefront_id)`
- Auto cleanup: индекс на `updated_at` (удаление старых >30 дней)

**Особенности cart_items:**
- Связь: `cart_id`, `listing_id`, `variant_id` (optional)
- Количество: `quantity` (INT)
- Snapshot: `price_snapshot`, `title_snapshot` (на момент добавления)
- Резервация: JOIN с `inventory_reservations`

### 8. Заказы (Orders - E-commerce)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| orders | Заказы (полный e-commerce) | BIGSERIAL |
| order_items | Товары в заказе | BIGSERIAL |

**Особенности orders:**
- Номер: `order_number` (VARCHAR 50) UNIQUE
- Пользователь: `user_id` (NULL для гостевых заказов)
- Статусы: pending → confirmed → processing → shipped → delivered
- Оплата: `payment_status`, `payment_method`, `payment_transaction_id`
- Финансы: `subtotal`, `tax`, `shipping`, `discount`, `total`
- Комиссия: `commission`, `seller_amount`
- Эскроу: `escrow_release_date`, `escrow_days` (default 3)
- Адреса: `shipping_address`, `billing_address` (JSONB)
- Доставка: `tracking_number`, `shipping_provider`, `shipment_id` (FK к Delivery MS)
- Отмена: `cancelled_at`, `cancellation_reason`

**Особенности order_items:**
- Snapshot: `price_snapshot`, `title_snapshot`, `sku_snapshot`
- Количество: `quantity`, `unit_price`
- Связи: `order_id`, `listing_id`, `variant_id` (optional)

### 9. Чаты (Messaging System)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| chats | Чаты по объявлениям | BIGSERIAL |
| messages | Сообщения в чатах | BIGSERIAL |
| chat_attachments | Вложения (фото, файлы) | BIGSERIAL |
| c2c_chats | Legacy чаты (C2C) | BIGSERIAL |
| c2c_messages | Legacy сообщения (C2C) | BIGSERIAL |

**Особенности chats:**
- Участники: `listing_id`, `buyer_id`, `seller_id`, `storefront_id`
- Статус: `status` (active, archived, blocked)
- Последнее: `last_message_at`, `last_message_content` (cache)
- Непрочитанные: `unread_count_buyer`, `unread_count_seller`
- Типирование: `typing_user_id`

**Особенности messages:**
- Отправитель: `sender_id` (user FK)
- Контент: `content` (TEXT), `message_type` (text, image, file, system)
- Статусы: `is_read`, `is_system`, `read_at`, `delivered_at`
- Вложения: JOIN с `chat_attachments`

### 10. Аналитика (Analytics & Search)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| search_queries | История поиска | BIGSERIAL |
| indexing_queue | Очередь индексации OpenSearch | BIGSERIAL |

**Особенности search_queries:**
- Запрос: `query` (TEXT), `normalized_query` (lowercase)
- Метрики: `results_count`, `clicked_listing_id`
- Пользователь: `user_id` (optional), `session_id` (anonymous)
- Фильтры: `filters` (JSONB)

### 11. Системные таблицы

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| schema_migrations | Версии миграций | - |
| users | Кеш пользователей (синхронизация из Auth MS) | BIGSERIAL |

## Схемы ключевых таблиц

### categories (Categories V2 - JSONB i18n)

```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(255) NOT NULL UNIQUE,

    -- Hierarchy
    parent_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    level INTEGER NOT NULL CHECK (level BETWEEN 1 AND 3),
    path VARCHAR(1000) NOT NULL, -- 'clothing/mens/tshirts'
    sort_order INTEGER NOT NULL DEFAULT 0,

    -- Multilingual JSONB
    name JSONB NOT NULL DEFAULT '{}', -- {"sr": "...", "en": "...", "ru": "..."}
    description JSONB DEFAULT '{}',
    meta_title JSONB DEFAULT '{}',
    meta_description JSONB DEFAULT '{}',
    meta_keywords JSONB DEFAULT '{}',

    -- Display
    icon VARCHAR(50),
    image_url VARCHAR(500),
    is_active BOOLEAN NOT NULL DEFAULT true,

    -- External mappings
    google_category_id INTEGER,
    facebook_category_id VARCHAR(100),

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_categories_slug ON categories(slug);
CREATE INDEX idx_categories_parent_id ON categories(parent_id) WHERE parent_id IS NOT NULL;
CREATE INDEX idx_categories_level ON categories(level);
CREATE INDEX idx_categories_path ON categories USING btree(path varchar_pattern_ops);
CREATE INDEX idx_categories_name_gin ON categories USING gin(name jsonb_path_ops);
CREATE INDEX idx_categories_active_l1 ON categories(sort_order)
    WHERE level = 1 AND is_active = true;
```

### attributes (Unified Attributes - JSONB metadata)

```sql
CREATE TABLE attributes (
    id SERIAL PRIMARY KEY,
    code VARCHAR(100) NOT NULL UNIQUE,

    -- i18n JSONB
    name JSONB NOT NULL DEFAULT '{}',
    display_name JSONB NOT NULL DEFAULT '{}',

    -- Configuration
    attribute_type VARCHAR(50) NOT NULL CHECK (
        attribute_type IN ('text', 'textarea', 'number', 'boolean',
                           'select', 'multiselect', 'date', 'color', 'size')
    ),
    purpose VARCHAR(20) NOT NULL DEFAULT 'regular' CHECK (
        purpose IN ('regular', 'variant', 'both')
    ),

    -- JSONB metadata
    options JSONB DEFAULT '{}',           -- Select options
    validation_rules JSONB DEFAULT '{}', -- {"min": 0, "max": 100}
    ui_settings JSONB DEFAULT '{}',      -- {"placeholder": {...}, "icon": "..."}

    -- Behavior
    is_searchable BOOLEAN NOT NULL DEFAULT false,
    is_filterable BOOLEAN NOT NULL DEFAULT false,
    is_required BOOLEAN NOT NULL DEFAULT false,
    is_variant_compatible BOOLEAN NOT NULL DEFAULT false,
    affects_stock BOOLEAN NOT NULL DEFAULT false,
    affects_price BOOLEAN NOT NULL DEFAULT false,

    -- Full-text search
    search_vector TSVECTOR,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_attributes_code ON attributes(code);
CREATE INDEX idx_attributes_purpose ON attributes(purpose);
CREATE INDEX idx_attributes_search_vector ON attributes USING GIN(search_vector);
```

### listing_variants (Product Variants V2)

```sql
CREATE TABLE listing_variants (
    id BIGSERIAL PRIMARY KEY,
    listing_id BIGINT NOT NULL,
    sku VARCHAR(100),

    -- Variant attributes JSONB
    attributes JSONB NOT NULL DEFAULT '{}', -- {"color": "red", "size": "M"}

    -- Pricing (NULL = use listing base price)
    price DECIMAL(12,2),

    -- Stock
    stock INT NOT NULL DEFAULT 0,

    -- Media
    image_url VARCHAR(500),

    -- Status
    is_active BOOLEAN NOT NULL DEFAULT true,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_listing_variants_listing FOREIGN KEY (listing_id)
        REFERENCES listings(id) ON DELETE CASCADE
);

-- Indexes
CREATE INDEX idx_listing_variants_listing ON listing_variants(listing_id);
CREATE INDEX idx_listing_variants_sku ON listing_variants(sku) WHERE sku IS NOT NULL;
CREATE INDEX idx_listing_variants_attributes ON listing_variants USING gin(attributes);
CREATE UNIQUE INDEX idx_listing_variants_unique_attrs
    ON listing_variants(listing_id, md5(attributes::text));
```

### listings (Unified Listings)

```sql
CREATE TABLE listings (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID NOT NULL DEFAULT uuid_generate_v4() UNIQUE,

    -- Ownership
    user_id BIGINT NOT NULL CHECK (user_id > 0),
    storefront_id BIGINT,

    -- Core
    title VARCHAR(255) NOT NULL CHECK (LENGTH(TRIM(title)) >= 3),
    description TEXT,
    price DECIMAL(15,2) NOT NULL CHECK (price > 0),
    currency VARCHAR(3) NOT NULL DEFAULT 'RSD',
    category_id BIGINT NOT NULL CHECK (category_id > 0),

    -- Status
    status VARCHAR(50) NOT NULL DEFAULT 'draft' CHECK (
        status IN ('draft', 'active', 'inactive', 'sold', 'archived')
    ),
    visibility VARCHAR(50) NOT NULL DEFAULT 'public' CHECK (
        visibility IN ('public', 'private', 'unlisted')
    ),

    -- Inventory
    quantity INTEGER NOT NULL DEFAULT 1 CHECK (quantity >= 0),
    sku VARCHAR(100),

    -- Metadata
    views_count INTEGER NOT NULL DEFAULT 0,
    favorites_count INTEGER NOT NULL DEFAULT 0,

    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMPTZ,
    deleted_at TIMESTAMPTZ,
    is_deleted BOOLEAN NOT NULL DEFAULT false
);

-- Indexes
CREATE INDEX idx_listings_user_id ON listings(user_id) WHERE is_deleted = false;
CREATE INDEX idx_listings_storefront_id ON listings(storefront_id) WHERE is_deleted = false;
CREATE INDEX idx_listings_category_id ON listings(category_id) WHERE is_deleted = false;
CREATE INDEX idx_listings_search ON listings USING gin(
    to_tsvector('english', title || ' ' || COALESCE(description, ''))
);
```

### storefronts (B2C Stores)

```sql
CREATE TABLE storefronts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT,

    -- Branding
    logo_url VARCHAR(500),
    banner_url VARCHAR(500),
    theme JSONB DEFAULT '{"layout": "grid", "primaryColor": "#1976d2"}',

    -- Contact
    phone VARCHAR(50),
    email VARCHAR(255),
    website VARCHAR(255),

    -- Location (cube + earthdistance)
    address TEXT,
    city VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(2) DEFAULT 'RS',
    latitude NUMERIC(10,8),
    longitude NUMERIC(11,8),

    -- Settings
    settings JSONB DEFAULT '{}',
    seo_meta JSONB DEFAULT '{}',

    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    is_verified BOOLEAN DEFAULT FALSE,

    -- Stats
    rating NUMERIC(3,2) DEFAULT 0.00,
    reviews_count INTEGER DEFAULT 0,
    products_count INTEGER DEFAULT 0,
    sales_count INTEGER DEFAULT 0,

    -- Subscription
    subscription_plan VARCHAR(50) DEFAULT 'starter',
    subscription_expires_at TIMESTAMP,
    commission_rate NUMERIC(5,2) DEFAULT 3.00,

    -- Features
    ai_agent_enabled BOOLEAN DEFAULT FALSE,
    ai_agent_config JSONB DEFAULT '{}',

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Geo index
CREATE INDEX idx_storefronts_location ON storefronts USING GIST (
    ll_to_earth(latitude::float, longitude::float)
) WHERE latitude IS NOT NULL AND longitude IS NOT NULL;
```

### orders (E-commerce Orders)

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,

    -- User (NULL for guest)
    user_id BIGINT NULL,
    storefront_id BIGINT NOT NULL,

    -- Status
    status VARCHAR(20) NOT NULL, -- pending, confirmed, shipped, delivered, cancelled
    payment_status VARCHAR(20) NOT NULL, -- pending, paid, failed, refunded

    -- Payment
    payment_method VARCHAR(50),
    payment_transaction_id VARCHAR(255),

    -- Financials
    subtotal NUMERIC(10,2) NOT NULL,
    tax NUMERIC(10,2) DEFAULT 0.00,
    shipping NUMERIC(10,2) DEFAULT 0.00,
    discount NUMERIC(10,2) DEFAULT 0.00,
    total NUMERIC(10,2) NOT NULL,
    commission NUMERIC(10,2) DEFAULT 0.00,
    seller_amount NUMERIC(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'RSD',

    -- Addresses (JSONB)
    shipping_address JSONB,
    billing_address JSONB,

    -- Shipping
    shipping_method VARCHAR(100),
    shipping_provider VARCHAR(100),
    tracking_number VARCHAR(255),

    -- Escrow
    escrow_release_date TIMESTAMPTZ,
    escrow_days INTEGER DEFAULT 3,

    -- Delivery Service integration
    shipment_id BIGINT NULL,

    -- Cancellation
    cancelled_at TIMESTAMPTZ,
    cancellation_reason TEXT,

    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_orders_storefront FOREIGN KEY (storefront_id)
        REFERENCES storefronts(id) ON DELETE RESTRICT
);
```

## Полезные запросы

```sql
-- Активные листинги по категориям
SELECT c.name->>'sr' as category_name, COUNT(l.id) as count
FROM listings l
JOIN categories c ON c.id::text = l.category_id::text
WHERE l.is_deleted = false AND l.status = 'active'
GROUP BY c.id, c.name
ORDER BY count DESC;

-- Топ продавцов по количеству объявлений
SELECT user_id, COUNT(*) as listings_count
FROM listings
WHERE is_deleted = false AND status = 'active'
GROUP BY user_id
ORDER BY listings_count DESC
LIMIT 10;

-- Листинги с вариантами
SELECT l.id, l.title, COUNT(lv.id) as variants_count
FROM listings l
LEFT JOIN listing_variants lv ON lv.listing_id = l.id
WHERE l.is_deleted = false
GROUP BY l.id, l.title
HAVING COUNT(lv.id) > 0
ORDER BY variants_count DESC;

-- Статистика заказов по статусам
SELECT status, COUNT(*) as count, SUM(total) as total_amount
FROM orders
GROUP BY status
ORDER BY count DESC;

-- Топ магазинов по продажам
SELECT s.id, s.name, s.sales_count, s.rating
FROM storefronts s
WHERE s.is_active = true
ORDER BY s.sales_count DESC
LIMIT 10;

-- Товары в корзинах (с резервациями)
SELECT ci.id, l.title, ci.quantity,
       CASE WHEN ir.id IS NOT NULL THEN 'reserved' ELSE 'available' END as reservation_status
FROM cart_items ci
JOIN listings l ON l.id = ci.listing_id
LEFT JOIN inventory_reservations ir ON ir.cart_item_id = ci.id
WHERE ci.cart_id IN (
    SELECT id FROM shopping_carts WHERE user_id = 123
);

-- Активные чаты пользователя
SELECT c.id, l.title, c.last_message_at,
       c.unread_count_buyer, c.unread_count_seller
FROM chats c
JOIN listings l ON l.id = c.listing_id
WHERE c.buyer_id = 123 OR c.seller_id = 123
ORDER BY c.last_message_at DESC;

-- Категории L1 (топ-уровень) с количеством листингов
SELECT c.name->>'sr' as name_sr,
       c.name->>'en' as name_en,
       COUNT(l.id) as listings_count
FROM categories c
LEFT JOIN listings l ON l.category_id::text = c.id::text
WHERE c.level = 1 AND c.is_active = true
GROUP BY c.id, c.name
ORDER BY c.sort_order;

-- Эскроу: заказы готовые к релизу оплаты
SELECT id, order_number, seller_amount, escrow_release_date
FROM orders
WHERE escrow_release_date <= NOW()
  AND payment_status = 'paid'
  AND status = 'delivered'
ORDER BY escrow_release_date;
```

## Связь с другими сервисами

### Интеграция с Auth Service
- `users` таблица - локальный кеш из Auth MS
- `listings.user_id` - продавец (FK к Auth)
- `orders.user_id` - покупатель (FK к Auth)

### Интеграция с Delivery Service
- `orders.shipment_id` - FK к Delivery MS
- `orders.tracking_number` - трекинг номер доставки
- `orders.shipping_provider` - провайдер (Post Express, AKS, etc)

### Интеграция с OpenSearch
- `indexing_queue` - очередь индексации
- Индексы: `marketplace_listings`, `marketplace_storefronts`
- Full-text search по title, description, attributes

### Интеграция с Redis
- Кеш категорий (TTL 1 час)
- Кеш атрибутов (TTL 1 час)
- Сессии корзины (session_id → cart_id)

### Интеграция с MinIO (s3.vondi.rs)
- `listing_images.url` - URL изображений
- `storefronts.logo_url`, `storefronts.banner_url`

## Мониторинг и диагностика

```sql
-- Размер базы данных
SELECT pg_size_pretty(pg_database_size('listings_dev_db'));

-- Размер таблиц
SELECT tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

-- Количество подключений
SELECT COUNT(*) FROM pg_stat_activity WHERE datname = 'listings_dev_db';

-- Активные запросы
SELECT pid, usename, state, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE datname = 'listings_dev_db' AND state != 'idle'
ORDER BY duration DESC;
```

## Environment Variables

```bash
# В .env файле listings микросервиса
VONDILISTINGS_DB_HOST=localhost
VONDILISTINGS_DB_PORT=35434               # НЕ 5433!
VONDILISTINGS_DB_NAME=listings_dev_db     # НЕ vondi_db!
VONDILISTINGS_DB_USER=listings_user       # НЕ postgres!
VONDILISTINGS_DB_PASSWORD=listings_secret
VONDILISTINGS_DB_SSLMODE=disable

# Redis
VONDILISTINGS_REDIS_HOST=localhost
VONDILISTINGS_REDIS_PORT=36380

# OpenSearch
VONDILISTINGS_OPENSEARCH_URL=http://localhost:9200

# gRPC
VONDILISTINGS_GRPC_PORT=50053

# HTTP
VONDILISTINGS_HTTP_PORT=8086
```

## Документация

- **CLAUDE.md:** `/p/github.com/vondi-global/listings/CLAUDE.md`
- **Миграции:** `/p/github.com/vondi-global/listings/migrations/`
- **Sync скрипты:** `/p/github.com/vondi-global/listings/scripts/README_SYNC.md`

---

**Дата создания паспорта:** 2025-12-21
**Автор:** Claude Code
**Версия:** 1.0.0
