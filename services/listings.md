# Listings Service Passport

**Версия:** 1.0.0
**Статус:** Production Ready
**Последнее обновление:** 2025-12-21

---

## 1. Обзор

**Vondi Listings Service** — микросервис управления товарами, объявлениями, категориями, атрибутами и вариантами товаров для маркетплейса.

### Ключевые возможности

- Унифицированное управление C2C и B2C объявлениями
- UUID-based категории с JSONB локализацией (sr, en, ru)
- **Унифицированная система атрибутов (Source of Truth для всей платформы)**
- Управление вариантами товаров (ProductVariantV2)
- FIFO-стратегия управления инвентарем
- Полнотекстовый поиск (OpenSearch)
- Redis кеширование
- Аналитика поиска

**⚠️ КРИТИЧНО: SOURCE OF TRUTH**

Этот микросервис является единственным источником истины для:
- **Категорий** (categories: 663 записи)
- **Атрибутов** (attributes: 30 записей)
- **Связей** (category_attributes: 600 связей)
- **Значений атрибутов** (attribute_values, variant_attribute_values)

**Удалённые legacy таблицы из монолита (2025-12-21):**
- ❌ `unified_attributes` (vondi_db:5433)
- ❌ `unified_category_attributes` (vondi_db:5433)
- ❌ `unified_attribute_values` (vondi_db:5433)
- ❌ `variant_attribute_mappings` (vondi_db:5433)

### Технический стек

- **Язык:** Go 1.23
- **Архитектура:** DDD (Domain-Driven Design)
- **База данных:** PostgreSQL 15 (120+ миграций, 37 таблиц)
- **Кеш:** Redis 7
- **Поиск:** OpenSearch 2.11
- **Хранилище:** MinIO (S3-compatible)
- **API:** gRPC + HTTP REST

---

## 2. Конфигурация

### Идентификация

| Параметр | Значение |
|----------|----------|
| Репозиторий | `/p/github.com/vondi-global/listings` |
| gRPC порт | 50053 |
| HTTP порт | 8086 |
| Metrics порт | 9093 |
| PostgreSQL | listings_dev_db:35434 |
| Redis | localhost:36380 |

### Переменные окружения

**Формат:** `VONDILISTINGS_*`

#### Основные

