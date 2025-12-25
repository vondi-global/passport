# Паспорт: Storage Infrastructure (Хранилища данных)

> Обновлено: 2025-12-21 | Версия: 2.0.0

## Обзор

Платформа Vondi использует 4 типа хранилищ данных:
1. **PostgreSQL** — реляционные базы данных (6 инстансов)
2. **Redis** — кеширование и очереди (6 инстансов)
3. **OpenSearch** — полнотекстовый поиск (1 инстанс)
4. **MinIO S3** — объектное хранилище файлов (2 инстанса)

**Общий объем хранилищ:** ~50+ GB (БД + volumes + images)

---

## 1. PostgreSQL Instances (6 баз данных)

### 1.1 Monolith DB (vondi_db)

| Параметр | Значение |
|----------|----------|
| **Назначение** | Основная БД монолита |
| **Порт** | **5433** (НЕ стандартный 5432!) |
| **Имя БД** | vondi_db |
| **Пользователь** | postgres |
| **Пароль** | mX3g1XGhMRUZEX3l |
| **Таблиц** | 160 |
| **Индексов** | 735 |
| **Размер** | ~41 MB |
| **Контейнер** | opensearch (внешняя БД) |

**Connection String:**
```bash
postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db?sslmode=disable
```

**Основные модули:**
- Marketplace (listings, categories, attributes)
- Storefronts (B2C витрины)
- Orders & Payments
- Delivery
- Users (балансы, контакты, настройки)
- Chat & Messages
- Search & Analytics

**Миграции:** `/p/github.com/vondi-global/vondi/backend/migrations/`

**Паспорт:** `.passport/databases/vondi_db.md`

---

### 1.2 Listings Microservice DB

| Параметр | Значение |
|----------|----------|
| **Назначение** | Объявления, заказы, корзины, чаты |
| **Порт** | 35434 |
| **Имя БД** | listings_dev_db |
| **Пользователь** | listings_user |
| **Пароль** | listings_secret |
| **Таблиц** | 37 |
| **Индексов** | 263 |
| **Контейнер** | listings_postgres |

**Connection String:**
```bash
postgres://listings_user:listings_secret@localhost:35434/listings_dev_db?sslmode=disable
```

**Основные таблицы:**
- listings (5 строк)
- **categories (663)** - Source of Truth для всей платформы
- **attributes (30)** - Source of Truth для всей платформы
- **category_attributes (600)** - Связи категория-атрибут
- orders (11), cart_items
- chats (6), messages (38)
- inventory_reservations (21)

**⚠️ КРИТИЧНО:**
Listings DB является единственным источником истины для категорий и атрибутов после удаления legacy таблиц из монолита (2025-12-21).

**Миграции:** `/p/github.com/vondi-global/listings/migrations/`

**Паспорт:** `.passport/databases/listings_db.md`

---

### 1.3 Delivery Microservice DB

| Параметр | Значение |
|----------|----------|
| **Назначение** | Доставка, Post Express интеграция |
| **Порт** | 35432 |
| **Имя БД** | delivery_test_db |
| **Пользователь** | delivery_user |
| **Пароль** | delivery_password |
| **Тип** | PostGIS (PostgreSQL 17 + PostGIS 3.5) |
| **Таблиц** | 18 бизнес + 36 PostGIS системных |
| **Контейнер** | delivery-postgres |

**Connection String:**
```bash
postgres://delivery_user:delivery_password@localhost:35432/delivery_test_db?sslmode=disable
```

**Основные таблицы:**
- deliveries (геоданные)
- delivery_providers, delivery_zones
- post_express_locations, post_express_offices
- delivery_tracking_events

**Миграции:** `/p/github.com/vondi-global/delivery/migrations/`

**Паспорт:** `.passport/databases/delivery_db.md`

---

### 1.4 Auth Service DB

| Параметр | Значение |
|----------|----------|
| **Назначение** | Аутентификация, пользователи, JWT, OAuth |
| **Порт** | 25432 |
| **Имя БД** | auth_db |
| **Пользователь** | auth_user |
| **Пароль** | auth_secret |
| **Таблиц** | 10 |
| **Контейнер** | auth-redis-1 |

