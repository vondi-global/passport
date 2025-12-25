# Паспорт: Redis Infrastructure

> Обновлено: 2025-12-21

## Обзор архитектуры

Redis используется как:
- **Кеширование** - объявления, переводы, категории, атрибуты
- **Сессии** - JWT токены, refresh tokens, пользовательские сессии
- **Rate Limiting** - защита API от перегрузок
- **Event Streams** - асинхронные события между микросервисами (Redis Streams)
- **Pub/Sub** - real-time уведомления

**Архитектура:** Каждый микросервис имеет свой Redis инстанс для изоляции данных и производительности.

## Инстансы Redis

| Сервис | Порт | Контейнер | Пароль | Назначение |
|--------|------|-----------|--------|------------|
| **System** | 6379 | localhost | - | Монолит: кеш, переводы |
| **Auth** | 26379 | auth-redis | auth_redis_pass | Сессии, токены, rate limiting |
| **Listings** | 36379 | listings_redis | redis_password | Кеш объявлений, поиск, избранное |
| **Delivery** | 36380 | delivery-redis | - | Кеш тарифов, расчёты доставки |
| **WMS** | 36381 | wms-redis | - | Event streams, кеш задач |
| **Notifications** | 36382 | notify-redis | - | Очередь уведомлений, Pub/Sub |
| **Admin Portal** | 6380 | admin-portal-redis | - | Health checks микросервисов |

---

## Auth Redis (26379)

> **Сервис:** Auth Microservice
> **Контейнер:** auth-redis
> **Документация:** `/p/github.com/vondi-global/auth/README.md`

| Параметр | Значение |
|----------|----------|
| Порт | 26379 |
| Пароль | auth_redis_pass |
| БД | 0 |
| Контейнер | auth-redis |
| Docker Compose | `/p/github.com/vondi-global/auth/docker-compose.yml` |

### Подключение

```bash
# Через хост
redis-cli -p 26379 -a auth_redis_pass

# Через Docker
docker exec auth-redis redis-cli -a auth_redis_pass PING

# Статистика
docker exec auth-redis redis-cli -a auth_redis_pass INFO memory
docker exec auth-redis redis-cli -a auth_redis_pass DBSIZE
```

### Паттерны ключей

```
# JWT токены (access tokens)
jwt:{user_id}:{jti}
  TTL: 15 минут (900s)
  Тип: string
  Пример: jwt:123:a1b2c3d4

# Refresh токены
refresh:{user_id}:{token_hash}
  TTL: 30 дней (2592000s)
  Тип: string
  Пример: refresh:123:e5f6g7h8

# Сессии пользователей
session:{session_id}
  TTL: 24 часа (86400s)
  Тип: hash
  Поля: user_id, email, roles, created_at, last_activity

# Rate limiting (по IP)
ratelimit:{endpoint}:{ip}
  TTL: 60 секунд
  Тип: string (counter)
  Пример: ratelimit:/api/v1/auth/login:192.168.1.1

# Rate limiting (по User ID)
ratelimit:user:{user_id}:{endpoint}
  TTL: 60 секунд
  Тип: string (counter)

# Blacklist токенов (для logout)
blacklist:jwt:{jti}
  TTL: 15 минут (оставшееся время токена)
  Тип: string

# Верификационные коды (email, phone)
verification:email:{email}:{code}
  TTL: 10 минут (600s)
  Тип: string

verification:phone:{phone}:{code}
  TTL: 10 минут (600s)
  Тип: string

# Password reset tokens
reset_password:{user_id}:{token}
  TTL: 1 час (3600s)
  Тип: string
```

### Использование

- **Сессии:** JWT access/refresh токены, blacklist для logout
- **Rate Limiting:** защита от bruteforce, DDoS
- **Верификация:** email/phone коды, password reset
- **Кеш:** профили пользователей (TTL: 5m)

### Конфигурация

```yaml
# docker-compose.yml
auth-redis:
  image: redis:7-alpine
  container_name: auth-redis
  ports:
    - "26379:6379"
  command: redis-server --requirepass auth_redis_pass
  healthcheck:
    test: ["CMD", "redis-cli", "-a", "auth_redis_pass", "ping"]
    interval: 5s
    timeout: 3s
    retries: 3
  volumes:
    - auth-redis-data:/data
```

### Полезные команды

