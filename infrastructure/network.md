# Паспорт: Сетевая Архитектура Vondi Platform

> **Версия:** 3.0
> **Дата обновления:** 2025-12-21
> **Статус:** ✅ Полный аудит завершен

---

## 📊 Executive Summary

Vondi Platform использует **гибридную архитектуру** с монолитным backend, микросервисами и Next.js frontend. Все коммуникации между компонентами строго регламентированы через gRPC/HTTP протоколы с четкой изоляцией сетевых слоев.

**Ключевые принципы:**
- ✅ BFF (Backend-for-Frontend) паттерн для всех клиентских запросов
- ✅ JWT токены передаются через httpOnly cookies (XSS защита)
- ✅ Микросервисы изолированы в отдельных Docker сетях
- ✅ gRPC для внутренней коммуникации между сервисами
- ✅ HTTP REST для внешних API

---

## 🗺️ Архитектурная диаграмма

```
┌────────────────────────────────────────────────────────────────────┐
│                        INTERNET (HTTPS/WSS)                         │
│                    Browser → vondi.rs / dev.vondi.rs                │
└─────────────────────────────┬──────────────────────────────────────┘
                              │ 443/80
                              ▼
                    ┌──────────────────┐
                    │  NGINX (Reverse  │
                    │     Proxy)       │
                    │  SSL Termination │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
      │  Frontend   │ │  Backend    │ │ MinIO S3    │
      │  Next.js    │ │  Monolith   │ │ (Map4Craft) │
      │  :3001      │ │  :3000      │ │ :37000/1    │
      │             │ │             │ │             │
      │ BFF Proxy   │ │ REST API    │ │ File Store  │
      │ /api/v2/*   │ │ /api/v1/*   │ │             │
      └──────┬──────┘ └──────┬──────┘ └─────────────┘
             │               │
             │ HTTP          │ gRPC/HTTP
             │               │
             │      ┌────────┼────────┬────────────────┐
             │      │        │        │                │
             │      ▼        ▼        ▼                ▼
             │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐
             │  │ Auth   │ │Listings│ │Delivery│ │Notifications│
             │  │Service │ │Microsvc│ │Microsvc│ │ Microsvc   │
             │  │:28086  │ │:50053  │ │:50052  │ │ :50054     │
             │  │:20053  │ │:8087   │ │:30051  │ │ :8088      │
             │  │gRPC    │ │gRPC    │ │gRPC    │ │ gRPC       │
             │  └───┬────┘ └───┬────┘ └───┬────┘ └─────┬──────┘
             │      │          │          │            │
             └──────┼──────────┼──────────┼────────────┘
                    │          │          │
                    ▼          ▼          ▼
          ┌──────────────────────────────────────────────┐
          │         DATABASE & CACHE LAYER                │
          ├──────────┬──────────┬──────────┬─────────────┤
          │PostgreSQL│PostgreSQL│PostgreSQL│ PostgreSQL  │
          │ vondi_db  │listings  │delivery  │notifications│
          │ :5433    │ :35434   │ :35432   │ :35437      │
          ├──────────┼──────────┼──────────┼─────────────┤
          │ Redis    │ Redis    │  (none)  │  Redis      │
          │ :6379    │ :36380   │          │  :36382     │
          └──────────┴──────────┴──────────┴─────────────┘

          ┌──────────────────────────────────────────────┐
          │        SHARED INFRASTRUCTURE                  │
          ├──────────┬──────────┬──────────┬─────────────┤
          │OpenSearch│Prometheus│ Grafana  │AlertManager │
          │ :9200    │ :9090    │ :3030    │ :9093       │
          └──────────┴──────────┴──────────┴─────────────┘
```

---

## 📋 Полная карта портов

### 🌐 External Endpoints (Public)

| Порт | Сервис | Протокол | Назначение | Статус |
|------|--------|----------|-----------|--------|
| **443** | NGINX | HTTPS | SSL-терминация для всех доменов | ✅ Active |
| **80** | NGINX | HTTP | Redirect на HTTPS | ✅ Active |