**Connection String:**
```bash
postgres://auth_user:auth_secret@localhost:25432/auth_db?sslmode=disable
```

**Основные таблицы:**
- users (email, password_hash, roles)
- oauth_connections (Google, Facebook)
- sessions, refresh_tokens
- user_roles, permissions

**Миграции:** `/p/github.com/vondi-global/auth/migrations/`

---

### 1.5 Notifications Service DB

| Параметр | Значение |
|----------|----------|
| **Назначение** | Email, Telegram, Push, In-App уведомления |
| **Порт** | 35437 |
| **Имя БД** | notifications_db |
| **Пользователь** | notify_user |
| **Пароль** | notify_secret |
| **Таблиц** | 5 |
| **Контейнер** | notifications_postgres |

**Connection String:**
```bash
postgres://notify_user:notify_secret@localhost:35437/notifications_db?sslmode=disable
```

**Основные таблицы:**
- notifications (in-app)
- user_notification_preferences
- user_telegram_connections
- delivery_log
- notification_type_settings

**Миграции:** `/p/github.com/vondi-global/notifications/migrations/`

**Паспорт:** `.passport/services/notifications.md`

---

### 1.6 Zoho Mail Integration DB

| Параметр | Значение |
|----------|----------|
| **Назначение** | Email синхронизация, контрагенты, Knowledge Base |
| **Порт** | 5435 |
| **Имя БД** | zoho_mail |
| **Пользователь** | zoho |
| **Пароль** | zoho_secure_pass_2025 |
| **Таблиц** | 11 |
| **Контейнер** | zoho-mail-postgres |

**Connection String:**
```bash
postgres://zoho:zoho_secure_pass_2025@localhost:5435/zoho_mail?sslmode=disable
```

**Основные таблицы:**
- emails (вся переписка)
- counterparties (контрагенты)
- email_threads (обсуждения)
- attachments, activities
- email_rules

**Документация:** `/p/github.com/vondi-global/zoho-mail-integration/`

---

### PostgreSQL: Общие команды

```bash
# Проверка всех PostgreSQL инстансов
netstat -tlnp | grep -E "5433|35434|35432|25432|35437|5435"

# Размеры баз данных
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db" -c "SELECT pg_size_pretty(pg_database_size('vondi_db'));"

# Количество подключений
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db" -c "SELECT COUNT(*) FROM pg_stat_activity;"

# Активные запросы
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db" -c "SELECT pid, usename, state, query, now() - query_start AS duration FROM pg_stat_activity WHERE state != 'idle' ORDER BY duration DESC;"
```

---

## 2. Redis Instances (6 инстансов)

### 2.1 System Redis (монолит)

| Параметр | Значение |
|----------|----------|
| **Порт** | 6379 (стандартный, внешний 32768) |
| **Пароль** | Нет |
| **Контейнер** | svetu-redis-1 |
| **Назначение** | Кеширование для монолита |

**Использование:**
- Session storage
- Cache для API responses
- Rate limiting

**Подключение:**
```bash
redis-cli -p 6379 PING
redis-cli -p 6379 INFO
```

---

### 2.2 Listings Redis

| Параметр | Значение |
|----------|----------|
| **Порт** | 36380 |
| **Пароль** | redis_password |
| **Контейнер** | listings_redis |
| **Назначение** | Очереди, кеши Listings MS |

**Использование:**
- Order processing queues
- Shopping cart sessions
- WebSocket pub/sub
- Inventory locking

**Подключение:**
```bash
docker exec listings_redis redis-cli -a redis_password PING
docker exec listings_redis redis-cli -a redis_password INFO
```

**Очистка:**
```bash
docker exec listings_redis redis-cli -a redis_password FLUSHALL
```

---

### 2.3 Auth Redis

| Параметр | Значение |
|----------|----------|
| **Порт** | 26379 |
| **Пароль** | Нет |
| **Контейнер** | auth-redis-1 |
| **Назначение** | JWT blacklist, sessions |

**Использование:**
- Refresh token storage
- JWT blacklist (logout)
- OAuth state storage
- Rate limiting auth endpoints

