# Паспорт: Search Flow

> Обновлено: 2025-12-21
> Версия: 3.0

## Общая архитектура

Система поиска Vondi построена на микросервисной архитектуре с использованием OpenSearch для полнотекстового поиска и Redis для кеширования.

```
Browser/Client
    ↓
Next.js Frontend (BFF Proxy)  ← /api/v2/marketplace/search/*
    ↓
Backend (Fiber) ← /api/v1/marketplace/search/* валидирует JWT
    ↓ gRPC
Listings Service (:50053) ← SearchService gRPC
    ↓
OpenSearch (:9200) ← индекс marketplace_listings
    ↓
Redis (:36380) ← кеширование результатов
```

---

## Основные компоненты

| Компонент | Адрес | Назначение |
|-----------|-------|-----------|
| **Frontend (BFF)** | :3001 | Next.js, прокси для поисковых запросов |
| **Backend** | :3000 | Fiber, роутинг к микросервисам |
| **Listings Service** | :50053 (gRPC), :8086 (HTTP) | Микросервис поиска и объявлений |
| **OpenSearch** | :9200 | Полнотекстовый поиск |
| **Redis** | :36380 | Кеширование результатов поиска |
| **PostgreSQL** | :35434 | БД listings (аналитика, история) |

---

## Индекс OpenSearch: marketplace_listings

### Статистика (верифицировано 2025-11-24)

| Метрика | Значение |
|---------|----------|
| Документов | 6+ |
| Размер индекса | ~50 KB |
| Shards | 1 |
| Replicas | 0 (single-node) |
| Всего поисковых запросов | 938+ |
| Среднее время запроса | 0.18ms |

### Маппинг (36 полей)

```json
{
  "mappings": {
    "properties": {
      // Идентификация
      "id": { "type": "integer" },
      "uuid": { "type": "keyword" },

      // Основные поля (с анализатором serbian_analyzer)
      "title": {
        "type": "text",
        "analyzer": "serbian_analyzer",
        "fields": {
          "autocomplete": { "type": "text", "analyzer": "autocomplete_analyzer" },
          "keyword": { "type": "keyword" },
          "trigram": { "type": "text", "analyzer": "trigram_analyzer" }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "serbian_analyzer",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },

      // SKU и цена
      "sku": { "type": "keyword" },
      "price": { "type": "float" },
      "currency": { "type": "keyword" },

      // Инвентарь
      "quantity": { "type": "integer" },
      "stock_status": { "type": "keyword" },

      // Категория (nested)
      "category_id": { "type": "integer" },
      "category": {
        "type": "nested",
        "properties": {
          "id": { "type": "integer" },
          "name": { "type": "keyword" },
          "slug": { "type": "keyword" }
        }
      },

      // Витрина (nested)
      "storefront_id": { "type": "integer" },
      "storefront": {
        "type": "nested",
        "properties": {
          "id": { "type": "integer" },
          "name": { "type": "text" },
          "slug": { "type": "keyword" }
        }
      },

      // Изображения (nested)
      "images": {
        "type": "nested",
        "properties": {
          "id": { "type": "integer" },
          "url": { "type": "keyword" },
          "thumbnail_url": { "type": "keyword" },
          "is_main": { "type": "boolean" }
        }
      },

      // Атрибуты (nested для фильтрации)
      "attributes": {
        "type": "nested",
        "properties": {
          "code": { "type": "keyword" },
          "value_text": {
            "type": "text",
            "fields": {
              "keyword": { "type": "keyword" }
            }
          },
          "value_numeric": { "type": "float" }
        }
      },

      // Локация
      "location": { "type": "geo_point" },
      "address_line1": { "type": "text" },
      "city": { "type": "keyword" },
      "postal_code": { "type": "keyword" },
      "country": { "type": "keyword" },

      // Метаданные
      "user_id": { "type": "integer" },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" },
      "published_at": { "type": "date" },

      // Статистика
      "views_count": { "type": "integer" },
      "favorites_count": { "type": "integer" },
      "popularity_score": { "type": "float" },

      // Статус и видимость
      "status": { "type": "keyword" },
      "visibility": { "type": "keyword" },
      "source_type": { "type": "keyword" },
      "document_type": { "type": "keyword" },

      // Промо
      "is_promoted": { "type": "boolean" },
      "is_featured": { "type": "boolean" },

      // Рейтинг
      "rating": { "type": "float" },
      "seller_verified": { "type": "boolean" }
    }
  }
}
```

### Анализаторы

- **Serbian Analyzer**: standard tokenizer + lowercase + asciifolding (поддержка латиницы/кириллицы)
- **Autocomplete Analyzer**: edge_ngram tokenizer + lowercase (автодополнение)
- **Trigram Analyzer**: trigram tokenizer + lowercase (исправление опечаток)