```bash
# Application
VONDILISTINGS_ENV=development|production
VONDILISTINGS_LOG_LEVEL=debug|info|warn|error
VONDILISTINGS_LOG_FORMAT=json|console

# Server
VONDILISTINGS_GRPC_PORT=50053
VONDILISTINGS_GRPC_HOST=0.0.0.0
VONDILISTINGS_HTTP_PORT=8086
VONDILISTINGS_HTTP_HOST=0.0.0.0
VONDILISTINGS_METRICS_PORT=9093

# Database (КРИТИЧНО: НЕ монолитная БД!)
VONDILISTINGS_DB_HOST=localhost
VONDILISTINGS_DB_PORT=35434              # НЕ 5433!
VONDILISTINGS_DB_USER=listings_user      # НЕ postgres!
VONDILISTINGS_DB_NAME=listings_dev_db    # НЕ vondi_db!
VONDILISTINGS_DB_PASSWORD=listings_secret
VONDILISTINGS_DB_SSLMODE=disable

# Connection Pool
VONDILISTINGS_DB_MAX_OPEN_CONNS=25
VONDILISTINGS_DB_MAX_IDLE_CONNS=10
VONDILISTINGS_DB_CONN_MAX_LIFETIME=5m
VONDILISTINGS_DB_CONN_MAX_IDLE_TIME=10m

# Redis
VONDILISTINGS_REDIS_HOST=localhost
VONDILISTINGS_REDIS_PORT=36380
VONDILISTINGS_REDIS_PASSWORD=redis_password
VONDILISTINGS_REDIS_DB=0
VONDILISTINGS_REDIS_POOL_SIZE=10
VONDILISTINGS_REDIS_MIN_IDLE_CONNS=5

# Cache TTL
VONDILISTINGS_CACHE_LISTING_TTL=5m
VONDILISTINGS_CACHE_SEARCH_TTL=2m

# OpenSearch
VONDILISTINGS_OPENSEARCH_ADDRESSES=http://localhost:9200
VONDILISTINGS_OPENSEARCH_USERNAME=admin
VONDILISTINGS_OPENSEARCH_PASSWORD=admin
VONDILISTINGS_OPENSEARCH_INDEX=marketplace_listings

# MinIO (S3)
VONDILISTINGS_MINIO_ENDPOINT=localhost:9000
VONDILISTINGS_MINIO_ACCESS_KEY=minioadmin
VONDILISTINGS_MINIO_SECRET_KEY=minioadmin
VONDILISTINGS_MINIO_USE_SSL=false
VONDILISTINGS_MINIO_BUCKET=listings-images
VONDILISTINGS_MINIO_PUBLIC_BASE_URL=https://s3.vondi.rs

# Auth Service Integration
VONDILISTINGS_AUTH_SERVICE_URL=http://localhost:28086
VONDILISTINGS_AUTH_GRPC_URL=localhost:20053
VONDILISTINGS_AUTH_PUBLIC_KEY_PATH=./keys/public.pem
VONDILISTINGS_AUTH_ENABLED=true

# Delivery Service Integration
VONDILISTINGS_DELIVERY_ENABLED=false
VONDILISTINGS_DELIVERY_GRPC_ADDRESS=localhost:50052
VONDILISTINGS_DELIVERY_TIMEOUT=10s
VONDILISTINGS_DELIVERY_MAX_RETRIES=3
VONDILISTINGS_DELIVERY_RETRY_DELAY=100ms

# Rate Limiting
VONDILISTINGS_RATE_LIMIT_ENABLED=true
VONDILISTINGS_RATE_LIMIT_RPS=100
VONDILISTINGS_RATE_LIMIT_BURST=200

# Feature Flags
VONDILISTINGS_FEATURE_ASYNC_INDEXING=true
VONDILISTINGS_FEATURE_IMAGE_OPTIMIZATION=true
VONDILISTINGS_FEATURE_CACHE_ENABLED=true

# Health Checks
VONDILISTINGS_HEALTH_CHECK_TIMEOUT=5s
VONDILISTINGS_HEALTH_CHECK_INTERVAL=30s
VONDILISTINGS_HEALTH_STARTUP_TIMEOUT=60s
VONDILISTINGS_HEALTH_CACHE_DURATION=10s
VONDILISTINGS_HEALTH_ENABLE_DEEP_CHECKS=true

# CORS
VONDILISTINGS_CORS_ALLOWED_ORIGINS=http://localhost:3001,http://localhost:3000
VONDILISTINGS_CORS_ALLOWED_METHODS=GET,POST,PUT,DELETE,OPTIONS
VONDILISTINGS_CORS_ALLOWED_HEADERS=Content-Type,Authorization

# Tracing
VONDILISTINGS_TRACING_ENABLED=false
VONDILISTINGS_JAEGER_ENDPOINT=http://localhost:14268/api/traces
```

### Подключение к БД

```bash
# Docker контейнер
docker ps | grep listings_postgres

# psql
psql "postgres://listings_user:listings_secret@localhost:35434/listings_dev_db"

# Проверка таблиц
psql "postgres://listings_user:listings_secret@localhost:35434/listings_dev_db" -c "\dt"
```

---

## 3. Доменная модель

### 3.1 Listing (Объявление)

**Таблица:** `listings`

Унифицированная таблица для C2C и B2C товаров.

```sql
CREATE TABLE listings (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID NOT NULL DEFAULT gen_random_uuid(),
    user_id BIGINT NOT NULL,
    storefront_id BIGINT,
    category_id UUID NOT NULL,  -- UUID reference

    title VARCHAR(255) NOT NULL,
    description TEXT,
    price NUMERIC(12,2) NOT NULL,
    compare_at_price NUMERIC(12,2),

    source_type VARCHAR(20) DEFAULT 'c2c',  -- 'c2c' | 'b2c'
    status VARCHAR(20) DEFAULT 'draft',

    stock_quantity INTEGER DEFAULT 0,
    reserved_quantity INTEGER DEFAULT 0,

    views_count INTEGER DEFAULT 0,
    favorites_count INTEGER DEFAULT 0,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);
```

