# Categories Domain Passport

**Версия:** 1.0.0  
**Статус:** Production Ready  
**Последнее обновление:** 2025-12-21

---

## 1. Обзор домена

**Categories Domain** — система управления категориями товаров маркетплейса Vondi с поддержкой многоязычности, иерархической структуры и AI-детекции.

**⚠️ КРИТИЧНО: SOURCE OF TRUTH**

Listings Microservice (`listings_db:35434`) является единственным источником истины для:
- Категорий (categories)
- Атрибутов (attributes)
- Связей категория-атрибут (category_attributes)
- Значений атрибутов (attribute_values, variant_attribute_values)

**Удалённые legacy таблицы из монолита:**
- ❌ `unified_attributes` (vondi_db) - удалена 2025-12-21
- ❌ `unified_category_attributes` (vondi_db) - удалена 2025-12-21
- ❌ `unified_attribute_values` (vondi_db) - удалена 2025-12-21
- ❌ `variant_attribute_mappings` (vondi_db) - удалена 2025-12-21

### Ключевые возможности

- **UUID-based архитектура** (CategoryV2) вместо INTEGER ID
- **JSONB локализация** для 3 языков (sr, en, ru)
- **Иерархическая структура** — 3-level hierarchy (L1 → L2 → L3)
- **Materialized Path** для эффективной работы с деревом
- **AI-детекция категорий** через Claude API
- **Унифицированная система атрибутов** (Purpose: regular/variant/both)
- **SEO-оптимизация** (meta title, description, keywords)
- **Redis кеширование** с TTL 1 час

### Технологии

- **База данных:** PostgreSQL 15 (UUID, JSONB)
- **Кеш:** Redis 7
- **API:** gRPC + HTTP REST
- **Локализация:** JSONB (sr, en, ru)
- **AI:** Claude API (category detection)

---

## 2. Доменная модель

### 2.1 CategoryV2 (UUID + JSONB)

CategoryV2 — основная сущность для работы с категориями.

**Go структура:**
```go
type CategoryV2 struct {
    ID              uuid.UUID
    Slug            string
    ParentID        *uuid.UUID
    Level           int32
    Path            string  // materialized path
    SortOrder       int32
    
    // JSONB локализация
    Name            map[string]string
    Description     map[string]string
    MetaTitle       map[string]string
    MetaDescription map[string]string
    MetaKeywords    map[string]string
    
    Icon            *string
    ImageURL        *string
    IsActive        bool
    CreatedAt       time.Time
    UpdatedAt       time.Time
}
```

**Особенности:**
- UUID для глобальной уникальности
- JSONB вместо JOIN таблиц переводов
- Materialized path: O(1) получение потомков

### 2.2 LocalizedCategory (для клиента)

LocalizedCategory — категория с одним языком для frontend.

**Fallback логика локализации:**
1. Запрошенная локаль (например, "en")
2. Сербский (sr) — default
3. Английский (en)
4. Первое доступное значение
5. Пустая строка

### 2.3 CategoryTreeV2 (иерархия)

```go
type CategoryTreeV2 struct {
    Category      *LocalizedCategory
    Subcategories []*CategoryTreeV2
}
```

---

## 3. Локализация (JSONB)

### 3.1 Формат хранения

**PostgreSQL:**
```sql
name JSONB DEFAULT '{"sr": "", "en": "", "ru": ""}'
```

**Пример:**
```json
{
  "sr": "Elektronika",
  "en": "Electronics",
  "ru": "Электроника"
}
```

### 3.2 SQL запросы

**Извлечение:**
```sql
SELECT id, slug, 
       name->>'sr' AS name_sr,
       name->>'en' AS name_en
FROM categories;
```

**Фильтрация:**
```sql
SELECT * FROM categories
WHERE name->>'sr' ILIKE '%телефон%';
```

---

## 4. Materialized Path

### 4.1 Концепция

Materialized Path — хранение пути от корня в виде строки.

**Пример:**
```
Root (L0)       → path = "001"
  ├─ Child1 (L1) → path = "001.002"
  └─ Child2 (L1) → path = "001.004"
```

### 4.2 Производительность

| Операция | Сложность | SQL |
|----------|-----------|-----|
| Все потомки | O(1) | `WHERE path LIKE '001.002%'` |
| Уровень | O(1) | `level` поле |

---