---

## Индексация

### Когда происходит индексация

1. **Создание нового объявления** → автоматическая индексация
2. **Обновление объявления** → реиндексация документа
3. **Удаление объявления** → удаление из индекса
4. **Изменение статуса** → обновление в OpenSearch
5. **Ручная переиндексация** → через скрипт

### Процесс индексации

```python
# Скрипт: /p/github.com/vondi-global/listings/scripts/reindex_listings.py

# 1. Подключение к БД listings_dev_db
# 2. Выборка всех активных listings
# 3. Формирование OpenSearch документа
# 4. Bulk индексация (батчами по 100)
# 5. Проверка результата
```

### Структура индексируемого документа

```go
type ListingDocument struct {
    ID, UUID, Title, Description, SKU, Price, Currency, Quantity, StockStatus
    Category, Storefront, Images, Attributes (nested)
    Location (geo_point), City, Country
    UserID, CreatedAt, UpdatedAt, Status, Visibility, SourceType
    ViewsCount, FavoritesCount, PopularityScore
    IsPromoted, IsFeatured, Rating, SellerVerified
}
```

### Ручная переиндексация

```bash
# Полная переиндексация всех объявлений
python3 /p/github.com/vondi-global/listings/scripts/reindex_listings.py

# Проверка количества документов
curl -s "http://localhost:9200/marketplace_listings/_count" | jq '.'

# Проверка первого документа
curl -s "http://localhost:9200/marketplace_listings/_search?size=1" | jq '.hits.hits[0]._source'
```

---

## Query DSL примеры