**Подключение:**
```bash
docker exec auth-redis-1 redis-cli PING
```

---

### 2.4 Notifications Redis

| Параметр | Значение |
|----------|----------|
| **Порт** | 36382 |
| **Пароль** | Нет |
| **Контейнер** | notifications_redis |
| **Назначение** | Очереди уведомлений, rate limiting |

**Использование:**
- Email sending queue
- Telegram sending queue (rate limit 30 msg/sec)
- Scheduled notifications
- Retry logic

**Подключение:**
```bash
docker exec notifications_redis redis-cli PING
```

---

### 2.5 Admin Portal Redis

| Параметр | Значение |
|----------|----------|
| **Порт** | 6383 (внешний), 6379 (внутри) |
| **Пароль** | Нет |
| **Контейнер** | admin-portal-redis |
| **Назначение** | Кеш для Admin Portal |

**Подключение:**
```bash
redis-cli -p 6383 PING
```

---

### 2.6 Map4Craft Redis (внешний проект)

| Параметр | Значение |
|----------|----------|
| **Порт** | 37380 |
| **Контейнер** | map4craft_redis |
| **Назначение** | Личный проект (игра) |

**Примечание:** НЕ относится к Vondi платформе.

---

### Redis: Общие команды

```bash
# Проверить все Redis инстансы
docker ps | grep redis

# Проверить память
redis-cli -p 6379 INFO memory
docker exec listings_redis redis-cli -a redis_password INFO memory

# Проверить ключи
redis-cli -p 6379 KEYS "*"
docker exec listings_redis redis-cli -a redis_password KEYS "*"

# Мониторинг команд в реальном времени
redis-cli -p 6379 MONITOR

# Очистка конкретного Redis
redis-cli -p 6379 FLUSHALL
docker exec listings_redis redis-cli -a redis_password FLUSHALL
```

---

## 3. OpenSearch (1 инстанс)

| Параметр | Значение |
|----------|----------|
| **Версия** | 2.11.0 |
| **Endpoint** | http://localhost:9200 |
| **Dashboards** | http://localhost:5601 |
| **Username** | admin |
| **Контейнер** | opensearch |
| **Статус кластера** | Yellow (single-node) |

### Индексы

| Индекс | Документов | Размер | Shards | Replicas | Статус |
|--------|-----------|--------|--------|----------|--------|
| marketplace_listings | 6 | 43.6kb | 1 | 0 | Yellow |
| marketplace_backup_20250803 | 45 | 211.2kb | 1 | 1 | Yellow |
| unified_listings | 0 | 208b | 1 | 0 | Yellow |

### Конфигурация

```bash
# Backend .env
OPENSEARCH_URL=http://localhost:9200
OPENSEARCH_USERNAME=admin
OPENSEARCH_PASSWORD=<password>
OPENSEARCH_MARKETPLACE_INDEX=marketplace_listings
```

### Serbian Analyzer

```json
{
  "type": "custom",
  "tokenizer": "standard",
  "filter": ["lowercase", "asciifolding"]
}
```

Поддерживает сербский язык (латиница и кириллица).

### Маппинг marketplace_listings (36 полей)

**Основные поля:**
- `title`, `description` (text, serbian_analyzer)
- `price` (float), `currency`, `quantity` (integer)
- `category` (nested: id, name, slug)
- `storefront` (nested: id, name, slug)
- `images` (nested: url, thumbnail_url, is_main)
- `location` (geo_point)
- `status`, `visibility`, `source_type` (keyword)
- `created_at`, `updated_at`, `published_at` (date)

### Полезные команды

```bash
# Статус кластера
curl -s "http://localhost:9200/_cluster/health" | jq

# Список индексов
curl -s "http://localhost:9200/_cat/indices?v"

# Количество документов
curl -s "http://localhost:9200/marketplace_listings/_count" | jq

# Переиндексация
python3 /p/github.com/vondi-global/vondi/backend/reindex_unified.py

# Проверка анализатора
curl -s "http://localhost:9200/marketplace_listings/_analyze" -H "Content-Type: application/json" -d '{
  "analyzer": "serbian_analyzer",
  "text": "Телефон Samsung"
}' | jq
```