**Статусы:**
- `draft` — черновик
- `active` — активно
- `sold` — продано (C2C)
- `inactive` — неактивно
- `deleted` — удалено

### 3.2 Category (Категория V2)

**Таблица:** `categories`

UUID-based категории с JSONB локализацией.

```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(255) UNIQUE NOT NULL,
    parent_id UUID REFERENCES categories(id),
    level INTEGER DEFAULT 0,
    path VARCHAR(500),  -- materialized path: "001.002.003"
    sort_order INTEGER DEFAULT 0,

    -- JSONB локализация
    name JSONB NOT NULL DEFAULT '{"sr": "", "en": "", "ru": ""}',
    description JSONB DEFAULT '{"sr": "", "en": "", "ru": ""}',
    meta_title JSONB DEFAULT '{"sr": "", "en": "", "ru": ""}',
    meta_description JSONB DEFAULT '{"sr": "", "en": "", "ru": ""}',
    meta_keywords JSONB DEFAULT '{"sr": "", "en": "", "ru": ""}',

    icon VARCHAR(255),
    image_url VARCHAR(500),
    is_active BOOLEAN DEFAULT true,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Особенности:**
- UUID вместо INTEGER ID
- JSONB для многоязычности (sr, en, ru)
- Materialized path для иерархии
- Slug для SEO-friendly URLs

### 3.3 Attribute (Унифицированный атрибут)

**Таблица:** `attributes`

```sql
CREATE TABLE attributes (
    id SERIAL PRIMARY KEY,
    code VARCHAR(100) UNIQUE NOT NULL,

    -- JSONB локализация
    name JSONB NOT NULL DEFAULT '{"sr": "", "en": "", "ru": ""}',
    display_name JSONB NOT NULL DEFAULT '{"sr": "", "en": "", "ru": ""}',

    attribute_type VARCHAR(50) NOT NULL,  -- text, number, select, multi_select, boolean, date
    purpose VARCHAR(20) DEFAULT 'regular', -- 'regular' | 'variant' | 'both'

    -- JSONB конфигурация
    options JSONB DEFAULT '{}',  -- для select/multi_select
    validation_rules JSONB DEFAULT '{}',
    ui_settings JSONB DEFAULT '{}',

    is_searchable BOOLEAN DEFAULT false,
    is_filterable BOOLEAN DEFAULT false,
    is_required BOOLEAN DEFAULT false,
    is_variant_compatible BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Purpose (назначение атрибута):**
- `regular` — только для описания товара
- `variant` — только для вариантов
- `both` — универсальный (можно использовать везде)

### 3.4 ProductVariant (Вариант товара)

**Таблица:** `product_variants`

```sql
CREATE TABLE product_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id BIGINT NOT NULL REFERENCES listings(id),
    sku VARCHAR(100) UNIQUE NOT NULL,

    price NUMERIC(12,2),
    compare_at_price NUMERIC(12,2),

    stock_quantity INTEGER DEFAULT 0,
    reserved_quantity INTEGER DEFAULT 0,
    available_quantity INTEGER GENERATED ALWAYS AS (stock_quantity - reserved_quantity) STORED,

    low_stock_alert INTEGER DEFAULT 5,
    weight_grams NUMERIC(10,2),
    barcode VARCHAR(100),

    is_default BOOLEAN DEFAULT false,
    position INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'active',  -- active, out_of_stock, discontinued

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 3.5 Inventory (Инвентарь FIFO)

**Таблица:** `inventory_movements`

FIFO-стратегия управления остатками.

```sql
CREATE TABLE inventory_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL REFERENCES product_variants(id),

    movement_type VARCHAR(20) NOT NULL,  -- 'receive', 'reserve', 'deduct', 'release', 'adjust'
    quantity INTEGER NOT NULL,

    reference_type VARCHAR(50),  -- 'purchase_order', 'sale_order', 'adjustment'
    reference_id VARCHAR(100),

    notes TEXT,
    performed_by BIGINT,

    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Типы движений:**
- `receive` — поступление товара
- `reserve` — резервирование под заказ
- `deduct` — списание (продажа)
- `release` — освобождение резерва
- `adjust` — корректировка остатков

### 3.6 Основные таблицы БД

| Таблица | Назначение |
|---------|-----------|
| listings | Товары/объявления (C2C + B2C) |
| categories | Категории (UUID + JSONB) |
| attributes | Атрибуты (JSONB i18n) |
| category_attributes | Связь категория-атрибут |
| product_variants | Варианты товаров |
| variant_attribute_values | Значения атрибутов вариантов |
| listing_images | Изображения |
| listing_favorites | Избранное |
| listing_locations | Геолокация |
| listing_attributes | Значения атрибутов |
| inventory_movements | Движения товара (FIFO) |
| storefronts | Витрины магазинов |
| orders | Заказы |
| cart_items | Корзина |
| chats / messages | Чаты и сообщения |
| search_queries | Поисковые запросы |

---

## 4. gRPC API

### 4.1 ListingsService

**Proto:** `api/proto/listings/v1/listings.proto`

```protobuf
service ListingsService {
  // CRUD
  rpc GetListing(GetListingRequest) returns (GetListingResponse);
  rpc CreateListing(CreateListingRequest) returns (CreateListingResponse);
  rpc UpdateListing(UpdateListingRequest) returns (UpdateListingResponse);
  rpc DeleteListing(DeleteListingRequest) returns (DeleteListingResponse);

  // Query
  rpc ListListings(ListListingsRequest) returns (ListListingsResponse);
  rpc SearchListings(SearchListingsRequest) returns (SearchListingsResponse);
  rpc GetSimilarListings(GetSimilarListingsRequest) returns (GetSimilarListingsResponse);

  // Images
  rpc GetListingImage(ImageIDRequest) returns (ImageResponse);
  rpc GetListingImages(ListingIDRequest) returns (ImagesResponse);
  rpc AddListingImage(AddImageRequest) returns (ImageResponse);
  rpc DeleteListingImage(DeleteListingImageRequest) returns (DeleteListingImageResponse);
  rpc ReorderListingImages(ReorderImagesRequest) returns (ReorderImagesResponse);
  rpc UploadListingImages(stream UploadImageChunkRequest) returns (UploadImagesResponse);

  // Favorites
  rpc AddToFavorites(AddToFavoritesRequest) returns (google.protobuf.Empty);
  rpc RemoveFromFavorites(RemoveFromFavoritesRequest) returns (google.protobuf.Empty);
  rpc GetFavoritedUsers(ListingIDRequest) returns (UserIDsResponse);

  // Storefronts
  rpc CreateStorefront(CreateStorefrontRequest) returns (StorefrontResponse);
  rpc GetStorefront(GetStorefrontRequest) returns (StorefrontResponse);
  rpc UpdateStorefront(UpdateStorefrontRequest) returns (StorefrontResponse);
}
```

### 4.2 SearchService

**Proto:** `api/proto/search/v1/search.proto`

```protobuf
service SearchService {
  // Поиск
  rpc SearchListings(SearchListingsRequest) returns (SearchListingsResponse);
  rpc SearchWithFilters(SearchWithFiltersRequest) returns (SearchWithFiltersResponse);
  rpc GetSearchFacets(GetSearchFacetsRequest) returns (GetSearchFacetsResponse);

  // Автодополнение
  rpc GetSuggestions(GetSuggestionsRequest) returns (GetSuggestionsResponse);

  // Аналитика
  rpc GetPopularSearches(GetPopularSearchesRequest) returns (GetPopularSearchesResponse);
  rpc GetTrendingSearches(GetTrendingSearchesRequest) returns (TrendingSearchesResponse);
  rpc GetSearchHistory(GetSearchHistoryRequest) returns (SearchHistoryResponse);
}
```

### 4.3 CategoryServiceV2

**Proto:** `api/proto/categories/v2/categories_v2.proto`

```protobuf
service CategoryServiceV2 {
  rpc GetCategoryTreeV2(GetCategoryTreeV2Request) returns (GetCategoryTreeV2Response);
  rpc GetCategoryBySlugV2(GetCategoryBySlugV2Request) returns (GetCategoryBySlugV2Response);
  rpc GetCategoryByUUIDRequest(GetCategoryByUUIDRequest) returns (GetCategoryByUUIDResponse);
  rpc GetBreadcrumb(GetBreadcrumbRequest) returns (GetBreadcrumbResponse);
}
```

### 4.4 VariantService

**Proto:** `api/proto/variants/v1/variants.proto`

```protobuf
service VariantService {
  // CRUD
  rpc CreateVariant(CreateVariantRequest) returns (VariantResponse);
  rpc GetVariant(GetVariantRequest) returns (VariantResponse);
  rpc UpdateVariant(UpdateVariantRequest) returns (VariantResponse);
  rpc DeleteVariant(DeleteVariantRequest) returns (DeleteVariantResponse);

  // Query
  rpc ListVariants(ListVariantsRequest) returns (ListVariantsResponse);
  rpc GetVariantBySku(GetVariantBySkuRequest) returns (VariantResponse);
  rpc FindVariantByAttributes(FindVariantByAttributesRequest) returns (VariantResponse);

  // Stock
  rpc ReserveStock(ReserveStockRequest) returns (ReserveStockResponse);
  rpc ReleaseStock(ReleaseStockRequest) returns (ReleaseStockResponse);
  rpc ConfirmStockDeduction(ConfirmStockDeductionRequest) returns (ConfirmStockDeductionResponse);
}
```

### 4.5 AttributeService

**Proto:** `api/proto/attributes/v1/attributes.proto`

```protobuf
service AttributeService {
  // Admin CRUD
  rpc CreateAttribute(CreateAttributeRequest) returns (CreateAttributeResponse);
  rpc UpdateAttribute(UpdateAttributeRequest) returns (UpdateAttributeResponse);
  rpc DeleteAttribute(DeleteAttributeRequest) returns (DeleteAttributeResponse);
  rpc GetAttribute(GetAttributeRequest) returns (GetAttributeResponse);
  rpc ListAttributes(ListAttributesRequest) returns (ListAttributesResponse);

  // Category-Attribute Links
  rpc LinkAttributeToCategory(LinkAttributeToCategoryRequest) returns (LinkAttributeToCategoryResponse);
  rpc UpdateCategoryAttribute(UpdateCategoryAttributeRequest) returns (UpdateCategoryAttributeResponse);
  rpc UnlinkAttributeFromCategory(UnlinkAttributeFromCategoryRequest) returns (UnlinkAttributeFromCategoryResponse);

  // Public
  rpc GetCategoryAttributes(GetCategoryAttributesRequest) returns (GetCategoryAttributesResponse);
  rpc GetCategoryVariantAttributes(GetCategoryVariantAttributesRequest) returns (GetCategoryVariantAttributesResponse);
  rpc GetListingAttributes(GetListingAttributesRequest) returns (GetListingAttributesResponse);
  rpc SetListingAttributes(SetListingAttributesRequest) returns (SetListingAttributesResponse);

  // Validation
  rpc ValidateAttributeValues(ValidateAttributeValuesRequest) returns (ValidateAttributeValuesResponse);

  // Migration
  rpc BulkImportAttributes(BulkImportAttributesRequest) returns (BulkImportAttributesResponse);
}
```

---

## 5. Категории V2 (UUID + JSONB)

### Локализация через JSONB

```json
{
  "sr": "Elektronika",
  "en": "Electronics",
  "ru": "Электроника"
}
```

### Materialized Path

```sql
-- Root категория
path = '001'

-- Подкатегория
path = '001.002.003'
```

**Преимущества:**
- O(1) получение всех потомков: `WHERE path LIKE '001.002%'`
- O(1) уровень вложенности

### Извлечение локализации

```sql
-- Сербский
SELECT
    id,
    slug,
    name->>'sr' AS name,
    description->>'sr' AS description
FROM categories;
```

---

## 6. Унифицированные атрибуты

### Purpose-based система

| Purpose | Описание | Пример |
|---------|----------|--------|
| `regular` | Только для описания товара | Бренд, Гарантия |
| `variant` | Только для вариантов | Размер, Цвет |
| `both` | Универсальный | Состояние |

### Типы атрибутов

- `text` — текстовое поле
- `number` — числовое значение
- `select` — выбор из списка
- `multi_select` — множественный выбор
- `boolean` — да/нет
- `date` — дата
- `color` — цвет (hex)
- `size` — размер

---

## 7. Варианты товаров

### Пример: Кроссовки Nike

| SKU | Размер | Цвет | Цена | Остаток |
|-----|--------|------|------|---------|
| NIKE-42-WHITE | 42 | Белый | 9500 | 5 |
| NIKE-42-BLACK | 42 | Черный | 9500 | 3 |
| NIKE-43-WHITE | 43 | Белый | 9500 | 0 |

### SKU генерация

```go
func GenerateSKU(categoryCode, attributes string, sequence int) string {
    return fmt.Sprintf("%s-%s-%04d", categoryCode, attributes, sequence)
    // "SHOE-42-WHITE-0001"
}
```

### Автоматический available_quantity

```sql
available_quantity INTEGER GENERATED ALWAYS AS (stock_quantity - reserved_quantity) STORED
```

---

## 8. Инвентарь (FIFO)

### Типы движений

- `receive` — поступление
- `reserve` — резервирование
- `deduct` — списание
- `release` — освобождение резерва
- `adjust` — корректировка

### Stock workflow

```
Поступление +10 → stock=10
Reserve +2 → reserved=2, available=8
Confirm → stock=8, reserved=0
Release → stock=10, reserved=0
```

---

## 9. Миграции

**Всего:** 120+ миграций (180+ файлов)

### Основные

| №   | Название | Описание |
|-----|----------|----------|
| 001 | initial_schema | Базовая схема |
| 002 | c2c_schema | C2C объявления |
| 003 | add_storefronts | Витрины B2C |
| 050 | category_uuid_migration | UUID категории |
| 051 | category_jsonb_i18n | JSONB локализация |
| 070 | unified_attributes | Унифицированные атрибуты |
| 080 | product_variants_v2 | Варианты товаров |
| 090 | inventory_fifo | FIFO инвентарь |

### Применение

```bash
cd /p/github.com/vondi-global/listings
make migrate-up
make migrate-down
make migrate-status
```

---

## 10. Запуск и управление

### Запуск

```bash
# Быстрый запуск
/home/dim/.local/bin/start-listings-microservice.sh

# Остановка
/home/dim/.local/bin/stop-listings-microservice.sh

# Проверка
netstat -tlnp | grep ":50053"
tail -f /tmp/listings-microservice.log
```

### Health checks

```bash
# HTTP
curl http://localhost:8086/health

# Metrics
curl http://localhost:9093/metrics

# gRPC
grpcurl -plaintext localhost:50053 grpc.health.v1.Health/Check
```

### Docker

```bash
# Запустить
docker compose up -d

# Проверить
docker ps | grep listings

# Логи
docker logs -f listings_postgres
docker logs -f listings_redis
```

---

## 11. Директорная структура

```
listings/
├── api/proto/                  # Proto файлы (5 сервисов)
├── cmd/server/                 # Entrypoint
├── internal/
│   ├── config/                 # Конфигурация
│   ├── domain/                 # Доменная модель (DDD)
│   ├── application/            # Use cases
│   ├── infrastructure/         # Postgres, Redis, OpenSearch, MinIO
│   └── interfaces/             # gRPC + HTTP handlers
├── migrations/                 # 120+ SQL миграций
├── scripts/                    # Утилиты
├── docs/                       # Документация
├── go.mod
├── Makefile
└── docker-compose.yml
```

---

**Директория:** `/p/github.com/vondi-global/listings`
**Go версия:** 1.23
**Миграций:** 120+
**Таблиц:** 37
**Статус:** Production Ready
