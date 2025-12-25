# PVZ Service - Паспорт микросервиса

**Версия:** 1.0.0
**Статус:** In Development
**Обновлено:** 2025-12-21
**Репозиторий:** `github.com/vondi-global/pvz`
**Локальная директория:** `/p/github.com/vondi-global/pvz`

---

## 1. Обзор

PVZ Service - микросервис управления сетью Пунктов Выдачи Заказов (ПВЗ) платформы Vondi Marketplace с поддержкой двух моделей работы:

1. **Полноценные ПВЗ (Full)** - собственные точки выдачи с персоналом
2. **Mikro-ПВЗ** - партнёрские точки (кафе, магазины, салоны, аптеки, АЗС)

### Архитектура

**DDD (Domain-Driven Design)** с чистой архитектурой:
- `internal/domain/` - доменные сущности (PVZ, Operator, Partner, Handover, PickupEvent)
- `internal/service/` - бизнес-логика (shift management, capacity tracking, commission calc)
- `internal/repository/` - репозитории PostgreSQL
- `internal/grpc/` - gRPC сервер и handlers
- `internal/http/` - HTTP/REST API (Fiber framework)
- `internal/events/` - Redis Streams (pub/sub events)
- `internal/client/` - клиенты других сервисов

### Технологии

- **Язык:** Go 1.23+
- **База данных:** PostgreSQL 15 (pvz_db, порт 35438)
- **Кэш:** Redis 7 (порт 36384)
- **Web Framework:** Fiber v2
- **gRPC:** Protocol Buffers v3
- **Geo:** PostGIS (earthdistance)

### Ключевые особенности

1. **Dual Model** - Полноценные ПВЗ + Mikro-ПВЗ партнёров
2. **Operator Interface** - Смены, приёмка, выдача, возвраты
3. **Capacity Tracking** - Real-time мониторинг загрузки, auto-routing
4. **Partner Program** - Onboarding, комиссии, выплаты
5. **QR Scanning** - Приёмка от курьеров по QR коду
6. **Pickup Code** - 6-значный код для получения посылки
7. **Geo-based Routing** - Поиск ближайших ПВЗ (PostGIS)

---

## 2. Конфигурация

Все переменные окружения используют префикс `VONDIPVZ_`.

### 2.1 Service

```bash
VONDIPVZ_SERVICE_NAME=pvz-service
VONDIPVZ_SERVICE_VERSION=1.0.0
VONDIPVZ_SERVICE_ENV=development     # development | production
VONDIPVZ_LOG_LEVEL=info              # debug | info | warn | error
VONDIPVZ_LOG_FORMAT=json             # json | text
```

### 2.2 Server

```bash
VONDIPVZ_GRPC_PORT=50056
VONDIPVZ_HTTP_PORT=8089
VONDIPVZ_SERVER_READ_TIMEOUT=30s
VONDIPVZ_SERVER_WRITE_TIMEOUT=30s
VONDIPVZ_SERVER_IDLE_TIMEOUT=120s
```

### 2.3 Database

```bash
VONDIPVZ_DB_HOST=localhost
VONDIPVZ_DB_PORT=35438
VONDIPVZ_DB_NAME=pvz_db
VONDIPVZ_DB_USER=pvz_user
VONDIPVZ_DB_PASSWORD=pvz_secure_pass_2025
VONDIPVZ_DB_SSL_MODE=disable
VONDIPVZ_DB_MAX_CONNS=25
VONDIPVZ_DB_MAX_IDLE_CONNS=5
VONDIPVZ_DB_CONNECTION_MAX_LIFETIME=1h
```

**DSN:** `host=localhost port=35438 user=pvz_user password=*** dbname=pvz_db sslmode=disable`

### 2.4 Redis

```bash
VONDIPVZ_REDIS_HOST=localhost
VONDIPVZ_REDIS_PORT=36384
VONDIPVZ_REDIS_PASSWORD=
VONDIPVZ_REDIS_DB=0
VONDIPVZ_REDIS_POOL_SIZE=50
```

### 2.5 Микросервисы (интеграции)

```bash
# Auth Service
VONDIPVZ_AUTH_GRPC_ADDR=localhost:20053

# Payment Service
VONDIPVZ_PAYMENT_GRPC_ADDR=localhost:30051

# Delivery Service
VONDIPVZ_DELIVERY_GRPC_ADDR=localhost:50052

# Notification Service
VONDIPVZ_NOTIFY_GRPC_ADDR=localhost:50054

# Listings Service
VONDIPVZ_LISTINGS_GRPC_ADDR=localhost:50053

# WMS Service
VONDIPVZ_WMS_GRPC_ADDR=localhost:50055

# Monolith
VONDIPVZ_MONOLITH_HTTP_ADDR=localhost:3000
```

---

## 3. Доменная модель

### 3.1 PVZ (Пункт выдачи)

**Типы:** `full` (собственный), `mikro` (партнёрский)

