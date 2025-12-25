# OpenSearch Passport

**Version:** 2.11+
**Index:** `marketplace_listings`
**Connection:** `http://localhost:9200`
**Status:** Production (монолит), Migration to microservice in progress
**Updated:** 2024-12-21

---

## 1. Обзор OpenSearch

### 1.1 Назначение
OpenSearch используется для полнотекстового поиска объявлений (C2C marketplace) с поддержкой:
- Многоязычного поиска (Serbian, Russian, English)
- Autocomplete и suggestions
- Фасетной фильтрации (категории, цена, location, атрибуты)
- Геопоиска (geo_point для карты)
- Relevance scoring с учетом views, ratings

### 1.2 Идентификация

| Параметр | Значение |
|----------|----------|
| Версия | docker-cluster (opensearchproject/opensearch:2.11.0) |
| Endpoint | http://localhost:9200 |
| Dashboards | http://localhost:5601 |
| Username | admin |
| Password | (из .env) |
| Контейнер | opensearch |
| Статус кластера | Yellow (single-node, нет репликации) |

### 1.3 Архитектура
```
┌─────────────────┐
│ Frontend (BFF)  │
│   /api/v2/*     │
└────────┬────────┘
         │
         v
┌─────────────────┐      ┌──────────────────┐
│ Backend Monolith│─────>│  OpenSearch      │
│  (port 3000)    │      │  (port 9200)     │
│                 │      │                  │
│ /marketplace/   │      │ Index:           │
│   search        │      │ marketplace_     │
│                 │      │   listings       │
└─────────────────┘      └──────────────────┘
```

**ВАЖНО:** Планируется миграция на Search microservice (gRPC), но пока поиск работает через монолит.

### 1.4 Индексы (верифицировано 2024-11-24)
| Index Name | Документов | Размер | Статус | Назначение |
|-----------|-----------|--------|--------|-----------|
| `marketplace_listings` | 6 | 43.6kb | Production | C2C объявления (главный) |
| `marketplace_backup_20250803` | 45 | 211.2kb | Backup | Архивная копия |
| `unified_listings` | 0 | 208b | Empty | Legacy (планируется удалить) |
| `storefront_products` | - | - | Legacy | B2C товары (планируется удалить) |

---

## 2. Подключение

### 2.1 Конфигурация (.env)
```bash
# OpenSearch Configuration
OPENSEARCH_URL=http://localhost:9200
OPENSEARCH_USERNAME=admin
OPENSEARCH_PASSWORD=admin
OPENSEARCH_MARKETPLACE_INDEX=marketplace_listings
```

### 2.2 Go Client
```go
// backend/internal/storage/opensearch/client.go
import "github.com/opensearch-project/opensearch-go/v2"

client, err := opensearch.NewClient(opensearch.Config{
    Addresses: []string{"http://localhost:9200"},
    Username:  "admin",
    Password:  "admin",
})
```

### 2.3 Проверка доступности
```bash
# Cluster health
curl -s http://localhost:9200/_cluster/health | jq '.'

# Index stats
curl -s http://localhost:9200/marketplace_listings/_stats | jq '.indices.marketplace_listings'

# Document count
curl -s http://localhost:9200/marketplace_listings/_count | jq '.count'
```

---

## 3. Index Mappings: marketplace_listings

**Полная структура:** `/p/github.com/vondi-global/vondi/backend/opensearch/marketplace_mapping_v2.json` (812 строк)

### 3.1 Settings
```json
{
  "number_of_shards": 1,
  "number_of_replicas": 0,
  "analysis": {
    "analyzer": {
      "serbian_analyzer": {
        "tokenizer": "standard",
        "filter": ["lowercase", "asciifolding", "serbian_stemmer"]
      },
      "russian_analyzer": {
        "tokenizer": "standard",
        "filter": ["lowercase", "russian_stemmer"]
      },
      "english_analyzer": {
        "tokenizer": "standard",
        "filter": ["lowercase", "english_stemmer"]
      },
      "multilingual_search": {
        "tokenizer": "standard",
        "filter": ["lowercase", "asciifolding"]
      },
      "autocomplete": {
        "tokenizer": "standard",
        "filter": ["lowercase", "autocomplete_filter"]
      },
      "shingle_analyzer": {
        "tokenizer": "standard",
        "filter": ["lowercase", "shingle_filter"]
      }
    },
    "filter": {
      "autocomplete_filter": {
        "type": "edge_ngram",
        "min_gram": 1,
        "max_gram": 20
      },
      "shingle_filter": {
        "type": "shingle",
        "min_shingle_size": 2,
        "max_shingle_size": 3
      }
    }
  }
}
```

