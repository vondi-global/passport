# Паспорт Маркетплейса Vondi

> **Версия:** 1.0.0 | **Дата:** 2025-12-21 | **Статус:** ACTIVE

Исчерпывающая техническая документация платформы Vondi Marketplace.

---

## Навигация

### Архитектура
- [ARCHITECTURE.md](/p/github.com/vondi-global/passport/ARCHITECTURE.md) — Общая архитектура системы
- [GLOSSARY.md](/p/github.com/vondi-global/passport/GLOSSARY.md) — Глоссарий терминов

### Микросервисы
| Сервис | Документ | gRPC Port | HTTP Port |
|--------|----------|-----------|-----------|
| Backend Monolith | [services/monolith.md](/p/github.com/vondi-global/passport/services/monolith.md) | - | 3000 |
| Auth Service | [services/auth.md](/p/github.com/vondi-global/passport/services/auth.md) | 20053 | 28086 |
| Listings Service | [services/listings.md](/p/github.com/vondi-global/passport/services/listings.md) | 50053 | 8084 |
| Delivery Service | [services/delivery.md](/p/github.com/vondi-global/passport/services/delivery.md) | 50052 | 8082 |
| Warehouse/WMS | [services/warehouse.md](/p/github.com/vondi-global/passport/services/warehouse.md) | 50055 | 8085 |
| Payment Service | [services/payment.md](/p/github.com/vondi-global/passport/services/payment.md) | 30051 | 8087 |
| Notifications | [services/notifications.md](/p/github.com/vondi-global/passport/services/notifications.md) | 50054 | 8088 |

### Frontend
| Документ | Описание |
|----------|----------|
| [frontend/svetu-web.md](/p/github.com/vondi-global/passport/frontend/svetu-web.md) | Next.js 15 приложение |
| [frontend/components.md](/p/github.com/vondi-global/passport/frontend/components.md) | Компоненты и хуки |
| [frontend/state-management.md](/p/github.com/vondi-global/passport/frontend/state-management.md) | Redux + React Query |

### Базы данных
| База данных | Документ | Port |
|-------------|----------|------|
| vondi_db (Monolith) | [databases/vondi_db.md](/p/github.com/vondi-global/passport/databases/vondi_db.md) | 5433 |
| auth_db | [databases/auth_db.md](/p/github.com/vondi-global/passport/databases/auth_db.md) | 25432 |
| listings_db | [databases/listings_db.md](/p/github.com/vondi-global/passport/databases/listings_db.md) | 35434 |
| delivery_db | [databases/delivery_db.md](/p/github.com/vondi-global/passport/databases/delivery_db.md) | 35432 |
| wms_db | [databases/wms_db.md](/p/github.com/vondi-global/passport/databases/wms_db.md) | 35435 |
| Redis instances | [databases/redis.md](/p/github.com/vondi-global/passport/databases/redis.md) | various |

### Интеграции
| Интеграция | Документ |
|------------|----------|
| gRPC контракты | [integrations/grpc-contracts.md](/p/github.com/vondi-global/passport/integrations/grpc-contracts.md) |
| REST API | [integrations/rest-api.md](/p/github.com/vondi-global/passport/integrations/rest-api.md) |
| Events | [integrations/events.md](/p/github.com/vondi-global/passport/integrations/events.md) |
| Post Express | [integrations/external/post-express.md](/p/github.com/vondi-global/passport/integrations/external/post-express.md) |
| MinIO/S3 | [integrations/external/minio-s3.md](/p/github.com/vondi-global/passport/integrations/external/minio-s3.md) |
| OpenSearch | [search/opensearch.md](/p/github.com/vondi-global/passport/search/opensearch.md) |