```bash
# Проверить активные сессии
docker exec auth-redis redis-cli -a auth_redis_pass KEYS "session:*"

# Очистить blacklist
docker exec auth-redis redis-cli -a auth_redis_pass KEYS "blacklist:*" | xargs redis-cli -a auth_redis_pass DEL
```

---

## Listings Redis (36379)

> **Сервис:** Listings Microservice
> **Контейнер:** listings_redis
> **Документация:** `/p/github.com/vondi-global/listings/README.md`

| Параметр | Значение |
|----------|----------|
| Порт | 36379 |
| Пароль | redis_password |
| БД | 0 |
| Контейнер | listings_redis |
| Docker Compose | `/p/github.com/vondi-global/listings/docker-compose.yml` |

### Подключение

```bash
# Через хост
redis-cli -p 36379 -a redis_password

# Через Docker
docker exec listings_redis redis-cli -a redis_password PING

# Статистика
docker exec listings_redis redis-cli -a redis_password INFO memory
docker exec listings_redis redis-cli -a redis_password DBSIZE
```

### Паттерны ключей

```
# Кеш объявлений
listing:{listing_id}
  TTL: 5 минут (300s)
  Тип: string (JSON)
  Пример: listing:12345

# Кеш поиска
search:{query_hash}
  TTL: 2 минуты (120s)
  Тип: string (JSON массив)
  Пример: search:a1b2c3d4 (SHA256 от query params)

# Избранное пользователя
favorites:user:{user_id}
  TTL: нет (персистентное)
  Тип: set
  Пример: favorites:user:123

# Кеш категорий
category:{category_id}
  TTL: 1 час (3600s)
  Тип: string (JSON)

# Кеш атрибутов категории
category:{category_id}:attributes
  TTL: 1 час (3600s)
  Тип: string (JSON массив)

# Счётчики просмотров
views:listing:{listing_id}
  TTL: 24 часа (86400s)
  Тип: string (counter)

# Кеш продавца
seller:{seller_id}
  TTL: 10 минут (600s)
  Тип: string (JSON)

# Trending объявления
trending:listings
  TTL: 15 минут (900s)
  Тип: sorted set (score = views)
```

### Использование

- **Кеш объявлений:** снижение нагрузки на PostgreSQL
- **Поиск:** кеширование результатов OpenSearch
- **Избранное:** быстрый доступ к списку избранных объявлений
- **Trending:** популярные объявления с высоким числом просмотров

### Конфигурация

```yaml
# docker-compose.yml
listings_redis:
  image: redis:7-alpine
  container_name: listings_redis
  ports:
    - "36379:6379"
  command: redis-server --requirepass redis_password
  healthcheck:
    test: ["CMD", "redis-cli", "-a", "redis_password", "ping"]
  volumes:
    - listings-redis-data:/data
```

### Полезные команды

```bash
# Очистить кеш объявлений
docker exec listings_redis redis-cli -a redis_password --scan --pattern "listing:*" | xargs redis-cli DEL

# Топ trending
docker exec listings_redis redis-cli -a redis_password ZREVRANGE trending:listings 0 9 WITHSCORES
```

---

## Delivery Redis (36380)

> **Сервис:** Delivery Microservice
> **Контейнер:** delivery-redis
> **Документация:** `/p/github.com/vondi-global/delivery/README.md`

| Параметр | Значение |
|----------|----------|
| Порт | 36380 |
| Пароль | - |
| БД | 0 |
| Контейнер | delivery-redis |
| Docker Compose | `/p/github.com/vondi-global/delivery/docker-compose.yml` |

### Подключение

```bash
# Через хост
redis-cli -p 36380

# Через Docker
docker exec delivery-redis redis-cli PING

# Статистика
docker exec delivery-redis redis-cli INFO memory
```

### Паттерны ключей

```
# Кеш тарифов Post Express
tariff:postexpress:{from_postal}:{to_postal}
  TTL: 24 часа (86400s)
  Тип: string (JSON)
  Пример: tariff:postexpress:11000:21000

# Расчёт стоимости доставки
delivery_rate:{hash}
  TTL: 1 час (3600s)
  Тип: string (JSON)
  Пример: delivery_rate:a1b2c3d4 (от from, to, weight, dimensions)

# Кеш адресов (геокодинг)
geocode:{address_hash}
  TTL: 7 дней (604800s)
  Тип: string (JSON)

# Кеш пунктов выдачи (PVZ)
pvz:city:{city_id}
  TTL: 12 часов (43200s)
  Тип: string (JSON массив)
```