### Docker

```yaml
opensearch:
  image: opensearchproject/opensearch:2.11.0
  container_name: opensearch
  ports:
    - "9200:9200"
    - "5601:5601"
  environment:
    - discovery.type=single-node
    - bootstrap.memory_lock=true
    - "OPENSEARCH_JAVA_OPTS=-Xms1024m -Xmx1024m"
    - plugins.security.disabled=true
  volumes:
    - opensearch-data_dev:/usr/share/opensearch/data
```

**Паспорт:** `.passport/search/opensearch.md`

---

## 4. MinIO S3 (2 инстанса)

### 4.1 Map4Craft MinIO (основной для Vondi)

| Параметр | Значение |
|----------|----------|
| **Endpoint** | http://localhost:37000 (API) |
| **Console** | http://localhost:37001 (UI) |
| **Public URL** | https://s3.vondi.rs |
| **Access Key** | dev_backend |
| **Secret Key** | (см. .env) |
| **Контейнер** | map4craft_minio |

### Buckets (Vondi)

| Bucket | Назначение | Размер | Объектов |
|--------|------------|--------|----------|
| dimalocal-listings | C2C объявления | ~300KB | 15 |
| dimalocal-storefront-products | B2C продукты | ~250KB | 12 |
| dimalocal-chat-files | Вложения в чатах | ~150KB | 8 |

**Общий размер:** ~704KB (35 объектов)

### Конфигурация Backend

```bash
# .env
FILE_STORAGE_PROVIDER=minio
MINIO_ENDPOINT=s3.vondi.rs
MINIO_ACCESS_KEY=dev_backend
MINIO_SECRET_KEY=<secret>
MINIO_USE_SSL=true
MINIO_BUCKET_NAME=dimalocal-listings
MINIO_STOREFRONT_BUCKET=dimalocal-storefront-products
MINIO_CHAT_BUCKET=dimalocal-chat-files
MINIO_PUBLIC_URL=https://s3.vondi.rs
```

### Полезные команды

```bash
# Через mc (MinIO Client)
mc alias set vondi http://localhost:37000 dev_backend <secret>

# Список buckets
mc ls vondi/

# Список файлов
mc ls vondi/dimalocal-listings

# Загрузка файла
mc cp /path/to/file.jpg vondi/dimalocal-listings/

# Удаление файла
mc rm vondi/dimalocal-listings/file.jpg

# Статистика
mc du vondi/dimalocal-listings
```

### Прямой доступ через API

```bash
# Загрузка файла
curl -X PUT -T /path/to/file.jpg \
  http://localhost:37000/dimalocal-listings/file.jpg \
  -H "Host: localhost:37000" \
  --user "dev_backend:<secret>"

# Список объектов
curl -X GET http://localhost:37000/dimalocal-listings/ \
  --user "dev_backend:<secret>"
```

### Docker

```yaml
map4craft_minio:
  image: minio/minio:latest
  container_name: map4craft_minio
  ports:
    - "37000:9000"  # API
    - "37001:9001"  # Console
  environment:
    MINIO_ROOT_USER: dev_backend
    MINIO_ROOT_PASSWORD: <secret>
  command: server /data --console-address ":9001"
  volumes:
    - minio-data:/data
```

---

### 4.2 Vondi Agro MinIO (отдельный проект)

| Параметр | Значение |
|----------|----------|
| **Endpoint** | http://127.0.0.1:9010 |
| **Console** | http://127.0.0.1:9011 |
| **Контейнер** | vondi-agro-minio |
| **Назначение** | Спутниковые снимки, NDVI карты |

**Примечание:** НЕ используется основной платформой Vondi.

---

## 5. Backup Strategies (Стратегии резервного копирования)

### 5.1 PostgreSQL Backups

#### Автоматический дамп всех БД