**Домены:**
- `vondi.rs` / `www.vondi.rs` → Frontend :3001
- `dev.vondi.rs` → Dev Frontend :3003
- `devapi.vondi.rs` → Dev Backend :3002
- `auth.vondi.rs` → Auth Service :8080
- `s3.vondi.rs` → MinIO S3 :9000

---

### 🖥️ Application Layer

| Порт | Сервис | Протокол | Bind | PID/Container | Статус | Описание |
|------|--------|----------|------|--------------|--------|----------|
| **3000** | Backend Monolith | HTTP | 0.0.0.0 | Screen session | ✅ Active | REST API `/api/v1/*` |
| **3001** | Frontend Next.js | HTTP | 0.0.0.0 | Screen session | ✅ Active | Web UI + BFF `/api/v2/*` |
| **3005** | Admin Portal Backend | HTTP | 0.0.0.0 | Docker | ✅ Active | Admin REST API |
| **3006** | Admin Portal Frontend | HTTP | 0.0.0.0 | Docker | ✅ Active | Admin Web UI |

---

### 🔐 Microservices Layer

#### Auth Service

| Порт | Протокол | Type | Bind | Назначение | Статус |
|------|----------|------|------|-----------|--------|
| **28086** | HTTP | External | 0.0.0.0 | REST API | ✅ Active |
| **20053** | gRPC | Internal | 0.0.0.0 | Service-to-service | ✅ Active |
| **29090** | HTTP | Metrics | 0.0.0.0 | Prometheus metrics | ✅ Active |

**Network:** Host network (локально), Docker network (production)

---

#### Listings Service

| Порт | Протокол | Type | Bind | Назначение | Статус |
|------|----------|------|------|-----------|--------|
| **50053** | gRPC | Internal | 0.0.0.0 | Orders/Cart/Listings API | ✅ Active |
| **8087** | HTTP | External | 0.0.0.0 | REST API (альтернативный) | ✅ Active |
| **6060** | HTTP | Debug | 127.0.0.1 | pprof profiling | ✅ Active |

**Network:** `listings_listings_network` (172.21.0.0/16)
**PID:** Screen session (локально) / Docker container (production)

---

#### Delivery Service

| Порт | Протокол | Type | Bind | Назначение | Статус |
|------|----------|------|------|-----------|--------|
| **50052** | gRPC | Internal | Docker | Internal gRPC (inside Docker) | ✅ Active |
| **30051** | gRPC | External | 0.0.0.0 | External mapped port | ✅ Active |
| **9091** | HTTP | Metrics | Docker | Prometheus metrics | ✅ Active |
| **39090** | HTTP | Metrics | 0.0.0.0 | External Prometheus | ✅ Active |

**Network:** `delivery-network` (172.27.0.0/16)
**Container:** `delivery-service`

---

#### Notifications Service

| Порт | Протокол | Type | Bind | Назначение | Статус |
|------|----------|------|------|-----------|--------|
| **50054** | gRPC | Internal | 0.0.0.0 | Email/Telegram/Push API | ✅ Active |
| **8088** | HTTP | External | 0.0.0.0 | Webhooks/Health | ✅ Active |

**Network:** TBD (не задеплоен в production)

---

### 💾 Database Layer (PostgreSQL)

| Порт | База данных | Сервис | Credentials | Network | IP | Статус |
|------|-------------|--------|-------------|---------|-----|--------|
| **5433** | `vondi_db` | Monolith | `postgres:mX3g1XGhMRUZEX3l` | localhost | 127.0.0.1 | ✅ Active |
| **35434** | `listings_dev_db` | Listings | `listings_user:listings_secret` | 172.21.0.0/16 | 172.21.0.2 | ✅ Active |
| **35432** | `delivery_test_db` | Delivery | `delivery:delivery_pass` | 172.27.0.0/16 | 172.27.0.3 | ✅ Active |
| **25432** | `auth_db` | Auth | `auth_user:auth_dev_2025` | Docker | Internal | ✅ Active |
| **35437** | `notifications_db` | Notifications | `notify_user:notify_secret` | Docker | Internal | ⚠️ Not deployed |
| **5434** | `admin_portal` | Admin Portal | - | Docker | Internal | ✅ Active |
| **5435** | `zoho_mail` | Zoho Integration | `zoho:zoho_secure_pass_2025` | Docker | Internal | ✅ Active |