### Использование

- **Тарифы:** кеширование API ответов Post Express
- **Расчёты:** кеш стоимости доставки для повторных запросов
- **Геокодинг:** кеш координат адресов
- **PVZ:** список пунктов выдачи по городам

### Конфигурация

```yaml
# docker-compose.yml
delivery-redis:
  image: redis:7-alpine
  container_name: delivery-redis
  ports:
    - "36380:6379"
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
  volumes:
    - delivery-redis-data:/data
```

### Полезные команды

```bash
# Очистить кеш тарифов
docker exec delivery-redis redis-cli --scan --pattern "tariff:*" | xargs redis-cli DEL
```

---

## WMS Redis (36381)

> **Сервис:** WMS (Warehouse Management System)
> **Контейнер:** wms-redis
> **Документация:** `/p/github.com/vondi-global/warehouse/README.md`

| Параметр | Значение |
|----------|----------|
| Порт | 36381 |
| Пароль | - |
| БД | 0 |
| Контейнер | wms-redis |
| Docker Compose | `/p/github.com/vondi-global/warehouse/docker-compose.yml` |

### Подключение

```bash
# Через хост
redis-cli -p 36381

# Через Docker
docker exec wms-redis redis-cli PING

# Статистика
docker exec wms-redis redis-cli INFO memory
```

### Паттерны ключей

```
# Event Streams (Redis Streams)
events:warehouse:{warehouse_id}
  TTL: нет (XTRIM after 1000 entries)
  Тип: stream
  Пример: events:warehouse:123

events:picking
  TTL: нет (XTRIM after 1000 entries)
  Тип: stream
  События: PickingTaskCreated, PickingTaskCompleted

events:packing
  TTL: нет
  Тип: stream
  События: PackingTaskCreated, PackingTaskCompleted

events:inventory
  TTL: нет
  Тип: stream
  События: InventoryAdjusted, StockMoved

# Кеш задач
task:picking:{task_id}
  TTL: 1 час (3600s)
  Тип: string (JSON)

task:packing:{task_id}
  TTL: 1 час (3600s)
  Тип: string (JSON)

# Worker sessions (WebSocket)
worker:session:{worker_id}
  TTL: 8 часов (28800s)
  Тип: hash
  Поля: worker_id, warehouse_id, connected_at, last_activity

# Locks (distributed locks)
lock:inventory:{product_id}
  TTL: 30 секунд
  Тип: string

lock:location:{location_id}
  TTL: 30 секунд
  Тип: string
```

### Использование

- **Events:** асинхронные события складских операций
- **Кеш задач:** быстрый доступ к задачам сборки/упаковки
- **Worker sessions:** отслеживание активных сотрудников
- **Distributed locks:** синхронизация операций инвентаря

### Redis Streams

```bash
# Читать события
docker exec wms-redis redis-cli XREAD COUNT 10 STREAMS events:picking 0
```

### Конфигурация

```yaml
# docker-compose.yml
wms-redis:
  image: redis:7-alpine
  container_name: wms-redis
  ports:
    - "36381:6379"
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
  volumes:
    - wms-redis-data:/data
```


---

## Notifications Redis (36382)

> **Сервис:** Notification Microservice
> **Контейнер:** notify-redis
> **Документация:** `/p/github.com/vondi-global/notifications/README.md`

| Параметр | Значение |
|----------|----------|
| Порт | 36382 |
| Пароль | - |
| БД | 0 |
| Контейнер | notify-redis |
| Docker Compose | `/p/github.com/vondi-global/notifications/docker-compose.yml` |

### Подключение

```bash
# Через хост
redis-cli -p 36382

# Через Docker
docker exec notify-redis redis-cli PING

# Статистика
docker exec notify-redis redis-cli INFO memory
```

### Паттерны ключей

