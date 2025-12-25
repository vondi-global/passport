# Backend Monolith Service Passport

**Статус:** Production | **Версия:** 0.2.2 | **Последнее обновление:** 2025-12-21

---

## 1. Обзор

### Назначение
Backend-монолит платформы Vondi - основной HTTP API сервер, предоставляющий REST endpoints для:
- Управления пользователями и аутентификацией (через Auth Service)
- Marketplace (C2C объявления, B2C товары) - через Listings Microservice
- Заказов и доставки - через Delivery Microservice
- Платежей - через Payment Microservice
- Витрин продавцов (Storefronts)
- Админ-панели
- Чата и уведомлений
- Поиска (OpenSearch)

**Архитектура:** Монолитное приложение с переходом на микросервисы (гибридная модель)

### Стек технологий
| Компонент | Технология | Версия |
|-----------|------------|--------|
| Язык | Go | 1.25.0 |
| Web Framework | Fiber v2 | 2.52.9 |
| База данных | PostgreSQL | 14+ |
| Кэш | Redis | 6.x |
| Поиск | OpenSearch | 2.x |
| Хранилище | MinIO (S3) | Latest |
| Логирование | zerolog | 1.34.0 |
| Валидация | go-playground/validator | v10.28.0 |
| HTTP Клиенты | gRPC (микросервисы) | - |

### Порты и доступность
| Тип | Порт | Описание |
|-----|------|----------|
| HTTP API | 3000 | Основной REST API (`/api/v1/*`) |
| PostgreSQL | 5433 | База данных `vondi_db` (НЕ стандартный 5432!) |
| Redis | 6379 | Кэш категорий, атрибутов, сессий |
| OpenSearch | 9200 | Полнотекстовый поиск |
| MinIO | 9000 | S3-совместимое хранилище файлов |

**Production URL:** `https://devapi.vondi.rs` (backend), `https://dev.vondi.rs` (frontend)

---

## 2. Конфигурация

### Переменные окружения (критичные)

#### База данных
```bash
DATABASE_URL=postgres://postgres:PASSWORD@localhost:5433/vondi_db?sslmode=disable
MIGRATIONS_ON_API=off                    # Миграции вручную через ./migrator
MIGRATIONS_PATH=migrations
MIGRATIONS_FIXTURES_PATH=fixtures        # Тестовые данные
```

#### Auth Service (REQUIRED)
```bash
USE_AUTH_SERVICE=true                                      # Обязательно!
AUTH_SERVICE_URL=http://localhost:28080                    # HTTP endpoint
AUTH_GRPC_URL=localhost:20053                              # gRPC endpoint
AUTH_SERVICE_PUBLIC_KEY_PATH=/path/to/auth_public.pem     # JWT валидация
JWT_EXPIRATION_HOURS=24
JWT_ACCESS_TOKEN_DURATION=60m
```

#### Listings Microservice (REQUIRED)
```bash
LISTINGS_GRPC_URL=localhost:50053           # Основной каталог (C2C + B2C)
LISTINGS_HTTP_URL=http://localhost:8086     # HTTP fallback
LISTINGS_GRPC_TIMEOUT=10s
```

#### Delivery Microservice (REQUIRED)
```bash
DELIVERY_GRPC_URL=localhost:50052           # Доставка (Post Express, BEX)
```

#### Payment Microservice (REQUIRED)
```bash
PAYMENT_GRPC_URL=localhost:30051            # Платежи (Stripe, AllSecure)
PAYMENT_GRPC_TIMEOUT=30s
```

#### Redis
```bash
REDIS_URL=localhost:6379
REDIS_PASSWORD=YOUR_REDIS_PASSWORD
REDIS_DB=0
REDIS_POOL_SIZE=10
REDIS_CATEGORIES_TTL=6h                     # Кэш категорий
REDIS_ATTRIBUTES_TTL=4h                     # Кэш атрибутов
```