**Статусы:** `active`, `inactive`, `temporary_closed`, `overloaded`

**Поля:**
- ID, Code (PVZ-BG-001), Type, Status
- Name, Address, City, Latitude, Longitude
- MaxCapacity, CurrentParcels, ReservedSlots
- WorkingHours (JSON), CommissionRate
- SupportsPickup, SupportsReturn, SupportsTryOn

**Таблица:** `pvz` (порт 35438)

### 3.2 PVZOperator (Оператор ПВЗ)

**Роли:** `operator`, `manager`

**Поля:**
- UserID (→ auth_db.users), PVZID
- Role, IsActive, CurrentShiftStart
- TotalHandovers, TotalPickups, TotalReturns

**Таблица:** `pvz_operators`

### 3.3 PartnerAccount (Партнёр)

**Статусы:** `pending`, `active`, `suspended`, `rejected`

**Типы бизнеса:** `cafe`, `shop`, `salon`, `pharmacy`, `gas_station`, `post_office`

**Поля:**
- BusinessName, BusinessType, TaxID
- ContactName, Phone, Email
- Status, BankAccount, TotalEarnings

**Таблица:** `partner_accounts`

### 3.4 ParcelAtPVZ (Посылка в ПВЗ)

**Статусы:** `awaiting_pickup`, `picked_up`, `returned`, `overdue`, `expired`

**Поля:**
- ParcelID, PVZID, OrderID, CustomerID
- PickupCode (6 цифр), PickupCodeExpiry
- ArrivedAt, StorageDeadline, StorageFee
- PickedUpAt, ReturnedAt

**Таблица:** `parcels_at_pvz`

---

## Инфраструктура

| Компонент | Порт | Переменная |
|-----------|------|------------|
| gRPC | 50056 | `VONDIPVZ_GRPC_PORT` |
| HTTP | 8089 | `VONDIPVZ_HTTP_PORT` |
| PostgreSQL | 35438 | `VONDIPVZ_DB_PORT` |
| Redis | 36384 | `VONDIPVZ_REDIS_PORT` |

### Репозиторий
```
vondi-global/pvz
```

---

## Зависимости

| Сервис | gRPC | HTTP | Назначение |
|--------|------|------|------------|
| Auth | 20053 | 28086 | JWT, роли, permissions |
| Payment | 30051 | - | COD, refunds, escrow, комиссии |
| Delivery | 50052 | 15010 | Посылки, статусы, роутинг |
| Listings | 50053 | 8086 | Товары, изображения |
| Monolith | - | 3000 | Заказы, клиенты |
| Notifications | 50054 | 8088 | SMS/Email/Push |
| WMS | 50055 | 8085 | Склады (опционально) |

---

## API Endpoints

### Operator API
```
POST /api/v1/pvz/operator/shift/start    # Начать смену
POST /api/v1/pvz/operator/shift/end      # Закончить смену
GET  /api/v1/pvz/operator/stats          # Статистика смены
GET  /api/v1/pvz/handovers/pending       # Ожидающие посылки
POST /api/v1/pvz/handovers/:id/accept    # Принять посылку
POST /api/v1/pvz/handovers/:id/reject    # Отклонить посылку
POST /api/v1/pvz/pickup/validate-code    # Проверить код выдачи
POST /api/v1/pvz/pickup/complete         # Завершить выдачу
GET  /api/v1/pvz/inventory               # Посылки в ПВЗ
```

### Returns API
```
POST /api/v1/pvz/returns                 # Инициировать возврат
GET  /api/v1/pvz/returns/:id             # Детали возврата
POST /api/v1/pvz/returns/:id/photos      # Добавить фото
POST /api/v1/pvz/returns/:id/accept      # Одобрить возврат
POST /api/v1/pvz/returns/:id/reject      # Отклонить возврат
```

### Partner API
```
POST /api/v1/partners/register           # Регистрация партнёра
GET  /api/v1/partners/me                 # Мой профиль
PUT  /api/v1/partners/me                 # Обновить профиль
GET  /api/v1/partners/me/earnings        # Мои заработки
GET  /api/v1/partners/me/payouts         # История выплат
POST /api/v1/partners/me/documents       # Загрузить документ
```

### Admin API
```
GET  /api/v1/admin/partners/pending           # Заявки партнёров
POST /api/v1/admin/partners/:id/approve       # Одобрить
POST /api/v1/admin/partners/:id/reject        # Отклонить
POST /api/v1/admin/partners/:id/suspend       # Приостановить
POST /api/v1/admin/partners/:id/payouts       # Создать выплату
GET  /api/v1/admin/pvz/analytics/network      # Статистика сети
GET  /api/v1/admin/pvz/analytics/daily/:type  # Ежедневные метрики
GET  /api/v1/admin/pvz/analytics/top-pvzs     # Рейтинг ПВЗ
GET  /api/v1/admin/pvz/analytics/top-operators # Рейтинг операторов
```