### 3.2 Основные поля документа
```json
{
  "id": 12345,
  "user_id": 789,
  "storefront_id": null,
  "category_id": 42,
  "category_path_ids": [1, 10, 42],
  "status": "active",
  "price": 25000.00,
  "old_price": 30000.00,
  "has_discount": true,
  "show_on_map": true,
  "views_count": 150,
  "review_count": 5,
  "average_rating": 4.8,
  "created_at": "2024-12-01T10:00:00Z",
  "updated_at": "2024-12-10T15:30:00Z",
  "coordinates": {
    "lat": 44.8167,
    "lon": 20.4667
  }
}
```

### 3.3 Текстовые поля (multilingual)
```json
{
  "title": {
    "type": "text",
    "analyzer": "multilingual_search",
    "fields": {
      "ru": { "type": "text", "analyzer": "russian_analyzer" },
      "sr": { "type": "text", "analyzer": "serbian_analyzer" },
      "en": { "type": "text", "analyzer": "english_analyzer" },
      "autocomplete": { "type": "text", "analyzer": "autocomplete" },
      "shingles": { "type": "text", "analyzer": "shingle_analyzer" },
      "keyword": { "type": "keyword" }
    }
  },

  "translations": {
    "properties": {
      "sr": {
        "properties": {
          "title": { "type": "text", "analyzer": "serbian_analyzer" },
          "description": { "type": "text", "analyzer": "serbian_analyzer" }
        }
      },
      "ru": {
        "properties": {
          "title": { "type": "text", "analyzer": "russian_analyzer" },
          "description": { "type": "text", "analyzer": "russian_analyzer" }
        }
      },
      "en": {
        "properties": {
          "title": { "type": "text", "analyzer": "english_analyzer" },
          "description": { "type": "text", "analyzer": "english_analyzer" }
        }
      }
    }
  }
}
```

### 3.4 Nested Objects

**Images:**
```json
{
  "images": {
    "type": "nested",
    "properties": {
      "id": { "type": "integer" },
      "file_path": { "type": "keyword" },
      "public_url": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "storage_type": { "type": "keyword" },
      "is_main": { "type": "boolean" }
    }
  }
}
```

**Attributes (для фильтрации):**
```json
{
  "attributes": {
    "type": "nested",
    "properties": {
      "attribute_id": { "type": "integer" },
      "attribute_name": { "type": "keyword" },
      "attribute_type": { "type": "keyword" },
      "display_value": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "text_value_lowercase": { "type": "keyword" },
      "numeric_value": { "type": "double" },
      "boolean_value": { "type": "boolean" },
      "unit": { "type": "text", "fields": { "keyword": { "type": "keyword" } } }
    }
  }
}
```

### 3.5 Field Types Summary
| Field Type | Usage | Examples |
|-----------|-------|----------|
| `text` | Полнотекстовый поиск | title, description |
| `keyword` | Exact match, aggregations | status, category.slug |
| `integer` | Числовые ID | id, user_id, category_id |
| `double` | Цены, рейтинги | price, average_rating |
| `long` | Большие числа | year, mileage |
| `boolean` | Флаги | has_discount, show_on_map |
| `date` | Timestamps | created_at, updated_at |
| `geo_point` | Геопоиск | coordinates |
| `nested` | Вложенные объекты | attributes, images |
| `completion` | Autocomplete | title_suggest |

---

## 4. Analyzers (Language-specific)

### 4.1 Serbian Analyzer
```json
{
  "serbian_analyzer": {
    "tokenizer": "standard",
    "filter": [
      "lowercase",
      "asciifolding",  // ć→c, š→s, ž→z
      "serbian_stemmer"
    ]
  }
}
```