```bash
#!/bin/bash
# /home/dim/.local/bin/backup-all-postgres.sh
BACKUP_DIR="/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)

# Monolith
PGPASSWORD=mX3g1XGhMRUZEX3l pg_dump -h localhost -p 5433 -U postgres -d vondi_db \
  --no-owner --no-acl --column-inserts --inserts \
  -f "$BACKUP_DIR/vondi_db_$DATE.sql"

# Listings
PGPASSWORD=listings_secret pg_dump -h localhost -p 35434 -U listings_user -d listings_dev_db \
  -f "$BACKUP_DIR/listings_$DATE.sql"

# Delivery
PGPASSWORD=delivery_password pg_dump -h localhost -p 35432 -U delivery_user -d delivery_test_db \
  -f "$BACKUP_DIR/delivery_$DATE.sql"

# Auth
PGPASSWORD=auth_secret pg_dump -h localhost -p 25432 -U auth_user -d auth_db \
  -f "$BACKUP_DIR/auth_$DATE.sql"

# Notifications
PGPASSWORD=notify_secret pg_dump -h localhost -p 35437 -U notify_user -d notifications_db \
  -f "$BACKUP_DIR/notifications_$DATE.sql"

# Zoho Mail
PGPASSWORD=zoho_secure_pass_2025 pg_dump -h localhost -p 5435 -U zoho -d zoho_mail \
  -f "$BACKUP_DIR/zoho_mail_$DATE.sql"

# Удалить дампы старше 7 дней
find "$BACKUP_DIR" -name "*.sql" -mtime +7 -delete
```

**Cron:** Запускать каждый день в 03:00

```bash
0 3 * * * /home/dim/.local/bin/backup-all-postgres.sh
```

---

### 5.2 Redis Backups

Redis использует RDB snapshots:

```bash
# Сохранить снапшот вручную
redis-cli -p 6379 SAVE
docker exec listings_redis redis-cli -a redis_password SAVE

# Копировать RDB файл
docker cp listings_redis:/data/dump.rdb /backups/redis/listings_$(date +%Y%m%d).rdb
```

**Автоматический backup:**
- Redis по умолчанию сохраняет RDB каждые 900 сек (15 мин) если изменился 1+ ключ
- Для критичных данных (sessions, tokens) используй AOF (Append Only File)

---

### 5.3 OpenSearch Snapshots

```bash
# Создать snapshot repository (один раз)
curl -X PUT "http://localhost:9200/_snapshot/backup_repo" -H "Content-Type: application/json" -d '{
  "type": "fs",
  "settings": {
    "location": "/usr/share/opensearch/snapshots"
  }
}'

# Создать snapshot
curl -X PUT "http://localhost:9200/_snapshot/backup_repo/snapshot_$(date +%Y%m%d)" -H "Content-Type: application/json" -d '{
  "indices": "marketplace_listings",
  "ignore_unavailable": true,
  "include_global_state": false
}'

# Список snapshots
curl -X GET "http://localhost:9200/_snapshot/backup_repo/_all?pretty"

# Восстановить snapshot
curl -X POST "http://localhost:9200/_snapshot/backup_repo/snapshot_20251221/_restore"
```

---

### 5.4 MinIO Backups

```bash
# Зеркалирование bucket на локальный диск
mc mirror vondi/dimalocal-listings /backups/minio/dimalocal-listings

# Синхронизация на удаленное хранилище
mc mirror vondi/dimalocal-listings s3-remote/vondi-backup/dimalocal-listings

# Backup всех buckets
for bucket in $(mc ls vondi/ | awk '{print $NF}'); do
  mc mirror "vondi/$bucket" "/backups/minio/$bucket"
done
```

**Rclone (альтернатива):**

```bash
# Backup на Google Drive
rclone sync /backups/minio gdrive:/vondi-backups/minio

# Backup на S3 AWS
rclone sync /backups/minio s3:vondi-backups/minio
```

---

## 6. Data Retention Policies (Политики хранения)

### 6.1 PostgreSQL

| Таблица | Retention | Стратегия |
|---------|-----------|-----------|
| user_behavior_events | 90 дней | Партиционирование + DROP старых |
| search_queries | 30 дней | DELETE старых записей |
| delivery_tracking_events | 180 дней | Архивация в холодное хранилище |
| analytics_events | 60 дней | Агрегация + удаление деталей |
| c2c_messages | Бессрочно | Архивация старше 1 года |
| orders | Бессрочно | Не удалять (законодательство) |
| notifications | 30 дней | DELETE прочитанных |