## 5. Унифицированные атрибуты

### 5.1 Attribute Purpose

| Purpose | Описание | Пример |
|---------|----------|--------|
| `regular` | Только для описания товара | Бренд, Гарантия |
| `variant` | Только для вариантов | Размер, Цвет |
| `both` | Универсальный | Состояние |

### 5.2 Типы атрибутов

| Тип | Описание |
|-----|----------|
| `text` | Текстовое поле |
| `number` | Числовое значение |
| `select` | Выбор из списка |
| `multi_select` | Множественный выбор |
| `boolean` | Да/Нет |
| `date` | Дата |
| `color` | Цвет (hex) |
| `size` | Размер |

---

## 6. AI-детекция категорий

### 6.1 Методы детекции

| Метод | Accuracy |
|-------|----------|
| `ai_claude` | 0.85-0.95 |
| `keyword_match` | 0.70-0.85 |
| `similarity` | 0.60-0.80 |
| `fallback` | 0.50 |

### 6.2 Пример использования

**Запрос:**
```json
{
  "title": "iPhone 15 Pro Max 256GB",
  "description": "Novi telefon",
  "language": "sr"
}
```

**Ответ:**
```json
{
  "primary": {
    "category_name": "Pametni telefoni",
    "confidence_score": 0.92,
    "detection_method": "ai_claude"
  }
}
```

---

## 7. gRPC API

### 7.1 CategoryServiceV2

```protobuf
service CategoryServiceV2 {
  rpc GetCategoryTreeV2(...) returns (...);
  rpc GetCategoryBySlugV2(...) returns (...);
  rpc GetCategoryByUUID(...) returns (...);
  rpc GetBreadcrumb(...) returns (...);
}
```

### 7.2 AttributeService

```protobuf
service AttributeService {
  // Admin CRUD
  rpc CreateAttribute(...) returns (...);
  rpc UpdateAttribute(...) returns (...);
  rpc DeleteAttribute(...) returns (...);
  
  // Category-Attribute Links
  rpc LinkAttributeToCategory(...) returns (...);
  rpc GetCategoryAttributes(...) returns (...);
  rpc GetCategoryVariantAttributes(...) returns (...);
  
  // Validation
  rpc ValidateAttributeValues(...) returns (...);
}
```

### 7.3 CategoryDetectionService

```protobuf
service CategoryDetectionService {
  rpc DetectFromText(...) returns (...);
  rpc DetectBatch(...) returns (...);
  rpc ConfirmDetection(...) returns (...);
  rpc GetDetectionHistory(...) returns (...);
}
```

---

## 8. Use Cases

### 8.1 Получение дерева категорий

```go
resp, err := client.GetCategoryTreeV2(ctx, &Request{
    RootID:     nil,
    Locale:     "sr",
    ActiveOnly: true,
})
```

### 8.2 AI-детекция при создании листинга

```go
detection, _ := detectionSvc.DetectFromText(ctx, &Request{
    Title:    "iPhone 15 Pro",
    Language: "sr",
})

fmt.Printf("Предложена: %s (%.0f%%)\n", 
    detection.Primary.CategoryName,
    detection.Primary.ConfidenceScore * 100)
```

### 8.3 Получение атрибутов для формы

```go
attrs, _ := attrSvc.GetCategoryAttributes(ctx, &Request{
    CategoryID: categoryID,
    Locale:     "sr",
})

for _, attr := range attrs.Attributes {
    // Отрисовать форму
}
```

---

## 9. База данных

### 9.1 Таблица categories

```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(255) UNIQUE NOT NULL,
    parent_id UUID REFERENCES categories(id),
    level INTEGER DEFAULT 0,
    path VARCHAR(500) NOT NULL,
    
    name JSONB NOT NULL,
    description JSONB,
    meta_title JSONB,
    
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_categories_path ON categories(path);
CREATE INDEX idx_categories_name_sr ON categories 
    USING gin ((name->'sr') jsonb_path_ops);
```

### 9.2 Таблица attributes

```sql
CREATE TABLE attributes (
    id SERIAL PRIMARY KEY,
    code VARCHAR(100) UNIQUE NOT NULL,
    
    name JSONB NOT NULL,
    display_name JSONB NOT NULL,
    
    attribute_type VARCHAR(50) NOT NULL,
    purpose VARCHAR(20) DEFAULT 'regular',
    
    options JSONB DEFAULT '{}',
    is_searchable BOOLEAN DEFAULT false,
    is_filterable BOOLEAN DEFAULT false,
    is_required BOOLEAN DEFAULT false
);
```