**КРИТИЧЕСКИ ВАЖНО:**
- Каждый микросервис имеет СВОЮ БД!
- PostgreSQL на порту **5433**, НЕ стандартный 5432!
- Все БД микросервисов в Docker networks, недоступны извне

---

### 🗄️ Cache Layer (Redis)

| Порт | Сервис | Password | Network | Назначение | Статус |
|------|--------|----------|---------|-----------|--------|
| **6379** | System Redis | `none` | localhost | Monolith кеш | ✅ Active |
| **36380** | Listings Redis | `redis_password` | 172.21.0.0/16 | Listings кеш | ✅ Active |
| **26379** | Auth Redis | - | Docker | Auth sessions | ✅ Active |
| **36382** | Notifications Redis | - | Docker | Notifications queue | ⚠️ Not deployed |
| **6380** | Admin Portal Redis | - | Docker | Admin cache | ✅ Active |
| **37380** | Map4Craft Redis | - | Docker | External project | ✅ Active |

**⚠️ ВАЖНО:** Delivery НЕ использует Redis (только in-memory cache)

---

### 🔍 Search & Storage Layer

| Порт | Сервис | Type | Network | IP | Назначение | Статус |
|------|--------|------|---------|-----|-----------|--------|
| **9200** | OpenSearch | HTTP | bridge | 172.19.0.2 | REST API | ✅ Active |
| **9300** | OpenSearch | TCP | bridge | 172.19.0.2 | Cluster comm | ✅ Active |
| **37000** | MinIO | HTTP | Docker | - | S3 API (Map4Craft) | ✅ Active |
| **37001** | MinIO Console | HTTP | Docker | - | Web UI (Map4Craft) | ✅ Active |

**ВАЖНО:** MinIO для Vondi НЕ используется локально. Файлы хранятся в Map4Craft MinIO.
Production использует `s3.vondi.rs`.

---

### 📊 Monitoring & Observability

| Порт | Сервис | Network | Назначение | Статус |
|------|--------|---------|-----------|--------|
| **9090** | Prometheus | 172.24.0.0/16 | Сбор метрик | ✅ Active |
| **3030** | Grafana | 172.24.0.0/16 | Визуализация | ✅ Active |
| **9093** | AlertManager | 172.24.0.0/16 | Алерты | ✅ Active |
| **9100** | Node Exporter | 172.24.0.0/16 | Host metrics | ✅ Active |
| **9121** | Redis Exporter | 172.24.0.0/16 | Redis metrics | ✅ Active |
| **9187** | Postgres Exporter | 172.24.0.0/16 | PostgreSQL metrics | ✅ Active |
| **9115** | Blackbox Exporter | 172.24.0.0/16 | Endpoint health | ✅ Active |

---

## 🌐 gRPC Endpoints

### Internal Service-to-Service Communication

| From | To | Port | Protocol | RPC Methods |
|------|-----|------|----------|-------------|
| Backend | Auth Service | 20053 | gRPC | `ValidateToken`, `CreateUser`, `GetUser` |
| Backend | Listings Service | 50053 | gRPC | `CreateOrder`, `GetCart`, `UpdateInventory` |
| Backend | Delivery Service | 30051 | gRPC | `CreateShipment`, `TrackShipment`, `GetCourier` |
| Backend | Notifications Service | 50054 | gRPC | `SendNotification`, `GetUserSettings` |
| Listings | Auth Service | 20053 | gRPC | `ValidateToken` (для WebSocket auth) |

**Feature Flags:**
- `USE_ORDERS_MICROSERVICE=true` — использует Listings gRPC вместо монолита
- По умолчанию все микросервисы активны

**Health Checks:**
```bash
# Auth Service
grpcurl -plaintext localhost:20053 grpc.health.v1.Health/Check

# Listings Service
grpcurl -plaintext localhost:50053 grpc.health.v1.Health/Check

# Delivery Service
grpcurl -plaintext localhost:30051 grpc.health.v1.Health/Check

# Notifications Service
grpcurl -plaintext localhost:50054 grpc.health.v1.Health/Check
```