```
# Очередь уведомлений (Pub/Sub)
notifications:queue
  TTL: нет
  Тип: list
  Пример: LPUSH notifications:queue '{"user_id":123,"type":"email",...}'

# Статус доставки
notification:delivery:{notification_id}
  TTL: 7 дней (604800s)
  Тип: string (JSON)
  Пример: notification:delivery:abc123

# Rate limiting (по каналу)
ratelimit:email:{user_id}
  TTL: 1 час (3600s)
  Тип: string (counter)

ratelimit:telegram:{user_id}
  TTL: 1 минута (60s)
  Тип: string (counter)

# Retry очередь (для failed notifications)
notifications:retry
  TTL: нет
  Тип: sorted set (score = next_retry_timestamp)

# Templates кеш
template:{template_id}:{locale}
  TTL: 1 час (3600s)
  Тип: string (HTML/text)
  Пример: template:welcome_email:sr

# User settings
settings:user:{user_id}
  TTL: 5 минут (300s)
  Тип: hash
  Поля: email_enabled, telegram_enabled, ...
```

### Использование

- **Pub/Sub:** real-time уведомления для WebSocket клиентов
- **Queue:** асинхронная обработка уведомлений
- **Rate limiting:** защита от спама (email, telegram, push)
- **Retry:** автоматический retry для failed notifications

### Pub/Sub Channels

```bash
# Подписаться на канал (для тестирования)
docker exec notify-redis redis-cli SUBSCRIBE notifications:user:123

# Отправить уведомление
docker exec notify-redis redis-cli PUBLISH notifications:user:123 '{"type":"order_shipped"}'

# Проверить активные каналы
docker exec notify-redis redis-cli PUBSUB CHANNELS
```

### Конфигурация

```yaml
# docker-compose.yml
notify-redis:
  image: redis:7-alpine
  container_name: notify-redis
  ports:
    - "36382:6379"
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
  volumes:
    - notify-redis-data:/data
```

### Полезные команды

```bash
# Проверить очередь
docker exec notify-redis redis-cli LLEN notifications:queue

# Просмотреть pending уведомления
docker exec notify-redis redis-cli LRANGE notifications:queue 0 9
```

---

## System Redis (6379)

> **Сервис:** Monolith (Legacy)
> **Контейнер:** localhost (системный)

| Параметр | Значение |
|----------|----------|
| Порт | 6379 |
| Пароль | - |
| БД | 0 |
| Контейнер | localhost (системный) |

### Подключение

```bash
redis-cli -p 6379

# Статистика
redis-cli -p 6379 INFO memory
redis-cli -p 6379 DBSIZE
```

### Паттерны ключей

```
# Переводы категорий
translations:category:{id}:{lang}:{field}
  TTL: 24 часа (86400s)
  Тип: string
  Пример: translations:category:1001:sr:name

# Кеш атрибутов
attributes:category:{category_id}
  TTL: 4 часа (14400s)
  Тип: string (JSON)

# Сессии (legacy)
session:{session_id}
  TTL: 24 часа (86400s)
  Тип: hash
```

### Использование

- Переводы категорий/атрибутов (постепенно мигрирует в Listings Redis)
- Legacy сессии (заменяются Auth Redis)

**Статус:** Постепенно выводится из эксплуатации. Новый код должен использовать специализированные Redis инстансы.

---

## Admin Portal Redis (6380)

Health checks микросервисов. `redis-cli -p 6380`

---

## Key Naming Conventions

Все ключи следуют паттерну: `namespace:entity:id[:attribute]`

### Структура

```
{namespace}:{entity}:{id}[:attribute][:sub_attribute]

namespace  - auth, listing, delivery, wms, notification
entity     - jwt, session, tariff, task, template
id         - уникальный идентификатор (user_id, listing_id, etc)
attribute  - опциональные уточнения (optional)
```

### Примеры

```
✅ Правильно
jwt:123:a1b2c3d4
favorites:user:123
tariff:postexpress:11000:21000
events:warehouse:456
template:welcome_email:sr

❌ Неправильно
user_123_jwt
listing-12345
tariff_from_11000_to_21000
```

### Префиксы по типам

| Префикс | Назначение |
|---------|------------|
| `jwt:` | JWT токены |
| `session:` | Сессии пользователей |
| `ratelimit:` | Rate limiting counters |
| `cache:` | Общий кеш |
| `lock:` | Distributed locks |
| `events:` | Redis Streams |
| `queue:` | FIFO очереди |

---

## TTL Policies

### Рекомендации по TTL

| Тип данных | TTL | Обоснование |
|------------|-----|-------------|
| JWT токены | 15 минут | Короткая жизнь access token |
| Refresh токены | 30 дней | Длительная авторизация |
| Сессии | 24 часа | Активность пользователя |
| Кеш объявлений | 5 минут | Баланс актуальности и нагрузки |
| Кеш поиска | 2 минуты | Частые изменения результатов |
| Тарифы доставки | 24 часа | Редко меняются |
| Rate limiting | 60 секунд | Временное окно для лимита |
| Верификационные коды | 10 минут | Безопасность |
| Переводы | 24 часа | Редко меняются |

