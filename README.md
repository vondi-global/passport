# Паспорт Маркетплейса Vondi

> **Версия:** 1.2.0 | **Дата:** 2025-12-23 | **Статус:** ACTIVE

Исчерпывающая техническая документация платформы Vondi Marketplace.

---

## Навигация

### Архитектура
- [ARCHITECTURE.md](./ARCHITECTURE.md) — Общая архитектура системы
- [GLOSSARY.md](./GLOSSARY.md) — Глоссарий терминов

### Микросервисы
| Сервис | Документ | gRPC Port | HTTP Port |
|--------|----------|-----------|-----------|
| Backend Monolith | [services/monolith.md](./services/monolith.md) | - | 3000 |
| Auth Service | [services/auth.md](./services/auth.md) | 20053 | 28086 |
| Listings Service | [services/listings.md](./services/listings.md) | 50053 | 8084 |
| Delivery Service | [services/delivery.md](./services/delivery.md) | 50052 | 8082 |
| Warehouse/WMS | [services/warehouse.md](./services/warehouse.md) | 50055 | 8085 |
| Payment Service | [services/payment.md](./services/payment.md) | 30051 | 8087 |
| Notifications | [services/notifications.md](./services/notifications.md) | 50054 | 8088 |
| PVZ Service | [services/pvz.md](./services/pvz.md) | 50056 | 8089 |
| Fraud Scoring | [services/fraud-scoring.md](./services/fraud-scoring.md) | 50058 | 8094 |

### Frontend
| Документ | Описание |
|----------|----------|
| [frontend/svetu-web.md](./frontend/svetu-web.md) | Next.js 15 приложение |
| [frontend/components.md](./frontend/components.md) | Компоненты и хуки |
| [frontend/state-management.md](./frontend/state-management.md) | Redux + React Query |

### Базы данных
| База данных | Документ | Port |
|-------------|----------|------|
| vondi_db (Monolith) | [databases/vondi_db.md](./databases/vondi_db.md) | 5433 |
| auth_db | [databases/auth_db.md](./databases/auth_db.md) | 25432 |
| listings_db | [databases/listings_db.md](./databases/listings_db.md) | 35434 |
| delivery_db | [databases/delivery_db.md](./databases/delivery_db.md) | 35432 |
| payment_db | [databases/payment_db.md](./databases/payment_db.md) | 35433 |
| wms_db | [databases/wms_db.md](./databases/wms_db.md) | 35435 |
| pvz_db | [databases/pvz_db.md](./databases/pvz_db.md) | 35438 |
| fraud_scoring (статистика) | [databases/fraud_scoring.md](./databases/fraud_scoring.md) | N/A (stateless) |
| Redis instances | [databases/redis.md](./databases/redis.md) | various |

### Интеграции
| Интеграция | Документ |
|------------|----------|
| gRPC контракты | [integrations/grpc-contracts.md](./integrations/grpc-contracts.md) |
| REST API | [integrations/rest-api.md](./integrations/rest-api.md) |
| Events | [integrations/events.md](./integrations/events.md) |
| Post Express | [integrations/external/post-express.md](./integrations/external/post-express.md) |
| MinIO/S3 | [integrations/external/minio-s3.md](./integrations/external/minio-s3.md) |
| OpenSearch | [search/opensearch.md](./search/opensearch.md) |

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

```
Frontend:      3001 (Next.js)
Backend:       3000 (Go/Fiber)
Auth:          28086/20053 (HTTP/gRPC)
Listings:      8084/50053
Delivery:      8082/50052
WMS:           8085/50055
Payment:       8087/30051
Notifications: 8088/50054
PVZ:           8089/50056
```

### Базы данных

```
Monolith DB:   localhost:5433  (vondi_db)
Auth DB:       localhost:25432 (auth_db)
Listings DB:   localhost:35434 (listings_dev_db)
Delivery DB:   localhost:35432 (delivery_db)
Payment DB:    localhost:35433 (payment_db)
WMS DB:        localhost:35435 (wms_db)
PVZ DB:        localhost:35438 (pvz_db)
```

### Redis

```
Auth Redis:     localhost:26379
Listings Redis: localhost:36379
Delivery Redis: localhost:36380
WMS Redis:      localhost:36381
PVZ Redis:      localhost:36384
```

---

## Статистика системы

| Метрика | Значение |
|---------|----------|
| Микросервисов | 8 |
| gRPC методов | 200+ |
| REST endpoints | 230+ |
| Таблиц БД | 210+ |
| Миграций | 225+ |
| Go файлов | 850+ |
| React компонентов | 85+ |
| Redux slices | 18 |
| Ролей | 28 |
| Разрешений | 67 |

---

## Версионирование

| Версия | Дата | Изменения |
|--------|------|-----------|
| 1.1.0 | 2025-12-21 | Добавлен PVZ Service (пункты выдачи заказов) |
| 1.0.0 | 2025-12-21 | Начальная версия паспорта |

---

**Последнее обновление:** 2025-12-21