---

## База данных (pvz_db)

### Таблицы
| Таблица | Описание |
|---------|----------|
| pvz | Пункты выдачи (full и mikro) |
| pvz_operators | Операторы ПВЗ |
| operator_shifts | Смены операторов |
| partner_accounts | Аккаунты партнёров |
| partner_payouts | Выплаты партнёрам |
| partner_commissions | Комиссии за выдачи |
| parcels_at_pvz | Посылки в ПВЗ |
| handovers | Передачи от курьеров |
| pickup_events | События выдачи |
| return_requests | Запросы на возврат |

### Миграции
- `001_initial_schema` — Основные таблицы
- `002_seed_data` — Тестовые данные
- `003_partner_payouts_commissions` — Выплаты и комиссии
- `004_returns` — Возвраты
- `005_analytics_views` — Аналитические views

---

## Бизнес-логика

### Статусы посылки
```
in_transit → arrived → awaiting_pickup → picked_up
                                       → returned
                                       → overdue (>3 дней)
                                       → expired (21 день)
```

### Комиссии партнёров
| Тип бизнеса | Комиссия партнёру |
|-------------|-------------------|
| Кафе | 70% |
| Магазин | 65% |
| Салон | 70% |
| Аптека | 65% |
| АЗС | 65% |
| Почта | 75% |

### Хранение
- 3 дня бесплатно
- 20 RSD/день после 3 дней
- Максимум 21 день

### Capacity Thresholds
- Warning: 90%
- Critical: 100%
- Auto-routing при переполнении

---

## Events (Redis Streams)

> **Статус интеграции:** ✅ ПОЛНОСТЬЮ ИНТЕГРИРОВАНО (2025-12-21)

### Публикуемые события (pvz.events → Delivery Service)

Publisher: `internal/events/publisher.go`

```yaml
pvz.parcel.arrived:
  parcel_id, pvz_id, pvz_code, recipient_id, pickup_code, storage_days, accepted_by

pvz.parcel.picked_up:
  parcel_id, pvz_id, recipient_id, picked_up_by, pickup_code, storage_fee, days_stored

pvz.parcel.returned:
  parcel_id, pvz_id, recipient_id, return_reason, returned_by, days_stored

pvz.handover.accepted:
  handover_id, pvz_id, parcel_ids, courier_id, accepted_by, parcel_count

pvz.handover.rejected:
  handover_id, pvz_id, parcel_ids, courier_id, rejected_by, reason

pvz.capacity.warning:
  pvz_id, pvz_code, current_capacity, max_capacity, usage_percent, level

pvz.capacity.critical:
  pvz_id, pvz_code, current_capacity, max_capacity, usage_percent, available_slots

pvz.shift.started:
  shift_id, pvz_id, operator_id, operator_name

pvz.shift.ended:
  shift_id, pvz_id, operator_id, duration_minutes, parcels_handled

pvz.partner.registered:
  partner_id, user_id, business_name, business_type

pvz.partner.approved:
  partner_id, user_id, pvz_id, approved_by

pvz.partner.rejected:
  partner_id, user_id, rejected_by, reason
```

### Подписка (delivery.events ← Delivery Service)

Consumer: `internal/events/consumer.go` (запускается в main.go)

```yaml
delivery.parcel.routed_to_pvz:
  # PVZ Service создаёт запись Handover с ExpectedAt
  parcel_id, pvz_id, recipient_id, estimated_arrival

delivery.courier.approaching:
  # PVZ Service уведомляет оператора о приближении курьера
  courier_id, pvz_id, parcel_ids, parcel_count, eta_minutes
```

### Интеграция в main.go

```go
// PVZ Service (cmd/server/main.go)
handoverEventAdapter := newHandoverEventAdapter(handoverService, parcelRepo, logger)
eventConsumer := events.NewConsumer(redisClient, handoverEventAdapter, logger)
eventConsumer.Start(context.Background())

// Delivery Service (cmd/server/main.go)
eventHandler := events.NewDefaultPVZEventHandler(serviceLogger)
eventConsumer := events.NewConsumer(redisClient, eventHandler, serviceLogger, cfg.Events.ConsumerName)
eventConsumer.Start(ctx)
```

---

## Кеширование (Redis)

| Ключ | TTL | Описание |
|------|-----|----------|
| pvz:capacity:{id} | 30s | Загрузка ПВЗ |
| pvz:network:stats | 5min | Статистика сети |
| pvz:daily:{type}:{date} | 1h | Ежедневные метрики |

---

## Документация

- **ТЗ:** `/p/github.com/vondi-global/docs/TZ_PVZ_SERVICE.md`
- **Прогресс:** `/p/github.com/vondi-global/docs/PROGRESS_PVZ_SERVICE.md`
- **README:** `/p/github.com/vondi-global/pvz/README.md`

---

**Статус:** ✅ Production Ready (все 4 фазы завершены)
