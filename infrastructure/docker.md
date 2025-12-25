# Docker Infrastructure Passport

> **Версия:** 2.0.0 | **Обновлено:** 2025-12-21

## Краткое описание

Полная карта Docker контейнеров платформы Vondi: микросервисы, базы данных, инфраструктура, мониторинг, тестовые окружения.

**Всего контейнеров:** 43+ (25 production + 9 monitoring + внешние проекты)

---

## Оглавление

1. [Обзор архитектуры](#обзор-архитектуры)
2. [Production окружение](#production-окружение)
3. [Development окружение](#development-окружение)
4. [Микросервисы](#микросервисы)
5. [Базы данных](#базы-данных)
6. [Networks](#networks)
7. [Volumes](#volumes)
8. [Environment Variables](#environment-variables)
9. [Health Checks](#health-checks)
10. [Порты (полная карта)](#порты-полная-карта)
11. [Harbor Registry](#harbor-registry)
12. [Команды запуска](#команды-запуска)
13. [Monitoring](#monitoring)

---

## Обзор архитектуры

### Production Stack

```
┌─────────────────────────────────────────────────────────────┐
│                        Nginx Reverse Proxy                   │
│                     (Harbor: svetu/nginx)                    │
│                  HTTP:80, HTTPS:443                          │
└────────┬────────────────────────────────────────────┬────────┘
         │                                            │
    ┌────▼────────┐                            ┌──────▼───────┐
    │  Frontend   │                            │   Backend    │
    │  (Next.js)  │                            │  (Fiber/Go)  │
    │   PORT:3000 │◄───── BFF Proxy ──────────┤   PORT:3000  │
    └─────────────┘                            └──┬───┬───┬───┘
                                                  │   │   │
         ┌────────────────────────────────────────┘   │   └──────────┐
         ▼                        ▼                   ▼              ▼
┌─────────────────┐    ┌──────────────────┐  ┌──────────────┐  ┌────────────┐
│ Auth Service    │    │ Listings MS      │  │ Delivery MS  │  │ Payment MS │
│ HTTP:28086      │    │ gRPC:50053       │  │ gRPC:50052   │  │ gRPC:50054 │
│ gRPC:20053      │    │ HTTP:8086        │  │ HTTP:9091    │  │ HTTP:8089  │
└────────┬────────┘    └────────┬─────────┘  └──────┬───────┘  └─────┬──────┘
         │                      │                    │                │
         ▼                      ▼                    ▼                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          PostgreSQL Cluster                             │
│  vondi_db:5433 | listings:35434 | delivery:35432 | auth:25432 |...      │
└─────────────────────────────────────────────────────────────────────────┘
         ▼                      ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     Redis Instances (6 instances)                        │
│  system:6379 | listings:36380 | auth:26379 | admin:6380 | notify:36382 │
└─────────────────────────────────────────────────────────────────────────┘
```

### Container Naming Convention

```
{service}[-{role}][-{env}]

Examples:
- hostel_db             # Main DB (prod)
- listings_postgres     # Listings microservice DB
- vondi-dev-backend     # Development backend
- wms_redis             # WMS Redis
```

---

## Production окружение

### docker-compose.prod.yml (Monolith + Infrastructure)

**Файл:** `/p/github.com/vondi-global/vondi/docker-compose.prod.yml`

| Service | Image | Порты | Назначение |
|---------|-------|-------|------------|
| **db** | harbor.svetu.rs/svetu/db/postgres:15 | - | PostgreSQL vondi_db |
| **opensearch** | harbor.svetu.rs/svetu/opensearch:2.11.0 | - | Full-text search |
| **mailserver** | harbor.svetu.rs/svetu/mail/server:latest | 25,587,465,143,993,110,995 | Email server |
| **mail-webui** | harbor.svetu.rs/svetu/mail/webui:latest | - | Roundcube webmail |
| **certbot** | harbor.svetu.rs/svetu/tools/certbot:latest | - | SSL certificates |
| **migrate** | harbor.svetu.rs/svetu/tools/migrate:latest | - | DB migrations |
| **minio** | harbor.svetu.rs/svetu/minio:RELEASE.2023-09-30 | 9000,9001 | S3 storage |
| **createbuckets** | harbor.svetu.rs/svetu/minio/mc:latest | - | MinIO setup |
| **backend** | harbor.svetu.rs/svetu/backend/api:latest | 3000 | Go backend |
| **nginx** | harbor.svetu.rs/svetu/nginx:latest | 80,443 | Reverse proxy |

**Networks:**
- `hostel_network` (bridge)

**Volumes:**
- `db_data` - PostgreSQL data
- `opensearch_data` - OpenSearch indexes (bind: /opt/hostel-data/opensearch)
- `uploads_data` - User uploads (bind: /opt/hostel-data/uploads)
- `roundcube_data` - Webmail data
- `minio_data` - S3 storage (bind: /opt/hostel-data/minio)

---

## Development окружение

### docker-compose.dev.yml (Monolith Development)

**Файл:** `/p/github.com/vondi-global/vondi/docker-compose.dev.yml`

| Service | Image | Container | Порты | ENV Prefix |
|---------|-------|-----------|-------|------------|
| **db** | postgres:15-alpine + PostGIS | - | ${DATABASE_PORT}:5432 | - |
| **redis** | redis:7-alpine | - | ${REDIS_PORT}:6379 | - |
| **opensearch** | opensearchproject/opensearch:2.11.0 | - | ${OPENSEARCH_PORT}:9200 | - |
| **opensearch-dashboards** | opensearchproject/opensearch-dashboards:2.11.0 | - | ${OPENSEARCH_DASHBOARD_PORT}:5601 | - |
| **backend** | Build: ./backend/Dockerfile | - | ${BACKEND_PORT}:3000 | - |
| **frontend** | Build: ./frontend/svetu/Dockerfile | - | ${FRONTEND_PORT}:3000 | NEXT_PUBLIC_* |

**Networks:**
- `vondi_network_dev` (bridge)

**Volumes:**
- `postgres_data_dev`
- `opensearch-data_dev`
- `redis_data_dev`

**Environment Variables:**
```bash
# Database
DATABASE_USER=postgres
DATABASE_PASS=mX3g1XGhMRUZEX3l
DATABASE_DBNAME=vondi_db
DATABASE_PORT=5433

# Redis
REDIS_PORT=6379

# OpenSearch
OPENSEARCH_PORT=9200
OPENSEARCH_DASHBOARD_PORT=5601

# Backend
BACKEND_PORT=3000
FRONTEND_PORT=3001

# MinIO
MINIO_ENDPOINT=s3.vondi.rs
MINIO_ACCESS_KEY=...
MINIO_SECRET_KEY=...
MINIO_BUCKET_NAME=listings
MINIO_CHAT_BUCKET=chat-files
MINIO_STOREFRONT_BUCKET=storefronts
MINIO_REVIEW_PHOTOS_BUCKET=review-photos
```

---

## Микросервисы

### Auth Service

**Файл:** `/p/github.com/vondi-global/auth/docker-compose.yml`

| Service | Image | Container | Порты | ENV Prefix |
|---------|-------|-----------|-------|------------|
| **postgres** | postgres:15-alpine | - | ${VONDIAUTH_DB_PORT}:5432 | VONDIAUTH_ |
| **redis** | redis:7-alpine | - | ${VONDIAUTH_REDIS_PORT}:6379 | VONDIAUTH_ |

**Default Ports:**
- DB: 25432
- Redis: 26379
- HTTP: 28086
- gRPC: 20053
- Prometheus: 29090

**Environment Variables:**
```bash
VONDIAUTH_DB_NAME=auth_db
VONDIAUTH_DB_USER=auth_user
VONDIAUTH_DB_PASSWORD=auth_secret
VONDIAUTH_DB_PORT=25432

VONDIAUTH_REDIS_PASSWORD=redis_secret
VONDIAUTH_REDIS_PORT=26379

VONDIAUTH_SERVER_HTTP_PORT=28086
VONDIAUTH_SERVER_GRPC_PORT=20053
VONDIAUTH_METRICS_PORT=29090
```

**Volumes:**
- `vondiauth_postgres_data`
- `vondiauth_redis_data`

---

### Listings Microservice

**Файл:** `/p/github.com/vondi-global/listings/docker-compose.yml`

| Service | Image | Container | Порты | ENV Prefix |
|---------|-------|-----------|-------|------------|
| **postgres** | postgres:15-alpine | listings_postgres | 35434:5432 | VONDILISTINGS_ |
| **redis** | redis:7-alpine | listings_redis | ${VONDILISTINGS_REDIS_PORT}:6379 | VONDILISTINGS_ |
| **app** | Build: ./Dockerfile | listings_app | - | VONDILISTINGS_ |

**Default Ports:**
- PostgreSQL: 35434
- Redis: 36380
- gRPC: 50053
- HTTP: 8086
- pprof: 6060

**Environment Variables:**
```bash
VONDILISTINGS_DB_NAME=listings_dev_db
VONDILISTINGS_DB_USER=listings_user
VONDILISTINGS_DB_PASSWORD=listings_secret
VONDILISTINGS_DB_PORT=35434

VONDILISTINGS_REDIS_PASSWORD=redis_password
VONDILISTINGS_REDIS_PORT=36380

VONDILISTINGS_GRPC_PORT=50053
VONDILISTINGS_HTTP_PORT=8086
```

**Network:**
- `listings_network` (bridge)
- app: `network_mode: host` (для доступа к localhost:5433)

**Volumes:**
- `postgres_data`
- `redis_data`

---

### Delivery Microservice

**Файл:** `/p/github.com/vondi-global/delivery/docker-compose.yml`

| Service | Image | Container | Порты |
|---------|-------|-----------|-------|
| **postgres** | postgis/postgis:17-3.5-alpine | delivery-postgres | 35432:5432 |

**Environment Variables:**
```bash
POSTGRES_USER=delivery_user
POSTGRES_PASSWORD=delivery_password
POSTGRES_DB=delivery_db
```

**Volumes:**
- `postgres_data`

**Note:** Redis контейнер удалён (рудимент). Delivery не использует Redis.

---

### Payment Service

**Файл:** `/p/github.com/vondi-global/payment/docker-compose.yml`

| Service | Image | Container | Порты | ENV Prefix |
|---------|-------|-----------|-------|------------|
| **postgres** | postgres:16 | - | ${VONDIPAYMENT_DB_PORT}:5432 | VONDIPAYMENT_ |

**Default Ports:**
- PostgreSQL: 35433
- gRPC: 50054
- HTTP: 8089

**Environment Variables:**
```bash
VONDIPAYMENT_DB_NAME=payment_db
VONDIPAYMENT_DB_USER=payment_user
VONDIPAYMENT_DB_PASSWORD=payment_secret
VONDIPAYMENT_DB_PORT=35433
```

**Volumes:**
- `vondipayment_postgres_data`

---

### Warehouse Management System (WMS)

**Файл:** `/p/github.com/vondi-global/warehouse/docker-compose.yml`

| Service | Image | Container | Порты | ENV Prefix |
|---------|-------|-----------|-------|------------|
| **wms-postgres** | postgres:15-alpine | wms_postgres | 35435:5432 | WMS_ |
| **wms-redis** | redis:7-alpine | wms_redis | 36381:6379 | WMS_ |
| **wms-service** | Build: ./Dockerfile.dev | wms_service | 50055:50055, 8085:8085 | WMS_ |

**Environment Variables:**
```bash
WMS_ENV=development
WMS_DB_HOST=wms-postgres
WMS_DB_PORT=5432
WMS_DB_USER=wms_user
WMS_DB_PASSWORD=wms_secure_pass_2025
WMS_DB_NAME=wms_db

WMS_REDIS_HOST=wms-redis
WMS_REDIS_PORT=6379

WMS_GRPC_PORT=50055
WMS_HTTP_PORT=8085

# Integrations
LISTINGS_GRPC_URL=host.docker.internal:50053
DELIVERY_GRPC_URL=host.docker.internal:50052
PAYMENT_GRPC_URL=host.docker.internal:50054
AUTH_GRPC_URL=host.docker.internal:50051
```

**Network:**
- `wms-network` (bridge)

**Volumes:**
- `wms_postgres_data`
- `wms_redis_data`
- `go_mod_cache`

---

### Notifications Microservice

**Файл:** `/p/github.com/vondi-global/notifications/docker-compose.yml`

| Service | Image | Container | Порты | ENV Prefix |
|---------|-------|-----------|-------|------------|
| **postgres** | postgres:15-alpine | notifications_postgres | 35437:5432 | VONDINOTIFY_ |
| **redis** | redis:7-alpine | notifications_redis | 36382:6379 | VONDINOTIFY_ |
| **app** | Build: ./Dockerfile | notifications_app | 50054:50054, 8088:8088 | VONDINOTIFY_ |
| **migrate** | migrate/migrate:v4.17.0 | notifications_migrate | - | - |

**Environment Variables:**
```bash
VONDINOTIFY_DB_HOST=postgres
VONDINOTIFY_DB_PORT=5432
VONDINOTIFY_DB_NAME=notifications_db
VONDINOTIFY_DB_USER=notify_user
VONDINOTIFY_DB_PASSWORD=notify_secret

VONDINOTIFY_REDIS_HOST=redis
VONDINOTIFY_REDIS_PORT=6379

VONDINOTIFY_SERVER_GRPC_PORT=50054
VONDINOTIFY_SERVER_HTTP_PORT=8088
```

**Network:**
- `notifications_network` (default)

**Volumes:**
- `postgres_data`
- `redis_data`

**Migrate Service:**
- Profile: `migrate` (run once)
- Command: `-path /migrations -database postgres://... up`

---

## Базы данных

### PostgreSQL Instances (8 databases)

| Container Name | Image | Port | Database | User | Password | Назначение |
|----------------|-------|------|----------|------|----------|------------|
| **hostel_db** | postgres:15 | 5433 | vondi_db | postgres | mX3g1XGhMRUZEX3l | Monolith |
| **listings_postgres** | postgres:15-alpine | 35434 | listings_dev_db | listings_user | listings_secret | Listings MS |
| **delivery-postgres** | postgis:17-3.5-alpine | 35432 | delivery_db | delivery_user | delivery_password | Delivery MS |
| **vondi-dev-postgres-auth** | postgres:15-alpine | 25432 | auth_db | auth_user | auth_secret | Auth MS |
| **vondipayment_postgres** | postgres:16 | 35433 | payment_db | payment_user | payment_secret | Payment MS |
| **wms_postgres** | postgres:15-alpine | 35435 | wms_db | wms_user | wms_secure_pass_2025 | WMS |
| **notifications_postgres** | postgres:15-alpine | 35437 | notifications_db | notify_user | notify_secret | Notifications |
| **zoho-mail-postgres** | pgvector/pgvector:pg15 | 5435 | zoho_mail | zoho | zoho_secure_pass_2025 | Zoho Mail |

**Health Checks:**
```bash
# Standard check (all instances)
test: ["CMD-SHELL", "pg_isready -U ${USER} -d ${DB}"]
interval: 10s
timeout: 5s
retries: 5
```

**Connection Strings:**
```bash
# Monolith
postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db?sslmode=disable

# Listings
postgres://listings_user:listings_secret@localhost:35434/listings_dev_db

# Delivery
postgres://delivery_user:delivery_password@localhost:35432/delivery_db

# Auth
postgres://auth_user:auth_secret@localhost:25432/auth_db

# Payment
postgres://payment_user:payment_secret@localhost:35433/payment_db

# WMS
postgres://wms_user:wms_secure_pass_2025@localhost:35435/wms_db

# Notifications
postgres://notify_user:notify_secret@localhost:35437/notifications_db

# Zoho Mail
postgres://zoho:zoho_secure_pass_2025@localhost:5435/zoho_mail
```

---

### Redis Instances (7 instances)

| Container Name | Image | Port | Password | Назначение |
|----------------|-------|------|----------|------------|
| **hostel_redis** | redis:7-alpine | 6379 | - | Monolith cache |
| **listings_redis** | redis:7-alpine | 36380 | redis_password | Listings cache |
| **vondi-dev-redis-auth** | redis:7-alpine | 26379 | redis_secret | Auth cache |
| **admin-redis** | redis:7-alpine | 6380 | admin_redis_secret | Admin portal |
| **notifications_redis** | redis:7-alpine | 36382 | - | Notifications queue |
| **map4craft_redis** | redis:7-alpine | 37380 | - | External project |
| **wms_redis** | redis:7-alpine | 36381 | - | WMS cache |

**Health Checks:**
```bash
# Without password
test: ["CMD", "redis-cli", "ping"]

# With password
test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${PASSWORD}", "ping"]

interval: 10s
timeout: 5s
retries: 5
```

**Connection:**
```bash
# Monolith
redis://localhost:6379

# Listings
redis://:redis_password@localhost:36380

# Auth
redis://:redis_secret@localhost:26379

# Admin
redis://:admin_redis_secret@localhost:6380

# Notifications
redis://localhost:36382

# WMS
redis://localhost:36381
```

---

## Networks

### Network Types

| Network Name | Driver | Services | Назначение |
|--------------|--------|----------|------------|
| **hostel_network** | bridge | backend, db, opensearch, nginx, minio, mailserver | Production monolith |
| **vondi_network_dev** | bridge | backend, db, redis, opensearch, frontend | Development |
| **listings_network** | bridge | postgres, redis | Listings MS |
| **wms-network** | bridge | wms-postgres, wms-redis, wms-service | WMS |
| **notifications_network** | bridge | postgres, redis, app | Notifications |
| **zoho-mail-network** | bridge | postgres, pgadmin | Zoho Mail |

### Network Aliases

**hostel_network:**
```yaml
mailserver:
  aliases:
    - mailserver
    - mail.svetu.rs

mail-webui:
  aliases:
    - mail-webui
```

---

## Volumes

### Named Volumes

| Volume Name | Type | Device/Path | Size | Назначение |
|-------------|------|-------------|------|------------|
| **db_data** | local | - | ~5GB | Monolith PostgreSQL data |
| **opensearch_data** | local bind | /opt/hostel-data/opensearch | ~10GB | OpenSearch indexes |
| **uploads_data** | local bind | /opt/hostel-data/uploads | ~20GB | User uploads |
| **minio_data** | local bind | /opt/hostel-data/minio | ~50GB | S3 storage |
| **roundcube_data** | local | - | ~100MB | Webmail data |
| **postgres_data_dev** | local | - | ~1GB | Dev monolith DB |
| **redis_data_dev** | local | - | ~500MB | Dev Redis |
| **opensearch-data_dev** | local | - | ~2GB | Dev OpenSearch |
| **vondiauth_postgres_data** | local | - | ~500MB | Auth DB |
| **vondiauth_redis_data** | local | - | ~100MB | Auth Redis |
| **listings_postgres_data** | local | - | ~1GB | Listings DB |
| **listings_redis_data** | local | - | ~200MB | Listings Redis |
| **delivery_postgres_data** | local | - | ~500MB | Delivery DB |
| **wms_postgres_data** | local | - | ~1GB | WMS DB |
| **wms_redis_data** | local | - | ~100MB | WMS Redis |
| **notifications_postgres_data** | local | - | ~200MB | Notifications DB |
| **notifications_redis_data** | local | - | ~100MB | Notifications Redis |
| **zoho_mail_postgres_data** | local | - | ~500MB | Zoho Mail DB |

### Bind Mounts

```yaml
# Production (hostel_network)
./nginx.conf:/etc/nginx/conf.d/default.conf
./frontend/hostel-frontend/build:/usr/share/nginx/html:ro
./certbot/conf:/etc/letsencrypt
./certbot/www:/var/www/certbot

# Development
./backend/uploads:/app/uploads
./backend/migrations:/app/migrations
./backend/fixtures:/app/fixtures
./keys:/app/keys:ro

# WMS
.:/app                                  # Live reload (dev)
go_mod_cache:/go/pkg/mod               # Go modules cache
```

---

## Environment Variables

### Common Prefixes

| Prefix | Service | Example |
|--------|---------|---------|
| **VONDIAUTH_** | Auth Service | VONDIAUTH_DB_HOST |
| **VONDILISTINGS_** | Listings | VONDILISTINGS_GRPC_PORT |
| **VONDINOTIFY_** | Notifications | VONDINOTIFY_REDIS_HOST |
| **VONDIPAYMENT_** | Payment | VONDIPAYMENT_DB_NAME |
| **WMS_** | WMS | WMS_DB_USER |
| **NEXT_PUBLIC_** | Frontend (public) | NEXT_PUBLIC_API_URL |

### Critical Variables

**Backend (Monolith):**
```bash
APP_MODE=production
ENV_FILE=.env
WS_ENABLED=true
DATABASE_URL=postgres://...
REDIS_URL=redis://...
OPENSEARCH_URL=http://opensearch:9200
MINIO_ENDPOINT=minio:9000
FILE_STORAGE_PROVIDER=minio
JWT_SECRET=${JWT_SECRET}
GOOGLE_CLIENT_ID=${GOOGLE_CLIENT_ID}
CLAUDE_API_KEY=${CLAUDE_API_KEY}
FIREBASE_PROJECT_ID=${FIREBASE_PROJECT_ID}
```

**Frontend:**
```bash
NODE_ENV=production
INTERNAL_API_URL=http://backend:3000
NEXT_PUBLIC_API_URL=https://dev.vondi.rs
NEXT_PUBLIC_MINIO_URL=https://s3.vondi.rs
NEXT_PUBLIC_WEBSOCKET_URL=wss://dev.vondi.rs
NEXT_PUBLIC_MAPBOX_TOKEN=${MAPBOX_TOKEN}
```

---

## Health Checks

### PostgreSQL

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${USER} -d ${DB}"]
  interval: 10s
  timeout: 5s
  retries: 5
```

### Redis

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 5s
  retries: 5
```

### Backend

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000"]
  interval: 10s
  timeout: 5s
  retries: 3
  start_period: 15s
```

### MinIO

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
  interval: 30s
  timeout: 10s
  retries: 3
```

### Nginx

```yaml
healthcheck:
  test: ["CMD", "wget", "--spider", "--quiet", "http://localhost/"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 15s
```

---

## Порты (полная карта)

### Microservices

| Service | HTTP | gRPC | Prometheus | pprof | Other |
|---------|------|------|------------|-------|-------|
| **Monolith Backend** | 3000 | - | - | - | - |
| **Frontend** | 3001 | - | - | - | - |
| **Auth** | 28086 | 20053 | 29090 | - | - |
| **Listings** | 8086 | 50053 | - | 6060 | - |
| **Delivery** | - | 50052 | 9091 | - | - |
| **Payment** | 8089 | 50054 | - | - | - |
| **WMS** | 8085 | 50055 | - | - | - |
| **Notifications** | 8088 | 50054 | - | - | - |
| **Admin Portal** | 3006 | - | - | - | - |

### Databases

| Database | PostgreSQL Port | Redis Port |
|----------|----------------|------------|
| **Monolith** | 5433 | 6379 |
| **Listings** | 35434 | 36380 |
| **Delivery** | 35432 | - |
| **Auth** | 25432 | 26379 |
| **Payment** | 35433 | - |
| **WMS** | 35435 | 36381 |
| **Notifications** | 35437 | 36382 |
| **Admin Portal** | 5434 | 6380 |
| **Zoho Mail** | 5435 | - |

### Infrastructure

| Service | Port(s) | Protocol |
|---------|---------|----------|
| **OpenSearch** | 9200 | HTTP |
| **OpenSearch Dashboards** | 5601 | HTTP |
| **MinIO** | 9000, 9001 | HTTP |
| **Nginx** | 80, 443 | HTTP/HTTPS |
| **Mailserver** | 25, 587, 465, 143, 993, 110, 995 | SMTP/IMAP/POP3 |
| **PgAdmin** | 5436 | HTTP |

---

## Harbor Registry

**Registry URL:** `harbor.svetu.rs`

### Namespaces

| Namespace | Назначение | Примеры образов |
|-----------|------------|-----------------|
| **svetu/backend/** | Backend services | api:latest |
| **svetu/db/** | Databases | postgres:15 |
| **svetu/opensearch/** | Search | opensearch:2.11.0 |
| **svetu/mail/** | Mail services | server:latest, webui:latest |
| **svetu/tools/** | Utilities | certbot:latest, migrate:latest |
| **svetu/nginx/** | Web servers | nginx:latest |
| **svetu/minio/** | S3 storage | minio:RELEASE.2023-09-30, mc:latest |

### Authentication

```bash
# Login to Harbor
docker login harbor.svetu.rs
# Username: admin
# Password: Harbor12345

# Pull image
docker pull harbor.svetu.rs/svetu/backend/api:latest

# Push image
docker tag my-image:latest harbor.svetu.rs/svetu/backend/api:v1.0.0
docker push harbor.svetu.rs/svetu/backend/api:v1.0.0
```

### CI/CD Integration

**GitHub Actions автоматически пушит образы в Harbor:**
```yaml
# .github/workflows/build.yml
- name: Build and push to Harbor
  run: |
    docker build -t harbor.svetu.rs/svetu/backend/api:${{ github.sha }} .
    docker push harbor.svetu.rs/svetu/backend/api:${{ github.sha }}
```

---

## Команды запуска

### Production (Single Stack)

```bash
# Start all services
docker compose -f docker-compose.prod.yml up -d

# Stop all services
docker compose -f docker-compose.prod.yml down

# View logs
docker compose -f docker-compose.prod.yml logs -f backend

# Restart specific service
docker compose -f docker-compose.prod.yml restart backend
```

### Development (Monolith)

```bash
# Start infrastructure
docker compose -f docker-compose.dev.yml up -d db redis opensearch

# Start all services
docker compose -f docker-compose.dev.yml up -d

# Rebuild and start
docker compose -f docker-compose.dev.yml up -d --build

# Stop all
docker compose -f docker-compose.dev.yml down

# Remove volumes (⚠️ data loss)
docker compose -f docker-compose.dev.yml down -v
```

### Microservices (Individual)

**Auth Service:**
```bash
cd /p/github.com/vondi-global/auth
docker compose up -d postgres redis
```

**Listings Service:**
```bash
cd /p/github.com/vondi-global/listings
docker compose up -d
```

**Delivery Service:**
```bash
cd /p/github.com/vondi-global/delivery
docker compose up -d
```

**Payment Service:**
```bash
cd /p/github.com/vondi-global/payment
docker compose up -d
```

**WMS:**
```bash
cd /p/github.com/vondi-global/warehouse
docker compose up -d
```

**Notifications:**
```bash
cd /p/github.com/vondi-global/notifications
docker compose up -d

# Run migrations
docker compose run --rm migrate
```

### Zoho Mail Integration

```bash
cd /p/github.com/vondi-global/zoho-mail-integration
docker compose up -d

# With PgAdmin (optional)
docker compose --profile admin up -d
```

---

## Monitoring

### Prometheus + Grafana (Listings)

**Файл:** `/p/github.com/vondi-global/listings/deployment/prometheus/docker-compose.yml`

| Service | Image | Port | Назначение |
|---------|-------|------|------------|
| **prometheus** | prom/prometheus:latest | 9090 | Metrics collection |
| **grafana** | grafana/grafana:latest | 3002 | Dashboards |
| **alertmanager** | prom/alertmanager:latest | 9093 | Alerting |

**Network:**
- `monitoring` (bridge)
- `listings_network` (external)

**Volumes:**
- `prometheus_data`
- `grafana_data`
- `alertmanager_data`

**Dashboards:**
- Listings Service Overview
- gRPC Performance
- PostgreSQL Metrics
- Redis Cache Stats

**Start Monitoring:**
```bash
cd /p/github.com/vondi-global/listings/deployment/prometheus
docker compose up -d

# Access Grafana
open http://localhost:3002
# Default: admin/admin
```

### Jaeger (Delivery Tracing)

**Файл:** `/p/github.com/vondi-global/delivery/monitoring/docker-compose.jaeger.yml`

| Service | Image | Port | Назначение |
|---------|-------|------|------------|
| **jaeger** | jaegertracing/all-in-one:latest | 16686 (UI), 6831 (UDP), 14268 (HTTP) | Distributed tracing |

**Start Tracing:**
```bash
cd /p/github.com/vondi-global/delivery/monitoring
docker compose -f docker-compose.jaeger.yml up -d

# Access UI
open http://localhost:16686
```

---

## Troubleshooting

### Container не запускается

```bash
# Проверить логи
docker logs <container_name>

# Проверить health status
docker ps --format "table {{.Names}}\t{{.Status}}"

# Удалить и пересоздать
docker compose down
docker compose up -d --force-recreate
```

### Port already in use

```bash
# Найти процесс на порту
netstat -tlnp | grep :3000

# Убить процесс
kill -9 <PID>

# Или использовать скрипт
/home/dim/.local/bin/kill-port-3000.sh
```

### Volume permissions

```bash
# Проверить владельца
ls -la /opt/hostel-data/

# Исправить права
sudo chown -R 101:101 /opt/hostel-data/uploads
sudo chown -R 1000:1000 /opt/hostel-data/minio
```

### Network issues

```bash
# Проверить networks
docker network ls

# Пересоздать network
docker network rm hostel_network
docker compose up -d
```

---

## Backup & Restore

### PostgreSQL Backup

```bash
# Monolith
docker exec hostel_db pg_dump -U postgres vondi_db | gzip > vondi_db_$(date +%Y%m%d).sql.gz

# Listings
docker exec listings_postgres pg_dump -U listings_user listings_dev_db | gzip > listings_$(date +%Y%m%d).sql.gz

# Auth
docker exec vondi-dev-postgres-auth pg_dump -U auth_user auth_db | gzip > auth_$(date +%Y%m%d).sql.gz
```

### PostgreSQL Restore

```bash
# Monolith
gunzip -c vondi_db_20251221.sql.gz | docker exec -i hostel_db psql -U postgres vondi_db

# Listings
gunzip -c listings_20251221.sql.gz | docker exec -i listings_postgres psql -U listings_user listings_dev_db
```

### Volume Backup

```bash
# Backup volume to tar
docker run --rm -v db_data:/data -v $(pwd):/backup alpine tar czf /backup/db_data.tar.gz /data

# Restore volume from tar
docker run --rm -v db_data:/data -v $(pwd):/backup alpine tar xzf /backup/db_data.tar.gz -C /
```

---

## Performance Tuning

### PostgreSQL

```yaml
# docker-compose override
services:
  db:
    command:
      - "postgres"
      - "-c"
      - "max_connections=200"
      - "-c"
      - "shared_buffers=256MB"
      - "-c"
      - "effective_cache_size=1GB"
      - "-c"
      - "work_mem=16MB"
```

### Redis

```yaml
services:
  redis:
    command: >
      redis-server
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
```

### OpenSearch

```yaml
services:
  opensearch:
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms2048m -Xmx2048m"
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
```

---

## Security

### Secrets Management

**НЕ коммить в git:**
- `.env` файлы
- Пароли в `docker-compose.yml`
- SSL сертификаты

**Использовать:**
```bash
# Environment variables
export DATABASE_PASS=$(cat /run/secrets/db_password)

# Docker secrets (Swarm mode)
docker secret create db_password db_password.txt
```

### Network Isolation

```yaml
# Внутренние сервисы без expose портов
services:
  db:
    # НЕТ ports: - только внутри network
    networks:
      - internal_network

networks:
  internal_network:
    internal: true  # Изоляция от внешней сети
```

---

## Changelog

### 2.0.0 - 2025-12-21
- Полный пересмотр и расширение документации
- Добавлены все микросервисы (Auth, Listings, Delivery, Payment, WMS, Notifications)
- Полная карта портов и баз данных (8 PostgreSQL, 7 Redis)
- Документированы environment variables с префиксами
- Добавлены примеры команд для каждого сервиса
- Расширен раздел monitoring (Prometheus, Grafana, Jaeger)
- Добавлены разделы Backup, Performance Tuning, Security
- Задокументирован Harbor registry с примерами

### 1.1.0 - 2025-11-24
- Начальная версия паспорта

---

## См. также

- [SYSTEM_PASSPORT.md](../../SYSTEM_PASSPORT.md) - Общая архитектура
- [network.md](./network.md) - Сетевая топология
- [storage.md](./storage.md) - MinIO/S3 хранилище
- [../services/monolith.md](../services/monolith.md) - Монолитный backend
- [../services/auth.md](../services/auth.md) - Auth микросервис
- [../services/listings.md](../services/listings.md) - Listings микросервис
- [../services/delivery.md](../services/delivery.md) - Delivery микросервис