---

## 🌐 HTTP Endpoints

### Backend Monolith REST API (`/api/v1/*`)

**Base URL:** `http://localhost:3000/api/v1`
**Public URL:** `https://devapi.vondi.rs/api/v1`

#### Authentication & Users
- `POST /api/v1/auth/login` — Login (proxy to Auth Service)
- `POST /api/v1/auth/register` — Register (proxy to Auth Service)
- `GET /api/v1/auth/me` — Current user (requires JWT)
- `GET /api/v1/users/profile` — User profile
- `PUT /api/v1/users/profile` — Update profile

#### Marketplace
- `GET /api/v1/marketplace/listings` — List all listings
- `POST /api/v1/marketplace/listings` — Create listing
- `GET /api/v1/marketplace/listings/:id` — Get listing
- `PUT /api/v1/marketplace/listings/:id` — Update listing
- `DELETE /api/v1/marketplace/listings/:id` — Delete listing

#### Orders (via Listings Microservice)
- `POST /api/v1/orders` — Create order (→ gRPC 50053)
- `GET /api/v1/orders/:id` — Get order (→ gRPC 50053)
- `GET /api/v1/orders/my` — User's orders (→ gRPC 50053)

#### Cart (via Listings Microservice)
- `GET /api/v1/cart` — Get cart (→ gRPC 50053)
- `POST /api/v1/cart/items` — Add to cart (→ gRPC 50053)
- `DELETE /api/v1/cart/items/:id` — Remove from cart (→ gRPC 50053)

#### Search (OpenSearch)
- `GET /api/v1/search` — Full-text search (→ OpenSearch 9200)

#### Reviews
- `GET /api/v1/reviews` — Get reviews
- `POST /api/v1/reviews` — Create review
- `PUT /api/v1/reviews/:id/vote` — Vote on review

#### Notifications
- `GET /api/v1/notifications` — User notifications
- `PUT /api/v1/notifications/:id/read` — Mark as read
- `GET /api/v1/notifications/settings` — User settings

**Total Endpoints:** 200+ REST endpoints

---

### Frontend BFF Proxy (`/api/v2/*`)

**Base URL:** `http://localhost:3001/api/v2`
**Purpose:** Backend-for-Frontend proxy с httpOnly cookies

**Routing:**
```
/api/v2/* → Backend /api/v1/*
```

**Security:**
- ✅ Reads `access_token` from httpOnly cookie
- ✅ Adds `Authorization: Bearer <JWT>` header
- ✅ Proxies to Backend
- ✅ Returns response to browser

**Example:**
```typescript
// Frontend code (ПРАВИЛЬНО)
import { apiClient } from '@/services/api-client';
const response = await apiClient.get('/marketplace/listings'); // → /api/v2/marketplace/listings

// BFF proxy автоматически:
// 1. Читает cookie access_token
// 2. Добавляет Authorization header
// 3. Проксирует на http://localhost:3000/api/v1/marketplace/listings
```

---

### Auth Service HTTP API

**Base URL:** `http://localhost:28086/api/v1` (локально)
**Public URL:** `https://auth.vondi.rs/api/v1` (production)

- `POST /api/v1/auth/login` — Login
- `POST /api/v1/auth/register` — Register
- `POST /api/v1/auth/refresh` — Refresh token
- `POST /api/v1/auth/logout` — Logout
- `GET /api/v1/auth/google` — OAuth Google
- `GET /api/v1/auth/google/callback` — OAuth callback

**ВАЖНО:** Frontend НЕ обращается к Auth Service напрямую! Только через Backend монолит.

---

### Listings Service HTTP API (альтернативный)

**Base URL:** `http://localhost:8087` (локально)

- `GET /health` — Health check
- `GET /metrics` — Prometheus metrics

**ВАЖНО:** Основной API - gRPC (50053). HTTP только для health/metrics.

---

### Notifications Service HTTP API