**Пример очистки:**

```sql
-- Удалить старые события поведения
DELETE FROM user_behavior_events WHERE created_at < NOW() - INTERVAL '90 days';

-- Удалить старые поисковые запросы
DELETE FROM search_queries WHERE created_at < NOW() - INTERVAL '30 days';

-- Архивация старых сообщений
INSERT INTO c2c_messages_archive SELECT * FROM c2c_messages WHERE created_at < NOW() - INTERVAL '365 days';
DELETE FROM c2c_messages WHERE created_at < NOW() - INTERVAL '365 days';
```

---

### 6.2 Redis

| Тип данных | TTL | Автоочистка |
|------------|-----|-------------|
| Session tokens | 24 часа | Redis EXPIRE |
| Shopping cart sessions | 7 дней | Redis EXPIRE |
| OAuth state | 10 минут | Redis EXPIRE |
| Rate limit counters | 1 час | Redis EXPIRE |
| Cache API responses | 15 минут | Redis EXPIRE |
| JWT blacklist | До истечения токена | Manual EXPIRE |

**Пример установки TTL:**

```bash
# Set key с автоудалением через 24 часа
redis-cli SETEX session:12345 86400 "session_data"

# Установить TTL на существующий ключ
redis-cli EXPIRE cart:user:67890 604800  # 7 дней
```

---

### 6.3 OpenSearch

| Индекс | Retention | Стратегия |
|--------|-----------|-----------|
| marketplace_listings | Бессрочно | Sync с БД |
| marketplace_backup_* | 30 дней | DELETE старых индексов |
| search_analytics_* | 90 дней | Rollover + ILM policy |

**Пример удаления старых индексов:**

```bash
# Удалить backup индексы старше 30 дней
curl -X DELETE "http://localhost:9200/marketplace_backup_2024*"
```

---

### 6.4 MinIO S3

| Bucket | Retention | Lifecycle Policy |
|--------|-----------|------------------|
| dimalocal-listings | Бессрочно | Только при удалении listing |
| dimalocal-chat-files | 180 дней | Автоудаление старых |
| dimalocal-storefront-products | Бессрочно | Только при удалении продукта |

**MinIO Lifecycle (пример):**

```bash
# Создать lifecycle policy
mc ilm add vondi/dimalocal-chat-files --expiry-days 180
```

---

## 7. Мониторинг и Health Checks

### 7.1 PostgreSQL

```bash
# Размер всех БД
psql -h localhost -p 5433 -U postgres -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database WHERE datname NOT IN ('template0', 'template1');"

# Количество подключений ко всем БД
psql -h localhost -p 5433 -U postgres -c "SELECT datname, COUNT(*) FROM pg_stat_activity GROUP BY datname;"

# Медленные запросы (требует pg_stat_statements)
psql -h localhost -p 5433 -U postgres -d vondi_db -c "SELECT query, calls, total_time, mean_time FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;"

# Проверка репликации (если настроена)
psql -h localhost -p 5433 -U postgres -c "SELECT * FROM pg_stat_replication;"
```

---

### 7.2 Redis

```bash
# Память всех инстансов
for port in 6379 36380 26379 36382 6383 37380; do
  echo "=== Redis :$port ==="
  redis-cli -p $port INFO memory | grep used_memory_human
done

# Количество ключей
redis-cli -p 6379 DBSIZE
docker exec listings_redis redis-cli -a redis_password DBSIZE

# Проверка hit/miss ratio
redis-cli -p 6379 INFO stats | grep keyspace
```

---

### 7.3 OpenSearch

```bash
# Health check
curl -s "http://localhost:9200/_cluster/health?pretty"

# Статистика по индексам
curl -s "http://localhost:9200/_stats?pretty"

# Использование памяти JVM
curl -s "http://localhost:9200/_cat/nodes?v&h=heap.percent,heap.current,heap.max"

# Rejected операции (признак перегрузки)
curl -s "http://localhost:9200/_cat/thread_pool?v&h=name,rejected"
```

---

### 7.4 MinIO