#### MinIO/S3
```bash
FILE_STORAGE_PROVIDER=minio
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=YOUR_SECURE_SECRET
MINIO_USE_SSL=false
MINIO_BUCKET_NAME=listings                  # Основной bucket
MINIO_CHAT_BUCKET=chat-files
MINIO_STOREFRONT_BUCKET=storefront-products
MINIO_REVIEW_PHOTOS_BUCKET=review-photos
FILE_STORAGE_PUBLIC_URL=http://localhost:3000
MINIO_PUBLIC_URL=http://localhost:9000
```

#### OpenSearch
```bash
OPENSEARCH_URL=http://localhost:9200
OPENSEARCH_USERNAME=admin
OPENSEARCH_PASSWORD=YOUR_PASSWORD
OPENSEARCH_MARKETPLACE_INDEX=marketplace_listings
```

#### Feature Flags
```bash
USE_UNIFIED_ATTRIBUTES=true                 # Unified Attributes система
UNIFIED_ATTRIBUTES_FALLBACK=true            # Fallback на старую схему
DUAL_WRITE_ATTRIBUTES=true                  # Дублировать запись
```

#### Дополнительно
```bash
PORT=3000
ENV=development                             # development | production
SERVER_HOST=http://localhost:3000
FRONTEND_URL=http://localhost:3001
LOG_LEVEL=debug                             # debug | info | warn | error
```

**Полный список:** См. `/p/github.com/vondi-global/vondi/backend/.env.example` (208 строк)

---

## 3. Структура проекта

```
backend/
├── cmd/
│   └── api/main.go              # Точка входа приложения
├── internal/
│   ├── app/                     # Инициализация приложения
│   ├── cache/                   # Redis кэширование
│   ├── client/                  # gRPC клиенты (WMS, Auth)
│   ├── clients/                 # HTTP клиенты (Listings, Payment, Attributes)
│   ├── config/                  # Конфигурация (config.go)
│   ├── domain/                  # Доменные модели (55 файлов)
│   │   ├── adapters/            # Адаптеры к внешним системам
│   │   ├── behavior/            # Поведенческие модели
│   │   ├── logistics/           # Логистика (доставка)
│   │   ├── models/              # Основные сущности (см. раздел 4)
│   │   └── search/              # Поисковые модели
│   ├── grpc/                    # gRPC клиенты/адаптеры
│   │   ├── categories/          # Категории (к Listings)
│   │   ├── orders/              # Заказы (к Listings)
│   │   ├── search/              # Поиск (к Search Microservice)
│   │   └── storefronts/         # Витрины (к Listings)
│   ├── interfaces/              # Интерфейсы репозиториев/сервисов
│   ├── logger/                  # Глобальный логгер (zerolog)
│   ├── metrics/                 # Prometheus метрики
│   ├── middleware/              # HTTP middleware (Auth, CORS, Logger)
│   ├── monitoring/              # Health checks, мониторинг
│   ├── proj/                    # Бизнес-модули (36 директорий)
│   │   ├── addresses/           # Адреса пользователей
│   │   ├── admin/               # Админ-панель (логистика, тесты)
│   │   ├── ai/                  # AI интеграции (переводы, рекомендации)
│   │   ├── analytics/           # Аналитика, метрики
│   │   ├── attributes/          # Атрибуты товаров
│   │   ├── balance/             # Баланс пользователей
│   │   ├── behavior_tracking/   # Отслеживание поведения
│   │   ├── chat/                # Чат между пользователями
│   │   ├── contacts/            # Контакты
│   │   ├── credit/              # Кредитная система
│   │   ├── delivery/            # Доставка (facade к Delivery Service)
│   │   ├── franchise/           # Франшиза (PVZ)
│   │   ├── geocode/             # Геокодирование (Mapbox, Nominatim)
│   │   ├── gis/                 # ГИС функции
│   │   ├── health/              # Health checks
│   │   ├── marketplace/         # Marketplace (facade к Listings Service)
│   │   ├── notifications/       # Уведомления
│   │   ├── orders/              # Заказы (facade к Listings Service)
│   │   ├── payments/            # Платежи (facade к Payment Service)
│   │   ├── public/              # Публичные endpoints
│   │   ├── recommendations/     # Рекомендации товаров
│   │   ├── reviews/             # Отзывы
│   │   ├── search_admin/        # Админ поиска
│   │   ├── search_optimization/ # Оптимизация поиска
│   │   ├── searchlogs/          # Логи поисковых запросов
│   │   ├── storefront/          # Витрины продавцов
│   │   ├── subscriptions/       # Подписки
│   │   ├── tracking/            # Трекинг посылок
│   │   ├── translation_admin/   # Админ переводов
│   │   ├── users/               # Пользователи (facade к Auth Service)
│   │   ├── viber/               # Viber бот
│   │   ├── vin/                 # VIN декодирование (автомобили)
│   │   └── warehouse_owner/     # Владельцы складов
│   ├── server/                  # HTTP сервер (Fiber, роуты)
│   ├── service/                 # Бизнес-логика
│   ├── services/                # Утилиты, хелперы
│   ├── storage/                 # Репозитории (PostgreSQL, OpenSearch, MinIO)
│   │   ├── postgres/            # 52 файла репозиториев
│   │   ├── opensearch/          # Поисковые индексы
│   │   └── minio/               # S3 хранилище
│   ├── types/                   # Общие типы
│   └── version/                 # Версионирование
├── migrations/                  # SQL миграции (100+ файлов)
│   ├── 000001_*.up.sql
│   ├── 000001_*.down.sql
│   └── ...
├── fixtures/                    # Тестовые данные (SQL)
├── docs/                        # Swagger документация
│   └── swagger.json
├── pkg/                         # Публичные утилиты
│   └── utils/
├── go.mod                       # Go зависимости
├── go.sum
├── Makefile                     # Команды сборки
├── Dockerfile                   # Docker образ
├── docker-compose.yml           # Локальная инфраструктура
├── .env.example                 # Шаблон переменных окружения
├── .golangci.yml                # Линтер конфиг
└── migrator                     # Утилита миграций (бинарник)
```