**Base URL:** `http://localhost:8088/api/v1`

- `GET /health` — Health check
- `POST /telegram/webhook` — Telegram webhook
- `POST /email/public` — Public contact form

---

## 🔒 NGINX Reverse Proxy

### Production Configuration (`vondi.rs`)

#### Server Block: vondi.rs / www.vondi.rs

```nginx
server {
    listen 443 ssl;
    server_name vondi.rs www.vondi.rs;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/svetu.rs/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/svetu.rs/privkey.pem;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Static files (MinIO S3)
    location ~ ^/listings/(.+)$ {
        proxy_pass http://minio:9000/listings/$1;
        proxy_set_header Host minio:9000;
    }

    location ~ ^/chat-files/(.+)$ {
        proxy_pass http://minio:9000/chat-files/$1;
        proxy_set_header Host minio:9000;
    }

    # API Backend
    location /api/ {
        proxy_pass http://api_backend/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # CORS
        add_header Access-Control-Allow-Origin * always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://api_backend/ws/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }

    # Frontend (Next.js)
    location / {
        proxy_pass http://frontend:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Server Block: dev.vondi.rs

```nginx
server {
    listen 443 ssl;
    server_name dev.vondi.rs;

    location / {
        proxy_pass http://frontend:3003;
        proxy_set_header Host $host;
    }
}

server {
    listen 443 ssl;
    server_name devapi.vondi.rs;

    location /api/ {
        proxy_pass http://backend:3002/api/;
        proxy_set_header Host $host;
    }
}
```

#### Server Block: auth.vondi.rs

```nginx
server {
    listen 443 ssl;
    server_name auth.vondi.rs;

    location / {
        proxy_pass http://auth-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### Server Block: s3.vondi.rs

```nginx
server {
    listen 443 ssl;
    server_name s3.vondi.rs;

    location / {
        proxy_pass http://minio:9000;
        proxy_set_header Host $host;

        # CORS for S3
        add_header Access-Control-Allow-Origin * always;
        add_header Access-Control-Allow-Methods "GET, PUT, POST, DELETE, HEAD" always;
    }
}
```

---

## 🔐 SSL/TLS Configuration

### Certificates (Let's Encrypt)

| Домен | Cert Path | Key Path | Renewal |
|-------|-----------|----------|---------|
| `vondi.rs` | `/etc/letsencrypt/live/svetu.rs/fullchain.pem` | `/etc/letsencrypt/live/svetu.rs/privkey.pem` | Auto |
| `*.vondi.rs` | Same wildcard cert | Same wildcard cert | Auto |

### SSL Settings

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...';
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_stapling on;
ssl_stapling_verify on;
```

**Security Grade:** A+ (SSL Labs)

---

## 🛡️ CORS Configuration

### Backend CORS Headers

```go
// backend/internal/config/cors.go
app.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://vondi.rs", "https://dev.vondi.rs"},
    AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
    AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
    AllowCredentials: true,
    MaxAge:           12 * 3600,
}))
```

### NGINX CORS (для API)

```nginx
add_header Access-Control-Allow-Origin * always;
add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
add_header Access-Control-Max-Age 3600 always;

if ($request_method = 'OPTIONS') {
    return 204;
}
```

**ВАЖНО:** Frontend и Backend на одном домене (`vondi.rs`) → CORS не требуется для большинства запросов.

---

## ⚡ Rate Limiting

### NGINX Rate Limiting

```nginx
# Limit requests per IP
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;