### Бизнес-домены
| Домен | Документ |
|-------|----------|
| Пользователи | [domains/users.md](/p/github.com/vondi-global/passport/domains/users.md) |
| Объявления | [domains/listings.md](/p/github.com/vondi-global/passport/domains/listings.md) |
| Заказы | [domains/orders.md](/p/github.com/vondi-global/passport/domains/orders.md) |
| Платежи | [domains/payments.md](/p/github.com/vondi-global/passport/domains/payments.md) |
| Доставка | [domains/delivery.md](/p/github.com/vondi-global/passport/domains/delivery.md) |
| Склад | [domains/warehouse.md](/p/github.com/vondi-global/passport/domains/warehouse.md) |
| Категории | [domains/categories.md](/p/github.com/vondi-global/passport/domains/categories.md) |
| Магазины | [domains/storefronts.md](/p/github.com/vondi-global/passport/domains/storefronts.md) |

### Бизнес-процессы
| Процесс | Документ |
|---------|----------|
| Аутентификация | [flows/authentication.md](/p/github.com/vondi-global/passport/flows/authentication.md) |
| Создание листинга | [flows/listing-creation.md](/p/github.com/vondi-global/passport/flows/listing-creation.md) |
| Жизненный цикл заказа | [flows/order-lifecycle.md](/p/github.com/vondi-global/passport/flows/order-lifecycle.md) |
| Платёжный поток | [flows/payment-flow.md](/p/github.com/vondi-global/passport/flows/payment-flow.md) |
| Fulfillment | [flows/fulfillment.md](/p/github.com/vondi-global/passport/flows/fulfillment.md) |
| Возвраты | [flows/returns.md](/p/github.com/vondi-global/passport/flows/returns.md) |
| Поиск | [flows/search.md](/p/github.com/vondi-global/passport/flows/search.md) |
| Загрузка файлов | [flows/file-upload.md](/p/github.com/vondi-global/passport/flows/file-upload.md) |

### Инфраструктура
| Документ | Описание |
|----------|----------|
| [infrastructure/docker.md](/p/github.com/vondi-global/passport/infrastructure/docker.md) | Docker конфигурация |
| [infrastructure/network.md](/p/github.com/vondi-global/passport/infrastructure/network.md) | Сетевая архитектура |
| [infrastructure/storage.md](/p/github.com/vondi-global/passport/infrastructure/storage.md) | Хранилища данных |
| [infrastructure/ci-cd.md](/p/github.com/vondi-global/passport/infrastructure/ci-cd.md) | CI/CD pipelines |

### Безопасность
| Документ | Описание |
|----------|----------|
| [security/authentication.md](/p/github.com/vondi-global/passport/security/authentication.md) | JWT, OAuth, Firebase |
| [security/authorization.md](/p/github.com/vondi-global/passport/security/authorization.md) | RBAC, permissions |
| [security/data-protection.md](/p/github.com/vondi-global/passport/security/data-protection.md) | Шифрование, GDPR |

---

## Быстрый старт

### Порты системы

```
Frontend:      3001 (Next.js)
Backend:       3000 (Go/Fiber)
Auth:          28086/20053 (HTTP/gRPC)
Listings:      8084/50053
Delivery:      8082/50052
WMS:           8085/50055
Payment:       8087/30051
Notifications: 8088/50054
```

### Базы данных

```
Monolith DB:   localhost:5433  (vondi_db)
Auth DB:       localhost:25432 (auth_db)
Listings DB:   localhost:35434 (listings_dev_db)
Delivery DB:   localhost:35432 (delivery_db)
WMS DB:        localhost:35435 (wms_db)
```

### Redis

```
Auth Redis:     localhost:26379
Listings Redis: localhost:36379
Delivery Redis: localhost:36380
WMS Redis:      localhost:36381
```

---

## Статистика системы

| Метрика | Значение |
|---------|----------|
| Микросервисов | 7 |
| gRPC методов | 170+ |
| REST endpoints | 200+ |
| Таблиц БД | 200+ |
| Миграций | 220+ |
| Go файлов | 800+ |
| React компонентов | 85+ |
| Redux slices | 18 |
| Ролей | 28 |
| Разрешений | 67 |

---

## Версионирование

| Версия | Дата | Изменения |
|--------|------|-----------|
| 1.0.0 | 2025-12-21 | Начальная версия паспорта |

---

**Последнее обновление:** 2025-12-21