### Настройка TTL в Go

```go
ctx.Set(key, value, 5*time.Minute)  // Set с TTL
ctx.Expire(key, 1*time.Hour)        // Обновить TTL
ttl := ctx.TTL(key).Val()           // Проверить TTL
```

---

## Redis Streams (Event Architecture)

Redis Streams используются для асинхронных событий между микросервисами.

### Структура событий

```go
type Event struct {
    EventID   string    // Генерируется Redis (1609459200000-0)
    Type      string    // OrderCreated, PickingTaskCompleted, etc
    AggregateID string  // ID сущности (order_id, task_id)
    Timestamp time.Time
    Payload   map[string]interface{}
}
```

### Основные стримы

| Stream | Producer | Consumer | События |
|--------|----------|----------|---------|
| `events:picking` | WMS | Dashboard, Analytics | PickingTaskCreated, Completed |
| `events:packing` | WMS | Dashboard, Analytics | PackingTaskCreated, Completed |
| `events:inventory` | WMS | Listings, Analytics | InventoryAdjusted, StockMoved |
| `events:warehouse:{id}` | WMS | Worker Apps | TaskAssigned, StatusChanged |

### Добавить событие (Go)

```go
// Publisher (WMS Service)
err := redisClient.XAdd(ctx, &redis.XAddArgs{
    Stream: "events:picking",
    Values: map[string]interface{}{
        "event_type":   "PickingTaskCompleted",
        "task_id":      taskID,
        "warehouse_id": warehouseID,
        "timestamp":    time.Now().Unix(),
    },
}).Err()
```

### Читать события (Go)

```go
// Consumer (Dashboard)
streams, err := redisClient.XRead(ctx, &redis.XReadArgs{
    Streams: []string{"events:picking", "0"},
    Count:   10,
    Block:   5 * time.Second,
}).Result()

for _, msg := range streams[0].Messages {
    eventType := msg.Values["event_type"].(string)
    taskID := msg.Values["task_id"].(string)
    // Process event...
}
```

### Consumer Groups

```bash
# Создать group
docker exec wms-redis redis-cli XGROUP CREATE events:picking dashboard-group 0 MKSTREAM

# Trim (в Go)
redisClient.XTrimMaxLen(ctx, "events:picking", 1000)
```

---

## Подключение из кода

### Go (github.com/redis/go-redis/v9)

```go
import "github.com/redis/go-redis/v9"

// Auth Redis (с паролем)
authRedis := redis.NewClient(&redis.Options{
    Addr:     "localhost:26379",
    Password: "auth_redis_pass",
    DB:       0,
})

// Listings Redis (с паролем)
listingsRedis := redis.NewClient(&redis.Options{
    Addr:     "localhost:36379",
    Password: "redis_password",
    DB:       0,
})

// WMS Redis (без пароля)
wmsRedis := redis.NewClient(&redis.Options{Addr: "localhost:36381"})

// Операции
authRedis.Set(ctx, "jwt:123:abc", tokenData, 15*time.Minute)
val, _ := authRedis.Get(ctx, "jwt:123:abc").Result()
authRedis.HSet(ctx, "session:xyz", "user_id", 123)
```

### TypeScript (ioredis)

```typescript
const redis = new Redis({host: 'localhost', port: 36379, password: 'redis_password'});

await redis.set('listing:123', JSON.stringify(data), 'EX', 300);
const cached = await redis.get('listing:123');
```

---

## Мониторинг и обслуживание

### Проверка всех инстансов

```bash
for port in 6379 26379 36379 36380 36381 36382 6380; do
  echo "=== Redis $port ==="
  redis-cli -p $port PING 2>/dev/null || echo "Offline"
done
```

### Статистика

```bash
# Память
redis-cli -p 26379 -a auth_redis_pass INFO memory | grep used_memory_human

# Большие ключи
redis-cli -p 26379 -a auth_redis_pass --bigkeys

# Slowlog
redis-cli -p 26379 -a auth_redis_pass SLOWLOG GET 10
```

### Backup