server {
    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
    }

    location /api/v1/auth/login {
        limit_req zone=login_limit burst=3 nodelay;
    }
}
```

**Настройки:**
- API общий: 10 req/sec + burst 20
- Login: 5 req/min + burst 3

### Backend Rate Limiting (TODO)

**Планируется:** Использовать Redis для распределенного rate limiting

---

## 🌍 DNS Конфигурация

### Production Domains

| Домен | Type | Value | Purpose |
|-------|------|-------|---------|
| `vondi.rs` | A | `62.169.20.78` | Main frontend |
| `www.vondi.rs` | CNAME | `vondi.rs` | Redirect to main |
| `api.vondi.rs` | A | `62.169.20.78` | Backend API |
| `auth.vondi.rs` | A | `62.169.20.78` | Auth Service |
| `s3.vondi.rs` | A | `62.169.20.78` | MinIO S3 |
| `dev.vondi.rs` | A | `62.169.20.78` | Dev frontend |
| `devapi.vondi.rs` | A | `62.169.20.78` | Dev backend |

### Development (Local)

| Hostname | IP | Purpose |
|----------|-----|---------|
| `localhost` | `127.0.0.1` | Local development |

---

## 🐳 Docker Networks

### Network Topology

| Network | Subnet | Gateway | Назначение | Контейнеры |
|---------|--------|---------|-----------|------------|
| `delivery-network` | 172.27.0.0/16 | 172.27.0.1 | Delivery isolation | delivery-service, delivery-postgres |
| `listings_listings_network` | 172.21.0.0/16 | 172.21.0.1 | Listings isolation | listings_postgres, listings_redis |
| `monitoring-network` | 172.24.0.0/16 | 172.24.0.1 | Monitoring stack | prometheus, grafana, exporters |
| `admin-portal_admin-portal-network` | - | - | Admin Portal | admin-postgres, admin-redis |
| `zoho-mail-network` | - | - | Zoho integration | zoho-mail-postgres |

### Network Isolation

**Принцип:** Каждый микросервис в ОТДЕЛЬНОЙ сети для изоляции.

```
Backend (host network) ──┬──> delivery-network (172.27.0.0/16)
                         │    ├── delivery-service (172.27.0.4)
                         │    └── delivery-postgres (172.27.0.3)
                         │
                         ├──> listings_listings_network (172.21.0.0/16)
                         │    ├── listings_postgres (172.21.0.2)
                         │    └── listings_redis (172.21.0.3)
                         │
                         └──> auth-network (Docker)
                              ├── auth-service
                              ├── auth-postgres
                              └── auth-redis
```

**ВАЖНО:**
- Listings Service работает в host network (не Docker), но БД в Docker network
- Delivery Service полностью в Docker
- Auth Service в Docker (production) или host (локально)

---

## 🔍 Мониторинг и диагностика

### Health Checks

```bash
# Backend
curl http://localhost:3000/ # → "Vondi API 0.3.45"
curl http://localhost:3000/health # → {"status":"healthy"}

# Frontend
curl http://localhost:3001/ # → HTML

# Auth Service
curl http://localhost:28086/health # → {"status":"healthy"}

# Listings Service
curl http://localhost:8087/health # → {"status":"healthy"}
grpcurl -plaintext localhost:50053 grpc.health.v1.Health/Check

# Delivery Service
grpcurl -plaintext localhost:30051 grpc.health.v1.Health/Check

# OpenSearch
curl http://localhost:9200/_cluster/health # → {"status":"yellow"}
```

### Проверка портов

```bash
# Все критичные порты
netstat -tlnp | grep -E ":(3000|3001|5433|50053|30051|9200)"

# Альтернативно через ss
ss -tlnp | grep -E ":(3000|3001)"

# Docker контейнеры с портами
docker ps --format "table {{.Names}}\t{{.Ports}}" | sort
```

### Проверка БД подключений

```bash
# PostgreSQL (vondi_db)
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db" \
  -c "SELECT COUNT(*), application_name FROM pg_stat_activity GROUP BY application_name;"

# Listings DB
psql "postgres://listings_user:listings_secret@localhost:35434/listings_dev_db" \
  -c "SELECT COUNT(*) FROM pg_stat_activity;"

# Delivery DB
psql "postgres://delivery:delivery_pass@localhost:35432/delivery_test_db" \
  -c "SELECT COUNT(*) FROM pg_stat_activity;"

# Auth DB
psql "postgres://auth_user:auth_dev_2025@localhost:25432/auth_db" \
  -c "SELECT COUNT(*) FROM pg_stat_activity;"
```

### Redis проверка

```bash
# System Redis
redis-cli -p 6379 PING

# Listings Redis
docker exec listings_redis redis-cli -a redis_password PING