### 1. Простой поиск (Basic Search)

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "телефон Samsung",
            "fields": [
              "title^3",
              "title.autocomplete^2",
              "description^1",
              "brand^2"
            ],
            "type": "best_fields",
            "fuzziness": "AUTO"
          }
        }
      ],
      "filter": [
        { "term": { "status": "active" } },
        { "term": { "visibility": "public" } }
      ]
    }
  },
  "size": 20,
  "from": 0
}
```

### 2. Фильтрация по категории

```json
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": [
        { "term": { "status": "active" } },
        { "term": { "category_id": "cat-uuid-123" } }
      ]
    }
  }
}
```

### 3. Фильтрация по цене

```json
{
  "query": {
    "bool": {
      "must": { "match": { "title": "ноутбук" } },
      "filter": [
        { "term": { "status": "active" } },
        {
          "range": {
            "price": {
              "gte": 500,
              "lte": 1500
            }
          }
        }
      ]
    }
  }
}
```

### 4. Фильтрация по атрибутам (nested)

```json
{
  "query": {
    "bool": {
      "must": { "match": { "title": "автомобиль" } },
      "filter": [
        { "term": { "status": "active" } },
        {
          "nested": {
            "path": "attributes",
            "query": {
              "bool": {
                "must": [
                  { "term": { "attributes.code": "brand" } },
                  { "term": { "attributes.value_text.keyword": "BMW" } }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```

### 5. Геопоиск (в радиусе)

```json
{
  "query": {
    "bool": {
      "must": { "match": { "title": "квартира" } },
      "filter": [
        { "term": { "status": "active" } },
        {
          "geo_distance": {
            "distance": "10km",
            "location": {
              "lat": 44.787197,
              "lon": 20.457273
            }
          }
        }
      ]
    }
  }
}
```

### 6. Function Score (ранжирование)

```json
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": { "match": { "title": "iPhone" } },
          "filter": [{ "term": { "status": "active" } }]
        }
      },
      "functions": [
        {
          "filter": { "term": { "is_promoted": true } },
          "weight": 1.5
        },
        {
          "filter": { "term": { "is_featured": true } },
          "weight": 1.3
        },
        {
          "filter": { "term": { "seller_verified": true } },
          "weight": 1.2
        },
        {
          "filter": { "range": { "rating": { "gte": 4.5 } } },
          "weight": 1.1
        },
        {
          "field_value_factor": {
            "field": "popularity_score",
            "factor": 1.2,
            "modifier": "log1p",
            "missing": 0
          }
        },
        {
          "gauss": {
            "created_at": {
              "origin": "now",
              "scale": "7d",
              "decay": 0.5
            }
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

**Ranking weights:**
- Promoted listings: 1.5x
- Featured listings: 1.3x
- Verified sellers: 1.2x
- High rated (≥4.5): 1.1x
- Popularity score: logarithmic boost
- Recency: Gaussian decay (newer preferred)

---

## Faceted Search (агрегации)

### 1. Агрегация по категориям

```json
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": { "field": "category_id", "size": 20 },
      "aggs": {
        "category_details": {
          "top_hits": { "size": 1, "_source": ["category.name", "category.slug"] }
        }
      }
    }
  }
}
```

### 2. Агрегация по ценовым диапазонам

```json
{
  "size": 0,
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 100 },
          { "from": 100, "to": 500 },
          { "from": 500, "to": 1000 },
          { "from": 1000, "to": 5000 },
          { "from": 5000 }
        ]
      }
    }
  }
}
```

### 3. Агрегация по атрибутам (nested)

```json
{
  "size": 0,
  "aggs": {
    "attributes_agg": {
      "nested": {
        "path": "attributes"
      },
      "aggs": {
        "attribute_codes": {
          "terms": {
            "field": "attributes.code",
            "size": 50
          },
          "aggs": {
            "attribute_values": {
              "terms": {
                "field": "attributes.value_text.keyword",
                "size": 20
              }
            }
          }
        }
      }
    }
  }
}
```

---

## Сортировка

### Доступные поля для сортировки

`_score` (релевантность), `price`, `created_at`, `updated_at`, `popularity_score`, `views_count`, `favorites_count`, `rating`

```json
// Примеры: по релевантности, по цене, по дате, комбинированная
{ "sort": [{ "_score": "desc" }] }
{ "sort": [{ "price": "asc" }] }
{ "sort": [{ "created_at": "desc" }] }
{ "sort": [{ "popularity_score": "desc" }, { "price": "asc" }] }
```

---

## Pagination

### Offset-based pagination (limit/offset)

**Использование:** Простые листинги, до 10,000 результатов

```json
{
  "query": { "match_all": {} },
  "size": 20,   // limit
  "from": 40    // offset (page 3: 20 * 2 = 40)
}
```

**Формула:** `offset = (page - 1) * limit`

**Ограничение OpenSearch:** `from + size ≤ 10,000`

### Search After (для глубокой пагинации)

```json
// Первый запрос
{
  "query": { "match_all": {} },
  "size": 20,
  "sort": [
    { "created_at": "desc" },
    { "_id": "desc" }  // tie-breaker
  ]
}

// Следующие страницы
{
  "query": { "match_all": {} },
  "size": 20,
  "sort": [
    { "created_at": "desc" },
    { "_id": "desc" }
  ],
  "search_after": [1700000000000, "listing-uuid-123"]
}
```

---

## Autocomplete/Suggestions

### 1. Prefix Suggestions

**Endpoint:** `GET /api/v1/marketplace/search/suggestions?prefix=телеф`

**Query DSL:**

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "prefix": {
            "title.autocomplete": {
              "value": "телеф"
            }
          }
        }
      ],
      "filter": [
        { "term": { "status": "active" } }
      ]
    }
  },
  "size": 10,
  "_source": ["title"]
}
```

### 2. Term Suggestions (исправление опечаток)

```json
{
  "suggest": {
    "text": "telefon",
    "title_suggest": {
      "term": {
        "field": "title",
        "suggest_mode": "popular",
        "min_word_length": 3,
        "prefix_length": 1,
        "max_edits": 2
      }
    }
  }
}
```

### 3. Phrase Suggestions ("Did you mean?")

```json
{
  "suggest": {
    "text": "айфон 14 про макс",
    "did_you_mean": {
      "phrase": {
        "field": "title.trigram",
        "size": 3,
        "gram_size": 3,
        "direct_generator": [
          {
            "field": "title.trigram",
            "suggest_mode": "always"
          }
        ],
        "highlight": {
          "pre_tag": "<em>",
          "post_tag": "</em>"
        }
      }
    }
  }
}
```

---

## API Endpoints

### 1. Basic Search

**Endpoint:** `GET /api/v1/marketplace/search`

**Parameters:** `query`, `category_id`, `limit`, `offset`

**Response:**
```json
{
  "success": true,
  "data": {
    "listings": [{ "id", "uuid", "title", "price", "category_id", "images" }],
    "total": 45, "limit": 20, "offset": 0, "took_ms": 12, "cached": false
  }
}
```

### 2. Search with Filters

**Endpoint:** `GET /api/v1/marketplace/search/filters`

**Parameters:** query, category_id, price_min, price_max, source_type, stock_status, sort_by, sort_order, limit, offset, include_facets

### 3. Get Search Facets

**Endpoint:** `GET /api/v1/marketplace/search/facets`

### 4. Get Autocomplete Suggestions

**Endpoint:** `GET /api/v1/marketplace/search/suggestions`

### 5. Get Popular Searches

**Endpoint:** `GET /api/v1/marketplace/search/popular`

---

## Redis Caching

### Cache Keys Structure

| Cache Type | Key Pattern | TTL | Example |
|------------|-------------|-----|---------|
| Basic Search | `search:v1:{hash}` | 5 min | `search:v1:a3f2b9d...` |
| Facets | `search:facets:v1:{hash}` | 10 min | `search:facets:v1:4c8e1a...` |
| Suggestions | `search:suggestions:v1:{hash}` | 1 hour | `search:suggestions:v1:9b3f2...` |
| Popular | `search:popular:v1:{hash}` | 1 hour | `search:popular:v1:7d4a8...` |
| Filtered Search | `search:filtered:v1:{hash}` | 5 min | `search:filtered:v1:2e9c4...` |
| History | `search:history:v1:{hash}` | 24 hours | `search:history:v1:8f1a3...` |

### Cache Configuration

```go
type SearchCacheConfig struct {
    SearchTTL         time.Duration // 5 minutes
    FacetsTTL         time.Duration // 10 minutes
    SuggestionsTTL    time.Duration // 1 hour
    PopularTTL        time.Duration // 1 hour
    FilteredSearchTTL time.Duration // 5 minutes
    HistoryTTL        time.Duration // 24 hours
}
```

---

## Search Analytics

### Tracked Metrics (PostgreSQL)

Таблица `search_queries` в БД `listings_dev_db`:

```sql
CREATE TABLE search_queries (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT,
    session_id TEXT,
    query TEXT NOT NULL,
    category_id TEXT,
    filters JSONB,
    results_count INT,
    took_ms INT,
    clicked_listing_id BIGINT,
    click_position INT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Popular Searches Query

```sql
-- Топ 10 поисковых запросов за последние 24 часа
SELECT
    query,
    COUNT(*) as search_count,
    AVG(results_count) as avg_results,
    AVG(took_ms) as avg_time_ms
FROM search_queries
WHERE created_at > NOW() - INTERVAL '24 hours'
  AND query IS NOT NULL
  AND query != ''
GROUP BY query
ORDER BY search_count DESC
LIMIT 10;
```

---

## Performance Optimization

**Search Weights:** Title (5.0), TitleNgram (2.0), Description (2.0), AttributeTextValue (5.0), CarMake (5.0)

**Circuit Breaker:** maxFailures=5, resetTimeout=60s, states: Closed/Open/HalfOpen

**Retry Logic:** Exponential backoff (100ms, 200ms, 400ms), max 3 retries

---

## Troubleshooting

### Проблема: Медленный поиск (> 500ms)

**Диагностика:**

```bash
curl -s "http://localhost:9200/_cluster/health" | jq '.'
curl -s "http://localhost:9200/marketplace_listings/_count" | jq '.'
curl -s "http://localhost:9200/_cat/nodes?v&h=heap.percent,heap.current,heap.max"
```

**Решения:**

1. Увеличить память JVM: `OPENSEARCH_JAVA_OPTS=-Xms2048m -Xmx2048m`
2. Оптимизировать маппинг (keyword вместо text для точных совпадений)
3. Использовать filter вместо query где возможно (кешируется)

### Проблема: Пустые результаты поиска

**Диагностика:**

```bash
curl -s "http://localhost:9200/marketplace_listings/_search?size=1" | jq '.hits.total.value'
```

**Решения:**

1. Переиндексировать: `python3 /p/github.com/vondi-global/listings/scripts/reindex_listings.py`
2. Проверить фильтры (status, visibility)

---

## Полезные команды

```bash
# OpenSearch health check
curl -s "http://localhost:9200/_cluster/health" | jq '.'

# Количество документов
curl -s "http://localhost:9200/marketplace_listings/_count" | jq '.'

# Список индексов
curl -s "http://localhost:9200/_cat/indices?v"

# Переиндексация
python3 /p/github.com/vondi-global/listings/scripts/reindex_listings.py

# Очистка search cache
docker exec listings_redis redis-cli --scan --pattern "search:*" | head -100 | xargs -L 1 docker exec -i listings_redis redis-cli DEL

# Проверка Listings microservice
curl http://localhost:8086/health | jq '.'

# Логи Listings service
tail -f /tmp/listings-microservice.log
```

---

## См. также

- [OpenSearch Passport](/p/github.com/vondi-global/.passport/search/opensearch.md)
- [Listings Database](/p/github.com/vondi-global/.passport/databases/listings_db.md)
- [Authentication Flow](/p/github.com/vondi-global/.passport/flows/authentication.md)

---

**Последнее обновление:** 2025-12-21
**Автор:** Vondi Search Team