### 9.3 Таблица category_attributes

```sql
CREATE TABLE category_attributes (
    id SERIAL PRIMARY KEY,
    category_id UUID NOT NULL REFERENCES categories(id),
    attribute_id INTEGER NOT NULL REFERENCES attributes(id),
    
    is_enabled BOOLEAN DEFAULT true,
    is_required BOOLEAN,
    is_filterable BOOLEAN,
    display_order INTEGER DEFAULT 0,
    
    UNIQUE(category_id, attribute_id)
);
```

---

## 10. Кеширование (Redis)

### 10.1 Ключи кеша

| Ключ | TTL |
|------|-----|
| `category:id:{uuid}` | 1 час |
| `category:slug:{slug}` | 1 час |
| `category:tree:{root_uuid}` | 1 час |
| `category:attributes:{uuid}` | 1 час |

### 10.2 Инвалидация

```go
func InvalidateCache(categoryID string) {
    redis.Del("category:id:" + categoryID)
    redis.Del("category:slug:*")
    redis.Del("category:tree:*")
}
```

---

## 11. Производительность

### 11.1 Бенчмарки

| Операция | Без кеша | С кешем | Улучшение |
|----------|----------|---------|-----------|
| GetCategoryTree | 45ms | 2ms | 22.5x |
| GetCategoryBySlug | 12ms | 1ms | 12x |
| GetCategoryAttributes | 25ms | 3ms | 8.3x |
| DetectFromText (AI) | 1200ms | N/A | — |

---

## 12. Полезные команды

### SQL запросы

```bash
# Подключиться к БД
psql "postgres://listings_user:listings_secret@localhost:35434/listings_dev_db"

# Все категории L1
SELECT id, slug, name->>'sr' FROM categories WHERE parent_id IS NULL;

# Дерево категории
SELECT id, slug, name->>'sr', level, path
FROM categories WHERE path LIKE '001%' ORDER BY path;

# Категории с атрибутами
SELECT c.slug, a.code, ca.is_required
FROM categories c
JOIN category_attributes ca ON ca.category_id = c.id
JOIN attributes a ON a.id = ca.attribute_id
WHERE c.slug = 'elektronika';
```

### Redis команды

```bash
# Очистка кеша категорий
docker exec listings_redis redis-cli DEL "category:*"
```

### gRPC тестирование

```bash
# Вызов метода
grpcurl -plaintext -d '{"locale": "sr", "active_only": true}' \
  localhost:50053 categoriessvc.v2.CategoryServiceV2/GetCategoryTreeV2
```

---

## 13. Интеграция с монолитом

### 13.1 Архитектура

```
Frontend (Next.js)
    ↓ BFF /api/v2/*
Backend Monolith (Fiber)
    ↓ gRPC
Listings Microservice
    ↓ PostgreSQL + Redis
```

### 13.2 Кеширование в монолите

Backend тоже кеширует gRPC ответы для минимизации латентности.

---

## 14. Миграции

| Миграция | Описание |
|----------|----------|
| `050_category_uuid_migration.up.sql` | INTEGER → UUID |
| `051_category_jsonb_i18n.up.sql` | VARCHAR → JSONB локализация |
| `070_unified_attributes.up.sql` | Унифицированные атрибуты |
| `071_category_attributes.up.sql` | Связи категория-атрибут |
| `075_category_detection.up.sql` | AI-детекция |

---

## 15. Контакты и ресурсы

**Директория:** `/p/github.com/vondi-global/listings`  
**Proto файлы:** `api/proto/categories/`, `api/proto/attributes/`  
**Документация:** `/p/github.com/vondi-global/.passport/services/listings.md`

**Ключевые файлы:**
- Domain: `internal/domain/category.go`
- Service: `internal/service/category_service_impl.go`
- Repository: `internal/repository/postgres/category_repo.go`
- gRPC Handler: `internal/transport/grpc/handlers_category_service.go`

**Миграции:** `listings/migrations/050-075_categories_*.sql`

---

**Статус:** Production Ready  
**Версия документа:** 1.0.0  
**Дата обновления:** 2025-12-21