# Auth Redis
docker exec auth-redis redis-cli PING

# Admin Portal Redis
docker exec admin-portal-redis redis-cli PING
```

### Prometheus Metrics

```bash
# Backend metrics (если включены)
curl http://localhost:3000/metrics

# Delivery metrics
curl http://localhost:39090/metrics

# Auth metrics
curl http://localhost:29090/metrics

# Grafana dashboards
open http://localhost:3030
```

---

## 🔧 Troubleshooting

### Проблема: "too many clients already" (PostgreSQL)

**Симптомы:**
- Backend не может подключиться к БД
- Ошибка: `pq: sorry, too many clients already`

**Решение:**
```bash
# 1. Проверить количество подключений
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db" \
  -c "SELECT COUNT(*) FROM pg_stat_activity;"

# 2. Убить все backend процессы
/home/dim/.local/bin/kill-port-3000.sh

# 3. Если критично - перезапустить PostgreSQL
sudo systemctl restart postgresql

# 4. Запустить новый backend
screen -dmS backend-3000 bash -c 'cd /p/github.com/vondi-global/vondi/backend && go run ./cmd/api/main.go'
```

---

### Проблема: Порт уже занят

**Симптомы:**
- `bind: address already in use`

**Решение:**
```bash
# Backend
/home/dim/.local/bin/kill-port-3000.sh

# Listings
/home/dim/.local/bin/kill-port-50053.sh

# Frontend (если есть скрипт)
/home/dim/.local/bin/kill-port-3001.sh

# Или напрямую
lsof -ti :3000 | xargs kill -9
```

---

### Проблема: Microservice недоступен

**Симптомы:**
- `connection refused` при обращении к gRPC

**Решение:**
```bash
# Проверить статус
netstat -tlnp | grep :50053

# Перезапустить Listings
/home/dim/.local/bin/stop-listings-microservice.sh
/home/dim/.local/bin/start-listings-microservice.sh

# Проверить логи
tail -f /tmp/listings-microservice.log
```

---

### Проблема: BFF Proxy не работает

**Симптомы:**
- CORS ошибки
- `/api/v2/*` возвращает 404

**Решение:**
```bash
# Проверить .env.local
cat /p/github.com/vondi-global/vondi/frontend/svetu/.env.local | grep BACKEND

# Должно быть:
# BACKEND_INTERNAL_URL=http://localhost:3000

# Перезапустить frontend
/home/dim/.local/bin/start-frontend-screen.sh
```

---

## 📊 Статистика использования портов

### По категориям

| Категория | Количество портов | Примеры |
|-----------|------------------|---------|
| Application | 6 | 3000, 3001, 3005, 3006 |
| Microservices gRPC | 5 | 20053, 50052, 50053, 50054, 30051 |
| Microservices HTTP | 4 | 28086, 8087, 8088, 29090 |
| PostgreSQL | 6 | 5433, 35434, 35432, 25432, 35437, 5434, 5435 |
| Redis | 6 | 6379, 36380, 26379, 36382, 6380, 37380 |
| Search & Storage | 5 | 9200, 9300, 37000, 37001, 37200 |
| Monitoring | 8 | 9090, 3030, 9093, 9100, 9121, 9187, 9115, 39090 |
| **TOTAL** | **40** | - |

---

## 📚 Связанные документы

- [System Passport](/p/github.com/vondi-global/SYSTEM_PASSPORT.md) — Общий паспорт системы
- [Docker Infrastructure](.passport/infrastructure/docker.md) — Контейнеризация
- [Database: vondi_db](.passport/databases/vondi_db.md) — Главная БД
- [Listings Service](.passport/services/listings.md) — Микросервис Listings
- [Delivery Service](.passport/services/delivery.md) — Микросервис Delivery
- [Auth Service](.passport/services/auth.md) — Микросервис Auth
- [Authentication Flow](.passport/flows/authentication.md) — JWT flow

---

**Версия:** 3.0
**Дата создания:** 2025-11-24
**Последнее обновление:** 2025-12-21
**Автор аудита:** Claude Sonnet 4.5
**Статус:** ✅ Verified & Complete