```bash
# Snapshot
redis-cli -p 26379 -a auth_redis_pass BGSAVE

# Restore
docker cp backup.rdb auth-redis:/data/dump.rdb && docker restart auth-redis
```

---

## Troubleshooting

### Проблема: "Could not connect to Redis at localhost:26379"

```bash
# 1. Проверить что контейнер запущен
docker ps | grep redis

# 2. Проверить порты
netstat -tlnp | grep -E "6379|26379|36379|36380|36381|36382"

# 3. Проверить логи контейнера
docker logs auth-redis
docker logs listings_redis

# 4. Перезапустить контейнер
docker restart auth-redis
```

### Проблема: "NOAUTH Authentication required"

```bash
# Указать пароль
redis-cli -p 26379 -a auth_redis_pass PING

# Или через Docker
docker exec auth-redis redis-cli -a auth_redis_pass PING
```

### Проблема: "OOM command not allowed when used memory > 'maxmemory'"

```bash
# Проверить maxmemory
redis-cli -p 26379 -a auth_redis_pass CONFIG GET maxmemory

# Увеличить maxmemory (временно)
redis-cli -p 26379 -a auth_redis_pass CONFIG SET maxmemory 512mb

# Или в docker-compose.yml
command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru

# Проверить eviction policy
redis-cli -p 26379 -a auth_redis_pass CONFIG GET maxmemory-policy

# Изменить policy
redis-cli -p 26379 -a auth_redis_pass CONFIG SET maxmemory-policy allkeys-lru
```

### Проблема: Высокая фрагментация

```bash
# Если mem_fragmentation_ratio > 1.5
redis-cli -p 26379 -a auth_redis_pass CONFIG SET activedefrag yes
```

### Проблема: Медленные команды

```bash
# ✅ SCAN вместо KEYS
redis-cli --scan --pattern "user:*"

# ✅ FLUSHALL ASYNC вместо FLUSHALL
redis-cli FLUSHALL ASYNC
```

---

## Best Practices

### 1. TTL для всех ключей

```go
// ❌ Без TTL (memory leak!)
redis.Set(ctx, "listing:123", data, 0)

// ✅ Всегда с TTL
redis.Set(ctx, "listing:123", data, 5*time.Minute)
```

### 2. Консистентные паттерны именования

```go
// ✅ Правильно
key := fmt.Sprintf("jwt:%d:%s", userID, jti)

// ❌ Неправильно
key := fmt.Sprintf("user_%d_token_%s", userID, jti)
```

### 3. Размер значений < 100 KB

```go
// Для больших данных используй compression
compressed := gzip.Compress(largeJSON)
redis.Set(ctx, key, compressed, ttl)
```

### 4. Pipeline для batch операций

```go
pipe := redis.Pipeline()
for _, id := range ids {
    pipe.Get(ctx, fmt.Sprintf("listing:%d", id))
}
pipe.Exec(ctx)
```

### 5. Graceful degradation

```go
listing, err := getFromCache(ctx, id)
if err != nil {
    listing, err = getFromDB(ctx, id)
}
```

### 6. Мониторинг

- `used_memory`, `connected_clients`, `evicted_keys`
- `keyspace_hits` / `keyspace_misses`
- `mem_fragmentation_ratio`

---

## Docker Compose Examples

```yaml
# Auth Redis (с паролем)
auth-redis:
  image: redis:7-alpine
  ports: ["26379:6379"]
  command: redis-server --requirepass auth_redis_pass --maxmemory 256mb

# WMS Redis (noeviction для критичных событий)
wms-redis:
  image: redis:7-alpine
  ports: ["36381:6379"]
  command: redis-server --maxmemory 512mb --maxmemory-policy noeviction
```

---

## Миграция на специализированные Redis

1. Auth данные (sessions, tokens) → Auth Redis (26379)
2. Listings кеш → Listings Redis (36379)
3. Delivery тарифы → Delivery Redis (36380)

---

## Ссылки

- [Redis Documentation](https://redis.io/docs/)
- [Redis Commands Reference](https://redis.io/commands)
- [Redis Best Practices](https://redis.io/docs/manual/patterns/)
- [Redis Streams Tutorial](https://redis.io/docs/manual/data-types/streams/)
- [go-redis Documentation](https://redis.uptrace.dev/)
- [IORedis Documentation](https://github.com/redis/ioredis)

---

**Последнее обновление:** 2025-12-21
**Автор:** Vondi DevOps Team
