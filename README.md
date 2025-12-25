# Паспорт Маркетплейса Vondi

> **Версия:** 2.0.0 | **Дата:** 2025-12-25 | **Статус:** ACTIVE - WMS PRIME 100% COMPLETE ✅

Исчерпывающая техническая документация платформы Vondi Marketplace и системы WMS Prime.

---

## Навигация

### Архитектура
- [ARCHITECTURE.md](./ARCHITECTURE.md) — Общая архитектура системы
- [GLOSSARY.md](./GLOSSARY.md) — Глоссарий терминов

### Микросервисы

#### Core Marketplace Services
| Сервис | Документ | gRPC Port | HTTP Port |
|--------|----------|-----------|-----------|
| Backend Monolith | [services/monolith.md](./services/monolith.md) | - | 3000 |
| Auth Service | [services/auth.md](./services/auth.md) | 20053 | 28086 |
| Listings Service | [services/listings.md](./services/listings.md) | 50053 | 8084 |
| Delivery Service | [services/delivery.md](./services/delivery.md) | 50052 | 8082 |
| Payment Service | [services/payment.md](./services/payment.md) | 30051 | 8087 |
| Notifications | [services/notifications.md](./services/notifications.md) | 50054 | 8088 |

#### WMS Prime Services
| Сервис | Документ | gRPC Port | HTTP Port |
|--------|----------|-----------|-----------|
| Warehouse/WMS | [services/warehouse.md](./services/warehouse.md) | 50055 | 8085 |
| PVZ Service | [services/pvz.md](./services/pvz.md) | 50056 | 8089 |
| Fraud Scoring | [services/fraud-scoring.md](./services/fraud-scoring.md) | 50058 | 8094 |
| ML Forecasting | [services/ml-forecasting.md](./services/ml-forecasting.md) | - | 8093 |
| WMS Gateway | [services/wms-gateway.md](./services/wms-gateway.md) | 50057 | 8085 |
| 3PL Gateway | [services/3pl-gateway.md](./services/3pl-gateway.md) | - | 8090 |
| Channel Adapters | [services/channel-adapters.md](./services/channel-adapters.md) | - | various |

### Frontend

#### Marketplace Applications
| Документ | Описание | Port |
|----------|----------|------|
| [frontend/svetu-web.md](./frontend/svetu-web.md) | Next.js 15 главное приложение | 3001 |
| [frontend/components.md](./frontend/components.md) | Компоненты и хуки | - |
| [frontend/state-management.md](./frontend/state-management.md) | Redux + React Query | - |

#### WMS Prime Portals
| Portal | Документ | Port | Назначение |
|--------|----------|------|------------|
| Developer Portal | [frontend/wms-dev-portal.md](./frontend/wms-dev-portal.md) | 3011 | API документация, OAuth2, WebHooks |
| Owner Portal | [frontend/wms-owner-portal.md](./frontend/wms-owner-portal.md) | 3004 | Управление бизнесом, аналитика, SLA |
| Partner Portal | [frontend/wms-partner-portal.md](./frontend/wms-partner-portal.md) | 3009 | Партнёрская программа, фулфилмент |
| PVZ Operator Portal | [frontend/wms-pvz-operator-portal.md](./frontend/wms-pvz-operator-portal.md) | 3003 | Работа с выдачей заказов в ПВЗ |
| Worker PWA | [frontend/wms-worker-pwa.md](./frontend/wms-worker-pwa.md) | 5174 | Мобильное приложение для кладовщиков |
| Admin Portal | [frontend/wms-admin-portal.md](./frontend/wms-admin-portal.md) | 3010 | Системное администрирование |

### Базы данных

