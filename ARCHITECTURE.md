# Архитектура системы Vondi Marketplace

> **Версия:** 1.0.0 | **Дата:** 2025-12-21

---

## 1. Обзор

Vondi Marketplace — микросервисная платформа для C2C/B2C торговли с фокусом на рынок Сербии.

### Ключевые характеристики

| Характеристика | Значение |
|----------------|----------|
| Архитектура | Microservices + Modular Monolith |
| Языки | Go 1.23, TypeScript 5, SQL |
| Протоколы | gRPC (proto3), REST, WebSocket |
| БД | PostgreSQL 15+, Redis 7+, OpenSearch 2.x |
| Storage | MinIO S3-compatible |
| Frontend | Next.js 15, React 19 |

---

## 2. Диаграмма компонентов

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              КЛИЕНТЫ                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Browser    │  │  Mobile App  │  │  Telegram    │  │   External   │    │
│  │   (React)    │  │   (Future)   │  │   Mini App   │  │    APIs      │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
└─────────┼──────────────────┼──────────────────┼──────────────────┼──────────┘
          │                  │                  │                  │
          ▼                  ▼                  ▼                  ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                              NGINX (Reverse Proxy)                          │
│                     dev.vondi.rs / svetu.rs / vondi.rs                      │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
          ┌──────────────────────────┼──────────────────────────┐
          │                          │                          │
          ▼                          ▼                          ▼
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│      FRONTEND       │  │   BACKEND MONOLITH  │  │     MICROSERVICES   │
│    Next.js 15.3.2   │  │    Go 1.23/Fiber    │  │      Go/gRPC        │
│      Port: 3001     │  │     Port: 3000      │  │   Ports: various    │
│                     │  │                     │  │                     │
│  - App Router       │  │  - REST API         │  │  - Auth :20053      │
│  - BFF Proxy        │  │  - Business Logic   │  │  - Listings :50053  │
│  - SSR/SSG          │  │  - Orchestration    │  │  - Delivery :50052  │
│  - Redux Toolkit    │  │  - DB Access        │  │  - WMS :50055       │
│  - React Query      │  │  - gRPC Clients     │  │  - Payment :30051   │
│                     │  │                     │  │  - Notify :50054    │
│                     │  │                     │  │  - PVZ :50056       │
└─────────┬───────────┘  └─────────┬───────────┘  └─────────┬───────────┘
          │                        │                        │
          │   /api/v2 (BFF)        │   gRPC                 │
          └────────────────────────┤                        │
                                   │                        │
          ┌────────────────────────┴────────────────────────┘
          │
          ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                           DATA LAYER                                        │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  PostgreSQL  │  │    Redis     │  │  OpenSearch  │  │    MinIO     │    │
│  │   (7 DBs)    │  │  (5 inst.)   │  │  (Search)    │  │  (S3 Files)  │    │
│  │              │  │              │  │              │  │              │    │
│  │ - vondi_db    │  │ - Sessions   │  │ - Listings   │  │ - Images     │    │
│  │ - auth_db    │  │ - Cache      │  │ - Full-text  │  │ - Documents  │    │
│  │ - listings   │  │ - Queues     │  │ - Facets     │  │ - Backups    │    │
│  │ - delivery   │  │ - Streams    │  │              │  │              │    │
│  │ - wms_db     │  │              │  │              │  │              │    │
│  │ - notify_db  │  │              │  │              │  │              │    │
│  │ - pvz_db     │  │              │  │              │  │              │    │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Сервисы

### 3.1 Backend Monolith

**Назначение:** Основной бизнес-логики, оркестрация микросервисов, REST API для frontend.

| Параметр | Значение |
|----------|----------|
| Язык | Go 1.23 |
| Framework | Fiber v2 |
| Port | 3000 |
| Директория | `/vondi/backend` |
| Структура | Domain-based модули |

**Модули:**
- `users` — пользователи, профили
- `listings` — объявления (legacy + v2)
- `orders` — заказы, корзина
- `payments` — транзакции, кошельки
- `delivery` — доставка (оркестрация)
- `storefronts` — магазины продавцов
- `admin` — админ-панель
- `search` — поиск (OpenSearch)

### 3.2 Auth Service

**Назначение:** Аутентификация, авторизация, управление сессиями.

| Параметр | Значение |
|----------|----------|
| Язык | Go 1.23 |
| Архитектура | DDD (Clean Architecture) |
| HTTP Port | 28086 |
| gRPC Port | 20053 |
| DB | auth_db (PostgreSQL :25432) |
| Redis | :26379 |