```bash
# Размер buckets
mc du vondi/

# Количество объектов
mc ls vondi/ | wc -l

# Health check
curl http://localhost:37000/minio/health/live

# Disk usage (внутри контейнера)
docker exec map4craft_minio df -h /data
```

---

## 8. Производительность и оптимизация

### 8.1 PostgreSQL

**Проблемы:**
1. Too many connections (лимит 100)
2. Медленные запросы (missing indexes)
3. Bloat (мертвые строки)

**Решения:**

```sql
-- Установить connection pooling в приложении
-- pgbouncer или встроенный pool в Go

-- Найти неиспользуемые индексы
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%pkey';

-- Vacuum analyze
VACUUM ANALYZE;

-- Пересборка индексов
REINDEX DATABASE vondi_db;
```

---

### 8.2 Redis

**Проблемы:**
1. Высокое потребление памяти
2. Slow commands (блокирующие операции)

**Решения:**

```bash
# Включить maxmemory policy
redis-cli CONFIG SET maxmemory 2gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# Проверить slow log
redis-cli SLOWLOG GET 10

# Анализ больших ключей
redis-cli --bigkeys
```

---

### 8.3 OpenSearch

**Проблемы:**
1. Yellow cluster (нет реплик)
2. Высокое использование heap
3. Медленный поиск

**Решения:**

```bash
# Установить replicas в 0 для single-node
curl -X PUT "http://localhost:9200/marketplace_listings/_settings" -H "Content-Type: application/json" -d '{
  "index": {"number_of_replicas": 0}
}'

# Увеличить JVM heap (в docker-compose.yml)
OPENSEARCH_JAVA_OPTS=-Xms2048m -Xmx2048m

# Force merge для оптимизации
curl -X POST "http://localhost:9200/marketplace_listings/_forcemerge?max_num_segments=1"
```

---

### 8.4 MinIO

**Проблемы:**
1. Медленная загрузка/скачивание
2. Большое количество мелких файлов

**Решения:**

```bash
# Включить compression
mc admin config set vondi compression enable=on

# Multipart uploads для больших файлов (в коде приложения)
# Используй SDK с автоматическим multipart для файлов > 5MB

# Cleanup incomplete uploads
mc rm --incomplete --recursive vondi/dimalocal-listings
```

---

## 9. Disaster Recovery (План восстановления)

### Сценарий 1: Полная потеря PostgreSQL

```bash
# 1. Остановить все сервисы
docker stop vondi-dev-backend listings_microservice delivery-service

# 2. Восстановить из последнего дампа
psql -h localhost -p 5433 -U postgres -d vondi_db < /backups/postgres/vondi_db_20251221.sql

# 3. Проверить целостность
psql -h localhost -p 5433 -U postgres -d vondi_db -c "SELECT COUNT(*) FROM listings;"

# 4. Запустить сервисы
docker start vondi-dev-backend listings_microservice delivery-service
```

---

### Сценарий 2: Потеря Redis (критичные данные)

```bash
# 1. Восстановить из RDB snapshot
docker cp /backups/redis/listings_20251221.rdb listings_redis:/data/dump.rdb

# 2. Перезапустить Redis
docker restart listings_redis

# 3. Проверить данные
docker exec listings_redis redis-cli -a redis_password DBSIZE
```

---

### Сценарий 3: Потеря OpenSearch

```bash
# 1. Переиндексация из БД
python3 /p/github.com/vondi-global/vondi/backend/reindex_unified.py

# 2. Или восстановление из snapshot
curl -X POST "http://localhost:9200/_snapshot/backup_repo/snapshot_20251221/_restore"

# 3. Проверка
curl -s "http://localhost:9200/marketplace_listings/_count" | jq
```

---

### Сценарий 4: Потеря MinIO

```bash
# 1. Восстановить из backup
mc mirror /backups/minio/dimalocal-listings vondi/dimalocal-listings

# 2. Проверить количество объектов
mc ls vondi/dimalocal-listings | wc -l

# 3. Обновить ссылки в БД (если URL изменился)
psql -h localhost -p 5433 -U postgres -d vondi_db -c "UPDATE listing_images SET image_url = REPLACE(image_url, 'old_url', 'new_url');"
```