#### PostgreSQL
| База данных | Документ | Port |
|-------------|----------|------|
| vondi_db (Monolith) | [databases/vondi_db.md](./databases/vondi_db.md) | 5433 |
| auth_db | [databases/auth_db.md](./databases/auth_db.md) | 25432 |
| listings_db | [databases/listings_db.md](./databases/listings_db.md) | 35434 |
| delivery_db | [databases/delivery_db.md](./databases/delivery_db.md) | 35432 |
| payment_db | [databases/payment_db.md](./databases/payment_db.md) | 35433 |
| wms_db | [databases/wms_db.md](./databases/wms_db.md) | 35435 |
| pvz_db | [databases/pvz_db.md](./databases/pvz_db.md) | 35438 |

#### Analytics & Caching
| База данных | Документ | Port |
|-------------|----------|------|
| ClickHouse (ML Forecasting) | [databases/clickhouse.md](./databases/clickhouse.md) | 8123 (HTTP), 9000 (Native) |
| Redis instances | [databases/redis.md](./databases/redis.md) | 26379-36384 |

### Интеграции

#### Internal APIs
| Интеграция | Документ |
|------------|----------|
| gRPC контракты | [integrations/grpc-contracts.md](./integrations/grpc-contracts.md) |
| REST API | [integrations/rest-api.md](./integrations/rest-api.md) |
| Events | [integrations/events.md](./integrations/events.md) |

#### External Services
| Интеграция | Документ |
|------------|----------|
| Post Express (Доставка) | [integrations/external/post-express.md](./integrations/external/post-express.md) |
| MinIO/S3 (Хранилище файлов) | [integrations/external/minio-s3.md](./integrations/external/minio-s3.md) |
| OpenSearch (Поиск) | [search/opensearch.md](./search/opensearch.md) |

#### Marketplace Channel Adapters (3PL)
| Канал | Документ | Статус |
|-------|----------|--------|
| eBay | [integrations/channels/ebay.md](./integrations/channels/ebay.md) | ✅ Active |
| Amazon SP-API | [integrations/channels/amazon.md](./integrations/channels/amazon.md) | ✅ Active |
| Etsy | [integrations/channels/etsy.md](./integrations/channels/etsy.md) | ✅ Active |
| Shopify | [integrations/channels/shopify.md](./integrations/channels/shopify.md) | ✅ Active |

### Бизнес-домены
| Домен | Документ |
|-------|----------|
| Пользователи | [domains/users.md](./domains/users.md) |
| Объявления | [domains/listings.md](./domains/listings.md) |
| Заказы | [domains/orders.md](./domains/orders.md) |
| Платежи | [domains/payments.md](./domains/payments.md) |
| Доставка | [domains/delivery.md](./domains/delivery.md) |
| Склад | [domains/warehouse.md](./domains/warehouse.md) |
| Категории | [domains/categories.md](./domains/categories.md) |

### Бизнес-процессы
| Процесс | Документ |
|---------|----------|
| Аутентификация | [flows/authentication.md](./flows/authentication.md) |
| Создание листинга | [flows/listing-creation.md](./flows/listing-creation.md) |
| Жизненный цикл заказа | [flows/order-lifecycle.md](./flows/order-lifecycle.md) |
| Платёжный поток | [flows/payment-flow.md](./flows/payment-flow.md) |
| Fulfillment | [flows/fulfillment.md](./flows/fulfillment.md) |
| Возвраты | [flows/returns.md](./flows/returns.md) |
| Поиск | [flows/search.md](./flows/search.md) |
| Загрузка файлов | [flows/file-upload.md](./flows/file-upload.md) |

### Инфраструктура
| Документ | Описание |
|----------|----------|
| [infrastructure/docker.md](./infrastructure/docker.md) | Docker конфигурация |
| [infrastructure/network.md](./infrastructure/network.md) | Сетевая архитектура |
| [infrastructure/storage.md](./infrastructure/storage.md) | Хранилища данных |
| [infrastructure/ci-cd.md](./infrastructure/ci-cd.md) | CI/CD pipelines |

### Безопасность
| Документ | Описание |
|----------|----------|
| [security/authentication.md](./security/authentication.md) | JWT, OAuth, Firebase |
| [security/authorization.md](./security/authorization.md) | RBAC, permissions |
| [security/data-protection.md](./security/data-protection.md) | Шифрование, GDPR |