**Примеры:**
- `automobil` → `automobil`
- `automobila` → `automobil` (stemming)
- `Škoda Fabia` → `skoda fabia` (asciifolding)

### 4.2 Russian Analyzer
```json
{
  "russian_analyzer": {
    "tokenizer": "standard",
    "filter": [
      "lowercase",
      "russian_stemmer"
    ]
  }
}
```

### 4.3 Autocomplete Filter (Edge N-grams)
```json
{
  "autocomplete_filter": {
    "type": "edge_ngram",
    "min_gram": 1,
    "max_gram": 20
  }
}
```

**Примеры:**
- `iPhone` → `i`, `ip`, `iph`, `ipho`, ..., `iphone`
- Поиск `iph` найдет `iPhone`

---

## 5. Query Templates

### 5.1 Basic Search (multi_match)
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "iPhone 15",
            "fields": [
              "title^3",
              "title.sr^2",
              "title.ru^2",
              "title.en^2",
              "description",
              "translations.sr.title^2",
              "translations.ru.title^2",
              "translations.en.title^2",
              "all_attributes_text"
            ],
            "type": "best_fields",
            "operator": "and",
            "fuzziness": "AUTO"
          }
        }
      ],
      "filter": [
        { "term": { "status": "active" } }
      ]
    }
  },
  "size": 20,
  "from": 0,
  "sort": [
    { "_score": { "order": "desc" } },
    { "views_count": { "order": "desc" } }
  ]
}
```

### 5.2 Category Filter
```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "category_id": 42 } }
      ]
    }
  }
}
```

### 5.3 Price Range
```json
{
  "query": {
    "bool": {
      "filter": [
        {
          "range": {
            "price": {
              "gte": 10000,
              "lte": 50000
            }
          }
        }
      ]
    }
  }
}
```

### 5.4 Geo Search (radius)
```json
{
  "query": {
    "bool": {
      "filter": [
        {
          "geo_distance": {
            "distance": "10km",
            "coordinates": {
              "lat": 44.8167,
              "lon": 20.4667
            }
          }
        }
      ]
    }
  }
}
```

### 5.5 Nested Attribute Filter
```json
{
  "query": {
    "bool": {
      "filter": [
        {
          "nested": {
            "path": "attributes",
            "query": {
              "bool": {
                "must": [
                  { "term": { "attributes.attribute_name": "color" } },
                  { "term": { "attributes.text_value_lowercase": "black" } }
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

### 5.6 Autocomplete Query
```json
{
  "suggest": {
    "title_suggest": {
      "prefix": "iph",
      "completion": {
        "field": "title_suggest",
        "size": 10,
        "skip_duplicates": true
      }
    }
  }
}
```

---

## 6. Aggregations (Facets)

### 6.1 Category Facets
```json
{
  "aggs": {
    "categories": {
      "terms": {
        "field": "category_id",
        "size": 50
      }
    }
  }
}
```

### 6.2 Price Range Histogram
```json
{
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "key": "0-10000", "to": 10000 },
          { "key": "10000-50000", "from": 10000, "to": 50000 },
          { "key": "50000-100000", "from": 50000, "to": 100000 },
          { "key": "100000+", "from": 100000 }
        ]
      }
    }
  }
}
```

### 6.3 Nested Attributes Aggregation
```json
{
  "aggs": {
    "color_facet": {
      "nested": {
        "path": "attributes"
      },
      "aggs": {
        "color_filter": {
          "filter": {
            "term": { "attributes.attribute_name": "color" }
          },
          "aggs": {
            "values": {
              "terms": {
                "field": "attributes.text_value_lowercase",
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

## 7. Reindexing Procedure

### 7.1 Python Script (Migration)
**Файл:** `/p/github.com/vondi-global/vondi/backend/migrate_opensearch_indexes.py`

```bash
# Полная миграция индексов
cd /p/github.com/vondi-global/vondi/backend
python3 migrate_opensearch_indexes.py
```

**Что делает:**
- Копирует mapping и settings из старого индекса
- Создает новый индекс с улучшенной структурой
- Переиндексирует данные через `_reindex` API
- Verifies document count

### 7.2 Bash Script (Storefront Products)
**Файл:** `/p/github.com/vondi-global/vondi/backend/reindex-products.sh`

```bash
# Переиндексация товаров витрин из PostgreSQL
./reindex-products.sh
```

### 7.3 Bulk API (manual reindex)
```bash
curl -X POST "http://localhost:9200/_bulk" \
  -H "Content-Type: application/x-ndjson" \
  --data-binary @- <<EOF
{"index":{"_index":"marketplace_listings","_id":"12345"}}
{"title":"iPhone 15","price":25000,"status":"active"}
{"index":{"_index":"marketplace_listings","_id":"12346"}}
{"title":"Samsung S24","price":28000,"status":"active"}
EOF
```

### 7.4 Go Client Bulk Insert
```go
func (c *OpenSearchClient) Execute(ctx context.Context, method, path string, body []byte) error {
    // backend/internal/storage/opensearch/client.go
    // Поддерживает /_bulk endpoint
}
```

---

## 8. Performance Tuning

### 8.1 Index Settings
```json
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "30s",
    "index.max_result_window": 10000
  }
}
```

**Рекомендации:**
- **shards=1** для малых/средних индексов (<10M документов)
- **replicas=0** для dev (НЕ для production!)
- **refresh_interval=30s** вместо 1s (снижает нагрузку)

### 8.2 Query Optimization

**Cache warming:**
```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "category_id": 42 } }
      ]
    }
  },
  "size": 0
}
```

**Request cache:**
```bash
curl -X PUT "http://localhost:9200/marketplace_listings/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index.requests.cache.enable": true}'
```

### 8.3 Pagination Best Practices
```json
{
  "query": { ... },
  "size": 20,
  "from": 0,
  "track_total_hits": 10000,
  "sort": [
    { "_score": "desc" },
    { "id": "asc" }
  ]
}
```

**ВАЖНО:**
- НЕ использовать `from > 10000` (используй search_after)
- Всегда указывай `sort` с уникальным tiebreaker
- `track_total_hits=10000` вместо `true` (экономит ресурсы)

---

## 9. Common Operations

### 9.1 Создание индекса
```bash
curl -X PUT "http://localhost:9200/marketplace_listings" \
  -H "Content-Type: application/json" \
  -d @opensearch/marketplace_mapping_v2.json