---

## 10. Безопасность хранилищ

### 10.1 PostgreSQL

**Текущие проблемы:**
- ⚠️ Используется дефолтный пользователь `postgres`
- ⚠️ Нет SSL соединения (`sslmode=disable`)
- ⚠️ Нет роли read-only для аналитики

**Рекомендации:**

```sql
-- Создать read-only роль
CREATE ROLE readonly_user WITH LOGIN PASSWORD 'readonly_pass';
GRANT CONNECT ON DATABASE vondi_db TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- Включить SSL (в postgresql.conf)
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
```

---

### 10.2 Redis

**Текущие проблемы:**
- ⚠️ Некоторые инстансы без пароля
- ⚠️ Нет ACL (Access Control Lists)

**Рекомендации:**

```bash
# Установить requirepass (в redis.conf)
requirepass strong_password_here

# Или через команду
redis-cli CONFIG SET requirepass strong_password_here

# Использовать ACL для разных приложений (Redis 6+)
redis-cli ACL SETUSER app_user on >app_password ~app:* +@all
```

---

### 10.3 OpenSearch

**Текущие проблемы:**
- ⚠️ Security plugin отключен (`plugins.security.disabled=true`)
- ⚠️ Нет HTTPS
- ⚠️ Дефолтные креденшалы `admin/admin`

**Рекомендации:**

```bash
# Включить security plugin
plugins.security.disabled=false

# Настроить HTTPS
plugins.security.ssl.http.enabled=true

# Создать отдельные роли
# read_only: только поиск
# indexer: создание/обновление документов
# admin: полный доступ
```

---

### 10.4 MinIO

**Текущие проблемы:**
- ⚠️ Используется `dev_backend` access key
- ⚠️ Нет bucket policies (публичный доступ контролируется вручную)

**Рекомендации:**

```bash
# Создать отдельные access keys для разных сервисов
mc admin user add vondi backend_service_key backend_service_secret
mc admin policy set vondi readwrite user=backend_service_key

# Установить bucket policy (read-only для публичных файлов)
mc policy set download vondi/dimalocal-listings

# Включить versioning для критичных buckets
mc version enable vondi/dimalocal-listings
```

---

## 11. Связанные документы

| Документ | Назначение |
|----------|------------|
| `.passport/databases/vondi_db.md` | Детали монолитной БД |
| `.passport/databases/listings_db.md` | Детали Listings БД |
| `.passport/databases/delivery_db.md` | Детали Delivery БД (PostGIS) |
| `.passport/search/opensearch.md` | OpenSearch конфигурация |
| `.passport/services/notifications.md` | Notifications DB + Redis |
| `SYSTEM_PASSPORT.md` | Общий обзор системы |

---

## 12. Чек-лист для новых разработчиков

### Проверка доступа ко всем хранилищам

```bash
# PostgreSQL
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db" -c "SELECT 1;"
psql "postgres://listings_user:listings_secret@localhost:35434/listings_dev_db" -c "SELECT 1;"
psql "postgres://delivery_user:delivery_password@localhost:35432/delivery_test_db" -c "SELECT 1;"
psql "postgres://auth_user:auth_secret@localhost:25432/auth_db" -c "SELECT 1;"
psql "postgres://notify_user:notify_secret@localhost:35437/notifications_db" -c "SELECT 1;"
psql "postgres://zoho:zoho_secure_pass_2025@localhost:5435/zoho_mail" -c "SELECT 1;"

# Redis
redis-cli -p 6379 PING
docker exec listings_redis redis-cli -a redis_password PING
docker exec auth-redis-1 redis-cli PING
docker exec notifications_redis redis-cli PING
redis-cli -p 6383 PING

# OpenSearch
curl -s "http://localhost:9200/_cluster/health" | jq '.status'

# MinIO
mc alias set vondi http://localhost:37000 dev_backend <secret>
mc ls vondi/
```

**Все команды должны вернуть успешный ответ!**

---

**Дата создания:** 2025-12-21
**Автор:** Claude Code
**Версия:** 2.0.0