**Статистика:**
- **Go файлов:** 500+ (включая тесты)
- **Миграций:** 100+ (схема + данные)
- **Модулей (proj/):** 36 бизнес-доменов
- **Репозиториев (storage/):** 52 файла PostgreSQL
- **Доменных моделей:** 55 файлов

---

## 4. Доменная модель

### Основные сущности (internal/domain/models/)

#### User (пользователь)
- **Управление:** Auth Microservice
- **Монолит:** Хранит локальный кэш, профили, настройки
- **Поля:** ID, Email, Phone, Name, Avatar, Balance, Role, Status
- **Связи:** Orders, Listings, Reviews, ChatMessages, Addresses

#### Listing (объявление C2C)
- **Управление:** Listings Microservice (gRPC: 50053)
- **Монолит:** Facade API (`/api/v1/marketplace/*`)
- **Поля:** ID, UserID, CategoryID, Title, Description, Price, Images, Attributes, Status
- **Связи:** Category, User (seller), Images, Attributes, Reviews, Favorites

#### StorefrontProduct (товар B2C)
- **Управление:** Listings Microservice
- **Монолит:** Facade API (`/api/v1/storefronts/:id/products/*`)
- **Поля:** ID, StorefrontID, Title, Price, Variants, Inventory
- **Связи:** Storefront, Variants, Images, Attributes

#### Order (заказ)
- **Управление:** Listings Microservice
- **Монолит:** Facade API (`/api/v1/orders/*`)
- **Поля:** ID, BuyerID, SellerID, Items, TotalAmount, Status, PaymentMethod, DeliveryAddress
- **Связи:** Buyer, Seller, OrderItems, Delivery, Payment