```

### 9.2 Удаление индекса
```bash
curl -X DELETE "http://localhost:9200/marketplace_listings"
```

### 9.3 Обновление mapping (добавить поле)
```bash
curl -X PUT "http://localhost:9200/marketplace_listings/_mapping" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "new_field": {
        "type": "keyword"
      }
    }
  }'
```

**ВАЖНО:** Нельзя изменить тип существующего поля! Нужен reindex.

### 9.4 Refresh Index
```bash
curl -X POST "http://localhost:9200/marketplace_listings/_refresh"
```

### 9.5 Clear Cache
```bash
# Query cache
curl -X POST "http://localhost:9200/marketplace_listings/_cache/clear?query=true"

# Request cache
curl -X POST "http://localhost:9200/marketplace_listings/_cache/clear?request=true"
```

### 9.6 Analyze API (тестирование анализаторов)
```bash
curl -X POST "http://localhost:9200/marketplace_listings/_analyze" \
  -H "Content-Type: application/json" \
  -d '{
    "analyzer": "serbian_analyzer",
    "text": "Продајем аутомобил"
  }'
```

---

## 10. Monitoring & Troubleshooting

### 10.1 Health Checks
```bash
# Cluster health
curl -s "http://localhost:9200/_cluster/health" | jq '.status'

# Index stats
curl -s "http://localhost:9200/marketplace_listings/_stats" | jq '.'

# Node stats
curl -s "http://localhost:9200/_nodes/stats" | jq '.'
```

### 10.2 Performance Metrics
```bash
# Search latency
curl -s "http://localhost:9200/marketplace_listings/_stats/search" \
  | jq '.indices.marketplace_listings.total.search.query_time_in_millis'

# Indexing rate
curl -s "http://localhost:9200/marketplace_listings/_stats/indexing" \
  | jq '.indices.marketplace_listings.total.indexing.index_total'