**Функции:**
- JWT RS256 (access 15m / refresh 720h)
- Google OAuth 2.0
- Firebase Phone Auth
- RBAC (28 ролей, 67 разрешений)
- Session management
- Account lockout (5 attempts)

### 3.3 Listings Service

**Назначение:** Управление товарами, категориями, атрибутами, инвентарём.

| Параметр | Значение |
|----------|----------|
| Язык | Go 1.23 |
| Архитектура | DDD |
| HTTP Port | 8084 |
| gRPC Port | 50053 |
| DB | listings_dev_db (:35434) |
| Redis | :36379 |

**gRPC сервисы:**
- ListingsService — CRUD объявлений
- SearchService — поиск и фильтрация
- CategoriesService — категории v2
- VariantsService — варианты товаров
- AttributesService — унифицированные атрибуты

### 3.4 Delivery Service

**Назначение:** Расчёт доставки, интеграция с Post Express.

| Параметр | Значение |
|----------|----------|
| Версия | 0.1.6 |
| HTTP Port | 8082 |
| gRPC Port | 50052 |
| DB | delivery_db (:35432) |

**Провайдеры:**
- Post Express Serbia (курьер + ПВЗ)
- Самовывоз (Pickup)
- Расширяемая архитектура (Factory Pattern)

### 3.5 Warehouse Service (WMS)

**Назначение:** Управление складами, picking, packing, returns, QC.

| Параметр | Значение |
|----------|----------|
| HTTP Port | 8085 |
| gRPC Port | 50055 |
| DB | wms_db (:35435) |
| Redis | :36381 |

**Функции:**
- Multi-warehouse support
- Zone/Location management
- Picking workflows (FIFO)
- Packing с контролем веса
- Returns processing
- Quality Control
- Gamification (Bronze→Platinum)
- Worker management

### 3.6 Payment Service

**Назначение:** Платежи, эскроу, кошельки, выплаты.

| Параметр | Значение |
|----------|----------|
| HTTP Port | 8087 |
| gRPC Port | 30051 |
| DB | payment_db (:35433) |

**Функции:**
- Escrow (холдирование средств)
- Multi-wallet (RSD, EUR)
- Fee calculation
- Payout management
- Transaction history

### 3.7 Notifications Service

**Назначение:** Отправка уведомлений (email, Telegram, push).

| Параметр | Значение |
|----------|----------|
| HTTP Port | 8088 |
| gRPC Port | 50054 |
| DB | notifications_db (:35436) |

**Каналы:**
- Email (Zoho Mail API)
- Telegram Bot
- Push notifications (future)
- SMS (future)

### 3.8 PVZ Service

**Назначение:** Управление сетью пунктов выдачи заказов (ПВЗ), партнёрская программа.

| Параметр | Значение |
|----------|----------|
| Язык | Go 1.23 |
| Архитектура | DDD |
| HTTP Port | 8089 |
| gRPC Port | 50056 |
| DB | pvz_db (:35438) |
| Redis | :36384 |

**Функции:**
- Dual model: Full ПВЗ + Mikro-ПВЗ партнёров
- Operator interface (смены, приёмка, выдача)
- Capacity tracking (real-time мониторинг загрузки)
- Partner program (onboarding, комиссии, выплаты)
- QR scanning (приёмка от курьеров)
- Pickup code system (6-значный код)
- Geo-based routing (PostGIS)
- Returns processing

---

## 4. Frontend

### 4.1 Next.js Application

| Параметр | Значение |
|----------|----------|
| Framework | Next.js 15.3.2 |
| React | 19 |
| TypeScript | 5 |
| Port | 3001 |
| Styling | Tailwind CSS |
| State | Redux Toolkit + React Query |

**Структура:**
```
frontend/src/
├── app/                 # App Router (69 routes)
│   ├── [locale]/        # i18n (en, ru, sr)
│   └── api/v2/          # BFF Proxy
├── components/          # 85+ React компонентов
├── hooks/               # 46 кастомных хуков
├── store/               # Redux (18 slices)
├── services/            # API clients
├── messages/            # i18n translations
└── types/               # TypeScript types
```

### 4.2 BFF Pattern

```
Browser → /api/v2/* (Next.js) → /api/v1/* (Backend)
          ↳ httpOnly cookies    ↳ Authorization: Bearer
```

Frontend никогда не обращается к backend напрямую. BFF proxy:
- Добавляет JWT из httpOnly cookie
- Трансформирует ответы
- Обрабатывает ошибки

---

## 5. Базы данных

### 5.1 PostgreSQL Instances