---

## Быстрый старт

### Порты системы

#### Marketplace Core
```
Frontend:      3001 (Next.js)
Backend:       3000 (Go/Fiber)
Auth:          28086/20053 (HTTP/gRPC)
Listings:      8084/50053
Delivery:      8082/50052
Payment:       8087/30051
Notifications: 8088/50054
```

#### WMS Prime Services
```
WMS Gateway:       8085/50057 (HTTP/gRPC)
PVZ Service:       8089/50056 (HTTP/gRPC)
Fraud Scoring:     8094/50058 (HTTP/gRPC)
ML Forecasting:    8093 (HTTP/FastAPI)
3PL Gateway:       8090 (HTTP)
Channel Adapters:  varies (eBay, Amazon, Etsy, Shopify)
```

#### WMS Prime Portals
```
Developer Portal:     3011 (Next.js)
Owner Portal:         3004 (Next.js)
Partner Portal:       3009 (Next.js)
PVZ Operator Portal:  3003 (Next.js)
Worker PWA:           5174 (Vite/React)
Admin Portal:         3010 (Next.js)
```

### Базы данных

#### PostgreSQL
```
Monolith DB:   localhost:5433  (vondi_db)
Auth DB:       localhost:25432 (auth_db)
Listings DB:   localhost:35434 (listings_dev_db)
Delivery DB:   localhost:35432 (delivery_db)
Payment DB:    localhost:35433 (payment_db)
WMS DB:        localhost:35435 (wms_db)
PVZ DB:        localhost:35438 (pvz_db)
```

#### Analytics
```
ClickHouse:    localhost:8123 (HTTP), localhost:9000 (Native)
               Databases: default, ml_forecasting
```

#### Caching (Redis)
```
Auth Redis:       localhost:26379
Listings Redis:   localhost:36379
Delivery Redis:   localhost:36380
WMS Redis:        localhost:36381
PVZ Redis:        localhost:36384
ML Forecast Cache: localhost:36385
```

---

## Статистика системы

### Архитектура
| Метрика | Marketplace | WMS Prime | **Всего** |
|---------|-------------|-----------|-----------|
| Микросервисов | 6 | 7 | **13** |
| Frontend Приложений | 1 | 6 | **7** |
| Баз данных (PostgreSQL) | 5 | 2 | **7** |
| Analytics DB (ClickHouse) | 0 | 1 | **1** |
| Redis Instances | 4 | 2 | **6** |

### API & Code
| Метрика | Значение |
|---------|----------|
| gRPC методов | 240+ |
| REST endpoints | 310+ |
| Таблиц БД (PostgreSQL) | 220+ |
| ClickHouse таблиц | 3 |
| Миграций | 250+ |
| Go файлов | 1,200+ |
| TypeScript/React файлов | 650+ |
| Python файлов (ML) | 50+ |
| **Total LOC** | **~130,000** |

### Features
| Метрика | Значение |
|---------|----------|
| React компонентов | 150+ |
| Redux slices | 24 |
| Ролей | 32 |
| Разрешений | 85+ |
| E2E Tests | 372 |
| K8s Deployments | 14 |
| Channel Adapters | 4 (eBay, Amazon, Etsy, Shopify) |

---

## Версионирование

| Версия | Дата | Изменения |
|--------|------|-----------|
| **2.0.0** | **2025-12-25** | **WMS PRIME 100% COMPLETE** - Реализованы все 6 фаз (A-F): ML Forecasting, 3PL Gateway, Channel Adapters (eBay/Amazon/Etsy/Shopify), 6 WMS порталов, K8s deployment, E2E тесты (372/372 ✅) |
| 1.2.0 | 2025-12-23 | Добавлен Fraud Scoring Service |
| 1.1.0 | 2025-12-21 | Добавлен PVZ Service (пункты выдачи заказов) |
| 1.0.0 | 2025-12-21 | Начальная версия паспорта |

---

**Последнее обновление:** 2025-12-25 (WMS Prime 2.0.0 Release)