```

### 10.3 Slow Query Log
```bash
curl -X PUT "http://localhost:9200/marketplace_listings/_settings" \
  -H "Content-Type: application/json" \
  -d '{
    "index.search.slowlog.threshold.query.warn": "2s",
    "index.search.slowlog.threshold.fetch.warn": "1s"
  }'
```

### 10.4 Common Issues

**Yellow status:**
- Normal для single-node кластера
- Решение: `"number_of_replicas": 0`

**Out of memory:**
```bash
# Проверить heap usage
curl -s "http://localhost:9200/_cat/nodes?v&h=heap.percent,heap.current"

# Увеличить heap в docker-compose.yml
OPENSEARCH_JAVA_OPTS=-Xms2g -Xmx2g
```

**Slow queries:**
```bash
# Включить profiling
curl -X POST "http://localhost:9200/marketplace_listings/_search" \
  -d '{"profile": true, "query": {...}}'
```

---

## 11. Integration with Backend

### 11.1 Go Client
**Файл:** `/p/github.com/vondi-global/vondi/backend/internal/storage/opensearch/client.go`

```go
type OpenSearchClient struct {
    client *opensearch.Client
}

func (c *OpenSearchClient) Search(ctx context.Context, indexName string, query map[string]interface{}) ([]byte, error)
func (c *OpenSearchClient) Suggest(ctx context.Context, indexName, field, prefix string, size int) ([]byte, error)
func (c *OpenSearchClient) Execute(ctx context.Context, method, path string, body []byte) ([]byte, error)
```

### 11.2 Search Handler
**Файл:** `/p/github.com/vondi-global/vondi/backend/internal/proj/marketplace/handler/search.go`

```go
// GET /api/v1/marketplace/search
func (h *Handler) SearchListings(c *fiber.Ctx) error {
    query := c.Query("query", "")
    categoryID := c.Query("category_id", "")
    limit := c.QueryInt("limit", 20)

    // Route to microservice (future)
    result, err := h.searchViaGRPC(ctx, query, categoryID, limit, offset)
}
```

### 11.3 Testing
```bash
# Поиск через backend API
curl -s "http://localhost:3000/api/v1/marketplace/search?q=iPhone&limit=5" | jq '.'

# С авторизацией
bash -c 'TOKEN=$(cat /tmp/token); curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:3000/api/v1/marketplace/search?category_id=42" | jq'
```

---

## 12. Security & Best Practices

### 12.1 Access Control
- OpenSearch доступен ТОЛЬКО изнутри backend (не для frontend)
- Frontend получает результаты через `/api/v2/marketplace/search`
- Basic auth (admin/admin) только для dev

### 12.2 Production Recommendations
1. **Replicas:** `"number_of_replicas": 1` для HA
2. **Sharding:** 1 shard на 10-50GB данных
3. **Memory:** Минимум 2GB JVM, не более 50% RAM
4. **Monitoring:** Алерты на heap usage > 75%
5. **Backups:** Регулярные snapshots через S3
6. **Security:** Включить security plugin, HTTPS, роли

---

## 13. References

### 13.1 Files
- **Mapping:** `/p/github.com/vondi-global/vondi/backend/opensearch/marketplace_mapping_v2.json`
- **Client:** `/p/github.com/vondi-global/vondi/backend/internal/storage/opensearch/client.go`
- **Migration:** `/p/github.com/vondi-global/vondi/backend/migrate_opensearch_indexes.py`
- **Handler:** `/p/github.com/vondi-global/vondi/backend/internal/proj/marketplace/handler/search.go`

### 13.2 External Docs
- OpenSearch Official: https://opensearch.org/docs/latest/
- Go Client: https://github.com/opensearch-project/opensearch-go
- Query DSL: https://opensearch.org/docs/latest/query-dsl/

### 13.3 Related Passports
- `/p/github.com/vondi-global/.passport/services/monolith.md` - Backend API
- `/p/github.com/vondi-global/.passport/databases/vondi_db.md` - PostgreSQL (source of truth)
- `/p/github.com/vondi-global/.passport/infrastructure/storage.md` - MinIO S3

---

**Дата создания:** 2024-12-21
**Автор:** Claude Opus 4.5
**Версия:** 2.0