#### Category (категория)
- **Управление:** Listings Microservice
- **Кэш:** Redis (TTL: 6h)
- **Поля:** ID, ParentID, NameSr, NameEn, NameRu, Slug, Icon, RequiredAttributes
- **Связи:** Parent, Children, Attributes, AttributeGroups

#### Attribute (атрибут товара)
- **Управление:** Listings Microservice
- **Кэш:** Redis (TTL: 4h)
- **Поля:** ID, GroupID, Name, Type (text/number/select), Required, Searchable
- **Связи:** AttributeGroup, AttributeValues

#### Storefront (витрина продавца)
- **Управление:** Listings Microservice
- **Монолит:** Дублирует часть данных (Products в отдельной БД)
- **Поля:** ID, OwnerID, Name, Description, Logo, BannerImage, Settings
- **Связи:** Owner, Products, Orders, Reviews

#### Review (отзыв)
- **Управление:** Монолит (PostgreSQL `vondi_db`)
- **Поля:** ID, ListingID/ProductID, UserID, Rating, Comment, Photos, Status
- **Связи:** Listing OR Product, User, Photos

#### ChatMessage (сообщение чата)
- **Управление:** Монолит (PostgreSQL `vondi_db`)
- **Поля:** ID, ConversationID, SenderID, ReceiverID, Text, Attachments, ReadAt
- **Связи:** Conversation, Sender, Receiver, Attachments

#### Delivery (доставка)
- **Управление:** Delivery Microservice (gRPC: 50052)
- **Монолит:** Facade API (`/api/v1/delivery/*`)
- **Поля:** OrderID, Provider (post-express/bex), TrackingNumber, Status, SenderAddress, ReceiverAddress
- **Связи:** Order, Provider

#### Payment (платеж)
- **Управление:** Payment Microservice (gRPC: 30051)
- **Монолит:** Facade API (`/api/v1/payments/*`)
- **Поля:** OrderID, Provider (stripe/allsecure), Amount, Currency, Status, TransactionID
- **Связи:** Order

---

## 5. REST API Endpoints

### Группировка по доменам