| Database | Port | Owner | Назначение |
|----------|------|-------|------------|
| vondi_db | 5433 | postgres | Монолит (80+ таблиц) |
| auth_db | 25432 | auth_user | Auth Service (9 таблиц) |
| listings_dev_db | 35434 | listings_user | Listings (25+ таблиц) |
| delivery_db | 35432 | delivery_user | Delivery (8 таблиц) |
| wms_db | 35435 | wms_user | WMS (30+ таблиц) |
| notifications_db | 35436 | notify_user | Notifications (5 таблиц) |
| pvz_db | 35438 | pvz_user | PVZ Service (10 таблиц) |

### 5.2 Redis Instances

| Instance | Port | Назначение |
|----------|------|------------|
| Auth Redis | 26379 | Sessions, tokens |
| Listings Redis | 36379 | Cache, rate limiting |
| Delivery Redis | 36380 | Rate calculations cache |
| WMS Redis | 36381 | Event streams, cache |
| PVZ Redis | 36384 | Capacity cache, event streams |

### 5.3 OpenSearch

| Параметр | Значение |
|----------|----------|
| Port | 9200 |
| Index | marketplace_listings |
| Назначение | Full-text search, facets |

### 5.4 MinIO (S3)

| Параметр | Значение |
|----------|----------|
| Endpoint | s3.vondi.rs |
| Port | 9000 |
| Buckets | listings-images, avatars, documents |

---

## 6. Коммуникация

### 6.1 gRPC (Синхронная)

```
Monolith ──gRPC──► Auth Service     (:20053)
         ──gRPC──► Listings Service (:50053)
         ──gRPC──► Delivery Service (:50052)
         ──gRPC──► WMS Service      (:50055)
         ──gRPC──► Payment Service  (:30051)
         ──gRPC──► Notifications    (:50054)
         ──gRPC──► PVZ Service      (:50056)
```

### 6.2 Redis Streams (Асинхронная)

| Stream | Publisher | Subscribers |
|--------|-----------|-------------|
| wms:events:picking | WMS | Monolith |
| wms:events:packing | WMS | Monolith |
| wms:events:inventory | WMS | Listings |
| orders:status:changed | Monolith | WMS, Notifications |
| payments:completed | Payment | Monolith, WMS |
| pvz.events | PVZ | Monolith, Delivery, Notifications |
| delivery.events | Delivery | PVZ |

### 6.3 REST API

Frontend → Monolith (/api/v1/*)
- JSON over HTTPS
- JWT Bearer authentication
- Pagination, filtering, sorting

---

## 7. Безопасность

### 7.1 Аутентификация

- **JWT RS256** — асимметричные ключи
- **Access Token** — 15 минут
- **Refresh Token** — 30 дней
- **Rotation** — при каждом refresh

### 7.2 Авторизация

- **RBAC** — Role-Based Access Control
- **28 ролей** (admin, seller, buyer, warehouse_worker, etc.)
- **67 разрешений** в 12 категориях
- **3-уровневая проверка** (role → permission → ownership)

### 7.3 Защита данных

- **bcrypt** — хеширование паролей (cost 12)
- **HTTPS** — TLS 1.3
- **CORS** — whitelist доменов
- **Rate Limiting** — защита от brute-force
- **Account Lockout** — после 5 неудачных попыток

---

## 8. Деплой

### 8.1 Окружения

| Окружение | URL | Назначение |
|-----------|-----|------------|
| Development | localhost | Локальная разработка |
| Staging | dev.vondi.rs | Тестирование |
| Production | vondi.rs | Продакшн |

### 8.2 Docker

Все сервисы запускаются в Docker:
- docker-compose.yml для каждого сервиса
- Harbor registry для образов
- Self-hosted GitHub Runner для CI/CD

### 8.3 CI/CD

- GitHub Actions
- Self-hosted runner на svetu.rs
- Автоматические тесты
- Автодеплой на dev.vondi.rs

---

## 9. Мониторинг

### 9.1 Логирование

- **zerolog** — структурированные логи (JSON)
- **Уровни**: debug, info, warn, error
- **Контекст**: request_id, user_id, trace_id

### 9.2 Health Checks

Каждый сервис имеет:
- `/health` — базовая проверка
- `/health/ready` — готовность к трафику
- `/health/live` — liveness probe

---

## 10. Версионирование

| Компонент | Текущая версия |
|-----------|----------------|
| Auth Service | 1.20.1 |
| Listings Service | 1.0.0 |
| Delivery Service | 0.1.6 |
| WMS Service | 0.1.0 |
| Payment Service | 0.1.0 |
| Notifications | 0.1.0 |
| PVZ Service | 1.0.0 |
| Frontend | 0.1.0 |
| Monolith | 0.1.0 |

---

**Последнее обновление:** 2025-12-21