#### Auth (`/api/v1/auth/*`)
**Управление:** Auth Service (библиотека `github.com/vondi-global/auth/pkg/http/fiber`)

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/auth/register` | Регистрация (email/phone) |
| POST | `/auth/login` | Вход (email+password) |
| POST | `/auth/refresh` | Обновление JWT токена |
| POST | `/auth/logout` | Выход |
| GET | `/auth/me` | Профиль текущего пользователя |
| POST | `/auth/google` | OAuth Google |
| POST | `/auth/google/callback` | Google OAuth callback |
| POST | `/auth/verify-phone` | Подтверждение телефона (Firebase) |

**Middleware:** `JWTParser` → `RequireAuth()` → `RequireAuth("admin")`

---

#### Users (`/api/v1/users/*`)
**Защита:** JWT токен обязателен

| Метод | Endpoint | Описание | Роль |
|-------|----------|----------|------|
| GET | `/users/profile` | Профиль пользователя | user |
| PUT | `/users/profile` | Обновление профиля | user |
| GET | `/users/:id/profile` | Профиль другого пользователя | user |
| GET | `/users/privacy-settings` | Настройки приватности | user |
| PUT | `/users/privacy-settings` | Обновление приватности | user |
| POST | `/users/phone/verify` | Подтверждение телефона | user |

**Admin endpoints:**
| Метод | Endpoint | Описание |
|-------|----------|----------|
| GET | `/admin/users/` | Список всех пользователей |
| GET | `/admin/users/:id` | Детали пользователя |
| PUT | `/admin/users/:id` | Обновление пользователя |
| PUT | `/admin/users/:id/status` | Изменение статуса |
| PUT | `/admin/users/:id/role` | Изменение роли |
| DELETE | `/admin/users/:id` | Удаление пользователя |
| GET | `/admin/admins/` | Список администраторов |
| POST | `/admin/admins/` | Добавление админа |
| DELETE | `/admin/admins/:email` | Удаление админа |

---

#### Marketplace (`/api/v1/marketplace/*`)
**Facade к:** Listings Microservice (gRPC: 50053)

| Метод | Endpoint | Описание | Auth |
|-------|----------|----------|------|
| GET | `/marketplace/listings` | Список объявлений | нет |
| GET | `/marketplace/listings/:id` | Детали объявления | нет |
| POST | `/marketplace/listings` | Создание объявления | user |
| PUT | `/marketplace/listings/:id` | Обновление объявления | user |
| DELETE | `/marketplace/listings/:id` | Удаление объявления | user |
| POST | `/marketplace/listings/:id/images` | Загрузка изображений | user |
| DELETE | `/marketplace/listings/:id/images/:imageId` | Удаление изображения | user |
| POST | `/marketplace/listings/:id/favorite` | Добавить в избранное | user |
| DELETE | `/marketplace/listings/:id/favorite` | Убрать из избранного | user |

**Admin:**
| GET | `/admin/marketplace/listings` | Все объявления с фильтрами |
| PUT | `/admin/marketplace/listings/:id/status` | Модерация (approve/reject) |

---

#### Orders (`/api/v1/orders/*`)
**Facade к:** Listings Microservice

| Метод | Endpoint | Описание | Auth |
|-------|----------|----------|------|
| GET | `/orders/` | Мои заказы (buyer + seller) | user |
| GET | `/orders/:id` | Детали заказа | user |
| POST | `/orders/` | Создание заказа | user |
| PUT | `/orders/:id/status` | Обновление статуса | seller |
| POST | `/orders/:id/confirm-delivery` | Подтвердить получение | buyer |
| POST | `/orders/:id/dispute` | Открыть спор | user |

**Admin:**
| GET | `/admin/orders/` | Все заказы платформы |
| PUT | `/admin/orders/:id/resolve-dispute` | Разрешить спор |

---

#### Storefronts (`/api/v1/storefronts/*`)
**Facade к:** Listings Microservice + Монолит (Products в локальной БД)

| Метод | Endpoint | Описание | Auth |
|-------|----------|----------|------|
| GET | `/storefronts/` | Список витрин | нет |
| GET | `/storefronts/:id` | Детали витрины | нет |
| POST | `/storefronts/` | Создание витрины | user |
| PUT | `/storefronts/:id` | Обновление витрины | owner |
| DELETE | `/storefronts/:id` | Удаление витрины | owner |
| GET | `/storefronts/:id/products` | Товары витрины | нет |
| POST | `/storefronts/:id/products` | Добавить товар | owner |
| PUT | `/storefronts/:id/products/:productId` | Обновить товар | owner |
| DELETE | `/storefronts/:id/products/:productId` | Удалить товар | owner |
| POST | `/storefronts/:id/products/:productId/images` | Загрузить изображения | owner |

---

#### Delivery (`/api/v1/delivery/*`)
**Facade к:** Delivery Microservice (gRPC: 50052)

| Метод | Endpoint | Описание | Auth |
|-------|----------|----------|------|
| POST | `/delivery/calculate` | Расчет стоимости | user |
| POST | `/delivery/create-shipment` | Создание заказа доставки | user |
| GET | `/delivery/track/:trackingNumber` | Трекинг посылки | user |
| GET | `/delivery/providers` | Список провайдеров | нет |
| GET | `/delivery/post-express/locations` | Пункты Post Express | нет |

**Admin:**
| GET | `/admin/delivery/shipments` | Все посылки |
| PUT | `/admin/delivery/shipments/:id/status` | Обновление статуса |

---

#### Payments (`/api/v1/payments/*`)
**Facade к:** Payment Microservice (gRPC: 30051)

| Метод | Endpoint | Описание | Auth |
|-------|----------|----------|------|
| POST | `/payments/create-intent` | Создание платежа | user |
| POST | `/payments/confirm` | Подтверждение платежа | user |
| GET | `/payments/:id` | Статус платежа | user |
| POST | `/payments/webhook` | Webhook от Stripe/AllSecure | нет |

---

#### Categories (`/api/v1/categories/*`)
**Facade к:** Listings Microservice + Redis cache

| Метод | Endpoint | Описание | Auth |
|-------|----------|----------|------|
| GET | `/categories/` | Дерево категорий | нет |
| GET | `/categories/:id` | Детали категории | нет |
| GET | `/categories/:id/attributes` | Атрибуты категории | нет |

**Admin:**
| POST | `/admin/categories/` | Создание категории |
| PUT | `/admin/categories/:id` | Обновление категории |
| DELETE | `/admin/categories/:id` | Удаление категории |

---

#### Search (`/api/v1/search/*`)
**Реализация:** OpenSearch + PostgreSQL fallback

| Метод | Endpoint | Описание |
|-------|----------|----------|
| GET | `/search/` | Поиск товаров/объявлений (q, category, filters) |
| GET | `/search/suggestions` | Автодополнение |
| GET | `/search/trending` | Популярные запросы |

---

#### Chat (`/api/v1/chat/*`)
**Реализация:** Монолит (PostgreSQL + WebSocket)

| Метод | Endpoint | Описание | Auth |
|-------|----------|----------|------|
| GET | `/chat/conversations` | Список диалогов | user |
| GET | `/chat/conversations/:id/messages` | Сообщения диалога | user |
| POST | `/chat/conversations/:id/messages` | Отправка сообщения | user |
| PUT | `/chat/messages/:id/read` | Пометить прочитанным | user |
| WebSocket | `/ws/chat` | WebSocket для real-time чата | user |

---

#### Reviews (`/api/v1/reviews/*`)
**Реализация:** Монолит (PostgreSQL)

| Метод | Endpoint | Описание | Auth |
|-------|----------|----------|------|
| GET | `/reviews/listing/:listingId` | Отзывы на объявление | нет |
| GET | `/reviews/product/:productId` | Отзывы на товар | нет |
| POST | `/reviews/` | Создание отзыва | user |
| PUT | `/reviews/:id` | Обновление отзыва | user |
| DELETE | `/reviews/:id` | Удаление отзыва | user |

---

#### Health & Monitoring

| Метод | Endpoint | Описание |
|-------|----------|----------|
| GET | `/` | Root (версия API) |
| GET | `/health` | Health check (БД, Redis, микросервисы) |
| GET | `/metrics` | Prometheus метрики |
| GET | `/swagger/*` | Swagger UI документация |

---

## 6. gRPC клиенты (к микросервисам)

### Внутренние микросервисы Vondi

| Сервис | gRPC URL | HTTP URL | Назначение |
|--------|----------|----------|------------|
| **Auth Service** | `localhost:20053` | `http://localhost:28080` | Аутентификация, пользователи, роли |
| **Listings Service** | `localhost:50053` | `http://localhost:8086` | Каталог (C2C + B2C), заказы, категории, избранное, чат |
| **Delivery Service** | `localhost:50052` | - | Доставка (Post Express, BEX), трекинг |
| **Payment Service** | `localhost:30051` | - | Платежи (Stripe, AllSecure, Mock) |

### Клиенты в коде

#### Listings Client
```go
// internal/clients/listings/client.go
import listingspb "github.com/vondi-global/listings/api/proto/listings/v1"

type Client struct {
    grpcClient listingspb.ListingsServiceClient
}

// Основные методы:
// - GetListing(ctx, id) - получить объявление
// - CreateListing(ctx, req) - создать объявление
// - UpdateListing(ctx, id, req) - обновить объявление
// - DeleteListing(ctx, id) - удалить объявление
// - SearchListings(ctx, filters) - поиск
```

#### Auth Service Client
```go
// Используется библиотека: github.com/vondi-global/auth/pkg/service
import authService "github.com/vondi-global/auth/pkg/service"

// Основные сервисы:
// 1. AuthService - валидация токенов
authSvc := authService.NewAuthServiceWithLocalValidation(client, logger)

// 2. UserService - CRUD пользователей
userSvc := authService.NewUserService(client, logger)
userSvc.GetUserByID(ctx, userID)
userSvc.UpdateUser(ctx, userID, data)

// 3. OAuthService - Google OAuth
oauthSvc := authService.NewOAuthService(client)
oauthSvc.ExchangeGoogleCode(ctx, code)
```

---

## 7. Зависимости

### Критичные библиотеки

| Библиотека | Версия | Назначение |
|------------|--------|------------|
| `github.com/gofiber/fiber/v2` | 2.52.9 | HTTP фреймворк (роутинг, middleware) |
| `github.com/jackc/pgx/v5` | 5.7.6 | PostgreSQL драйвер |
| `github.com/jmoiron/sqlx` | 1.4.0 | SQL расширения (StructScan) |
| `github.com/redis/go-redis/v9` | 9.17.2 | Redis клиент |
| `github.com/minio/minio-go/v7` | 7.0.95 | S3 клиент (MinIO) |
| `github.com/opensearch-project/opensearch-go/v2` | 2.3.0 | OpenSearch клиент |
| `github.com/golang-jwt/jwt/v5` | 5.2.3 | JWT токены |
| `github.com/google/uuid` | 1.6.0 | UUID генерация |
| `github.com/rs/zerolog` | 1.34.0 | Структурированное логирование |
| `github.com/go-playground/validator/v10` | 10.28.0 | Валидация структур |
| `github.com/golang-migrate/migrate/v4` | 4.19.0 | SQL миграции |

### Vondi внутренние библиотеки

| Библиотека | Назначение |
|------------|------------|
| `github.com/vondi-global/auth/pkg/http/fiber` | Auth Service интеграция (middleware, routes) |
| `github.com/vondi-global/auth/pkg/service` | Auth Service клиент (UserService, OAuthService) |
| `github.com/vondi-global/listings/api/proto/listings/v1` | Listings gRPC protobuf |
| `github.com/vondi-global/delivery/api/proto/delivery/v1` | Delivery gRPC protobuf |
| `github.com/vondi-global/payment/api/proto/payment/v1` | Payment gRPC protobuf |

---

## 8. Связанная документация

### Основные файлы
- **README:** `/p/github.com/vondi-global/vondi/backend/README.md`
- **CLAUDE.md:** `/p/github.com/vondi-global/CLAUDE.md` (инструкции для AI)
- **Swagger:** `/p/github.com/vondi-global/vondi/backend/docs/swagger.json`

### Руководства
- **Auth Integration:** `/p/github.com/vondi-global/auth/docs/MARKETPLACE_INTEGRATION_SPEC.md`
- **Database Guidelines:** `/p/github.com/vondi-global/vondi/docs/CLAUDE_DATABASE_GUIDELINES.md`
- **Troubleshooting:** `/p/github.com/vondi-global/vondi/docs/CLAUDE_TROUBLESHOOTING.md`

### Паспорта других сервисов
- **Auth Service:** `/p/github.com/vondi-global/.passport/services/auth.md`
- **Listings Service:** `/p/github.com/vondi-global/.passport/services/listings.md`
- **Delivery Service:** `/p/github.com/vondi-global/.passport/services/delivery.md`

### Системный паспорт
- **SYSTEM_PASSPORT.md:** `/p/github.com/vondi-global/SYSTEM_PASSPORT.md` (полная инфраструктура)

---

**Контакты:**
- **GitHub:** https://github.com/vondi-global/vondi
- **Документация:** https://docs.vondi.rs (в разработке)
- **Production API:** https://devapi.vondi.rs

**Автор паспорта:** Claude Code Agent
**Дата создания:** 2025-12-21
**Статус:** Living Document (обновляется при изменениях архитектуры)
