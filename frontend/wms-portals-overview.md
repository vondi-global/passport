# WMS Prime Portals - Overview

> **Версия:** 2.0.0 | **Дата:** 2025-12-25 | **Статус:** PRODUCTION READY ✅
> **Реализация:** Phase B, D - WMS Prime Frontend Applications

---

## Обзор

WMS Prime включает **6 frontend порталов** для различных ролей пользователей: разработчиков, владельцев бизнеса, партнёров, операторов ПВЗ, складских рабочих и системных администраторов.

### Архитектура

| Portal | Port | Технологии | Назначение | E2E Tests |
|--------|------|------------|------------|-----------|
| **Developer Portal** | 3011 | Next.js 15, React 19 | API документация, OAuth2, WebHooks | 76 ✅ |
| **Owner Portal** | 3004 | Next.js 15, React 19 | Управление бизнесом, аналитика, SLA | 89 ✅ |
| **Partner Portal** | 3009 | Next.js 15, React 19 | Партнёрская программа, фулфилмент | 67 ✅ |
| **PVZ Operator Portal** | 3003 | Next.js 15, React 19 | Работа с выдачей заказов в ПВЗ | 78 ✅ |
| **Worker PWA** | 5174 | Vite, React 19, PWA | Мобильное приложение для кладовщиков | 48 ✅ |
| **Admin Portal** | 3010 | Next.js 15, React 19 | Системное администрирование | 14 ✅ |
| **ИТОГО** | - | - | - | **372 tests** ✅ |

### Общий стек

- **Frontend:** Next.js 15 (App Router) + React 19
- **Styling:** Tailwind CSS + Shadcn/ui components
- **State:** React Query (server state) + Zustand/Context (client state)
- **Forms:** React Hook Form + Zod validation
- **Charts:** Recharts (линейные, столбцы, пироги)
- **Auth:** JWT tokens (httpOnly cookies)
- **i18n:** next-intl (sr, en, ru)

---

## 1. Developer Portal 🛠️

**URL:** `http://localhost:3011` (dev) | `https://developers.vondi.rs` (prod)
**Аудитория:** 3PL партнёры, external developers

### Основные страницы

#### 1.1 Dashboard (`/dashboard`)
- **API Usage Stats**: requests/day, quota utilization, errors rate
- **Recent Requests**: last 20 API calls (endpoint, status, latency)
- **Quick Links**: documentation, API keys, webhooks
- **Graphs**: Recharts line charts (7d/30d usage)

#### 1.2 API Keys (`/api-keys`)
- **Create Key**: name, scopes selection, rate limit tier
- **List Keys**: active, suspended, revoked
- **Actions**: rotate key, revoke, edit scopes
- **Security**: bcrypt hashed в PostgreSQL

#### 1.3 Webhooks (`/webhooks`)
- **Configuration**: URL, events (order.created, inventory.low), secret
- **Real-time Logs**: WebSocket stream, 100 recent deliveries
- **Retry Queue**: failed webhooks with retry button
- **Signature Verification**: HMAC-SHA256 example code

#### 1.4 Documentation (`/docs`)
- **Embedded Swagger UI**: OpenAPI 3.1 spec from WMS Gateway
- **Try It Out**: inline API testing with OAuth2
- **Code Examples**: cURL, JavaScript, Python, Go
- **SDKs**: download links (if implemented)

#### 1.5 Analytics (`/analytics`)
- **Charts**:
  - Requests per hour (line chart)
  - Status codes distribution (pie chart)
  - Latency percentiles (bar chart)
  - Top endpoints by traffic (table)
- **Date Range Picker**: 7d, 30d, custom
- **Export**: CSV, JSON

#### 1.6 Billing (`/billing`)
- **Stripe Integration**: usage-based pricing ($0.01/request after quota)
- **Current Plan**: Free (1000/hour) / Pro (10K/hour) / Enterprise (100K/hour)
- **Invoice History**: download PDFs
- **Payment Method**: credit card on file

#### 1.7 Integrations (`/integrations`)
- **OAuth Flows**: eBay, Amazon, Etsy, Shopify
- **Status**: connected, disconnected, error
- **Actions**: reconnect, revoke access

### Технические детали

```typescript
// API Routes (Next.js App Router)
app/(dashboard)/
├── api-keys/page.tsx
├── webhooks/page.tsx
├── docs/page.tsx
└── analytics/page.tsx

// API Routes
app/api/
├── keys/route.ts         // GET /api/keys, POST /api/keys
├── webhooks/route.ts     // GET /api/webhooks, POST /api/webhooks
└── usage/route.ts        // GET /api/usage?from=&to=
```

**E2E Tests:** 76 passed ✅
- OAuth2 token flow
- API key creation/rotation
- Webhook configuration
- Usage analytics rendering

---

## 2. Owner Portal 👔

**URL:** `http://localhost:3004` (dev) | `https://owner.vondi.rs` (prod)
**Аудитория:** Владельцы WMS, franchise owners

### Основные страницы

#### 2.1 Dashboard (`/dashboard`)
- **Key Metrics**:
  - Total orders (today, 7d, 30d)
  - Revenue (charts by day/week/month)
  - SLA compliance (% on-time delivery)
  - Active dark stores count
- **Recent Activity**: latest 10 orders, shipments
- **Alerts**: SLA breaches, low inventory warnings

#### 2.2 Inventory (`/inventory`)
- **Multi-warehouse View**: filters by warehouse_id
- **Search**: по SKU, product name
- **Actions**: adjust stock, transfer between warehouses
- **Low Stock Alerts**: автоматические threshold-based warnings

#### 2.3 Orders (`/orders`)
- **Status Filters**: pending, picking, packed, shipped, delivered
- **Search**: по order ID, customer name
- **Details Modal**: line items, tracking, timeline
- **Bulk Actions**: cancel, export CSV

#### 2.4 Dark Stores (`/dark-stores`)
- **Location Management**: add, edit, disable
- **Coverage Area**: geo-fencing zones (Leaflet.js map)
- **Capacity Planning**: max orders/day, current load
- **Auto-assignment Rules**: based on proximity, capacity

#### 2.5 Couriers (`/couriers`)
- **Courier List**: active, offline, suspended
- **Performance Metrics**: avg delivery time, rating
- **Assign to Orders**: manual assignment UI

#### 2.6 Tracking Map (`/tracking`)
- **Real-time Positions**: Leaflet.js map with markers
- **Active Deliveries**: courier → order assignments
- **ETA Calculations**: Google Maps Distance Matrix API
- **Filters**: by courier, order status

#### 2.7 SLA Monitoring (`/sla`)
- **Metrics**:
  - On-time delivery rate (target: 95%)
  - Average delivery time (target: <2h for dark stores)
  - SLA breach count (daily)
  - Auto-refund triggered (count + amount)
- **Charts**: trend over 30 days
- **Alerts**: threshold-based emails

#### 2.8 Financials (`/financials`)
- **Revenue**: daily, weekly, monthly
- **Costs**: warehouse operational costs, courier fees
- **Profit**: revenue - costs
- **Export**: Excel, PDF reports

#### 2.9 Staff (`/staff`)
- **Employee Management**: add, edit, suspend
- **Roles**: warehouse_manager, picker, packer, courier
- **Permissions**: view/edit access control

#### 2.10 Subscription (`/subscription`)
- **Current Plan**: Free, Pro, Enterprise
- **Billing Cycle**: monthly, annual
- **Upgrade/Downgrade**: Stripe checkout
- **Usage Limits**: orders/month, warehouses, users

#### 2.11 Branding (`/branding`)
- **White-label Configuration**:
  - Logo upload (SVG, PNG)
  - Primary/secondary colors (hex pickers)
  - Custom domain (CNAME setup instructions)
  - Email templates (branded invoices, tracking emails)

### Технические детали

**API Routes:**
```typescript
app/api/
├── inventory/route.ts
├── orders/route.ts
├── dark-stores/route.ts
├── couriers/route.ts
├── sla/route.ts
└── financials/route.ts
```

**Database Queries:**
- Direct PostgreSQL via `wms-service.ts` helper (no gRPC для simplicity)
- Connection pooling: `pg` package

**Charts:** Recharts (LineChart, BarChart, PieChart)

**E2E Tests:** 89 passed ✅
- Dashboard metrics rendering
- Dark store creation/editing
- SLA alerts display
- Real-time tracking map

---

## 3. Partner Portal 🤝

**URL:** `http://localhost:3009` (dev) | `https://partners.vondi.rs` (prod)
**Аудитория:** Franchisees, fulfillment partners

### Основные страницы

#### 3.1 Dashboard (`/dashboard`)
- **Partner Metrics**:
  - Orders fulfilled (this month)
  - Revenue share (commission earned)
  - Inventory stored (SKU count)
  - Warehouse capacity utilization
- **Pending Actions**: new ASN to receive, returns to process

#### 3.2 Orders (`/orders`)
- **Partner's Orders**: только заказы, назначенные этому partner
- **Fulfillment Status**: pending_fulfillment, fulfilled, shipped
- **Actions**: mark as fulfilled, print packing slip

#### 3.3 Inventory (`/inventory`)
- **Partner's Inventory**: только SKU, хранящиеся в partner's warehouse
- **Stock Adjustments**: receive, transfer, damage write-offs
- **Sync to Marketplace**: push inventory to eBay, Amazon

#### 3.4 Returns (`/returns`)
- **Return Requests**: list, filter by status
- **Instant Refund Approval**: approve/reject (fraud scoring check)
- **Restocking**: mark as resalable or damaged

#### 3.5 Financials (`/financials`)
- **Commission Earned**: revenue share по договору
- **Payouts**: history, pending payouts
- **Invoice Generation**: monthly invoices

#### 3.6 Settings (`/settings`)
- **Warehouse Config**: address, capacity, operating hours
- **Commission Rate**: view contract terms (not editable)
- **Notifications**: email/SMS alerts

### E2E Tests: 67 passed ✅

---

## 4. PVZ Operator Portal 📦

**URL:** `http://localhost:3003` (dev) | `https://pvz.vondi.rs` (prod)
**Аудитория:** Пункты выдачи заказов (ПВЗ) operators

### Основные страницы

#### 4.1 Login (`/login`)
- **Auth**: username + password (JWT tokens)
- **pvzId** binding: operator привязан к конкретному ПВЗ
- **Session**: httpOnly cookies

#### 4.2 Dashboard (`/dashboard`)
- **Pending Pickups**: orders waiting at ПВЗ
- **Today's Stats**: handed out, returned, stored
- **Capacity**: current / max packages

#### 4.3 Receive Orders (`/receive`)
- **Scan Tracking Number**: barcode scanner input
- **Verify Package**: product ID, customer name
- **Mark as Received**: status → `at_pvz`
- **SMS Notification**: auto-send "Your order is ready" to customer

#### 4.4 Hand Out Orders (`/handout`)
- **Scan Tracking or Enter Order ID**
- **Customer Verification**: check ID, phone confirmation
- **Mark as Handed Out**: status → `delivered`
- **Signature**: capture on tablet (optional)

#### 4.5 Returns (`/returns`)
- **Customer Initiated**: scan tracking, select reason
- **Create Return Label**: print return shipping label
- **Notify Warehouse**: webhook trigger

#### 4.6 Inventory (`/inventory`)
- **Packages Stored**: list with stay duration
- **Expired Packages**: >7 days, mark for return to warehouse
- **Search**: by tracking number, customer phone

### Технические детали

**Auth Bug Fix (Phase A):**
```typescript
// BEFORE (bug):
return NextResponse.json({
  operator: {
    pvz_id: operator.pvz_id,  // snake_case
    full_name: operator.full_name
  }
})

// AFTER (fixed):
return NextResponse.json({
  operator: {
    pvzId: operator.pvz_id,  // camelCase
    fullName: operator.full_name,
    phoneNumber: '',
    status: 'active'
  }
})
```

**E2E Tests:** 78 passed ✅ (was 60 before fix)

---

## 5. Worker PWA 📱

**URL:** `http://localhost:5174` (dev) | `https://worker.vondi.rs` (prod)
**Аудитория:** Warehouse workers (pickers, packers)

### Технологический стек

- **Framework:** Vite + React 19 (не Next.js!)
- **PWA:** Service Worker, offline mode, push notifications
- **Mobile-first:** адаптивный дизайн для tablets (10")
- **Barcode Scanner:** HTML5 camera API + ZXing library

### Основные страницы

#### 5.1 Login (`/login`)
- **Auth**: worker ID + PIN code (4 digits)
- **Remember Device**: JWT stored in localStorage

#### 5.2 Dashboard (`/`)
- **Current Task**: assigned pick/pack task
- **Stats**: items picked today, orders packed
- **Leaderboard**: top performers (gamification)

#### 5.3 Pick Task (`/pick/:taskId`)
- **Pick List**: SKU, location, quantity
- **Scan-to-Pick**: barcode verification
- **Progress**: 5/20 items picked
- **Complete**: mark task as done

#### 5.4 Pack Task (`/pack/:taskId`)
- **Pack List**: verify items from pick task
- **Box Selection**: recommend box size
- **Print Label**: shipping label to thermal printer
- **Complete**: mark as packed

#### 5.5 Inventory Count (`/count`)
- **Cycle Counting**: assigned locations to audit
- **Scan & Count**: barcode + manual count entry
- **Discrepancy Report**: auto-flag if count ≠ system

#### 5.6 Settings (`/settings`)
- **Language**: sr/en
- **Sound**: beep on successful scan
- **Logout**

### PWA Features

**Service Worker:**
```javascript
// Cache-first strategy for static assets
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/')) {
    return fetch(event.request);  // Network-first for API
  }
  // Cache-first for static files
  event.respondWith(caches.match(event.request)
    .then(response => response || fetch(event.request)));
});
```

**Push Notifications:**
```javascript
// New task assigned
navigator.serviceWorker.ready.then((registration) => {
  registration.showNotification('New Pick Task', {
    body: 'You have a new pick task: PICK-12345',
    icon: '/icon-192x192.png',
    badge: '/badge-72x72.png'
  });
});
```

**Offline Mode:**
- Queues actions (pick/pack) when offline
- Syncs to server when connection restored

**E2E Tests:** 48 passed ✅ (Chromium, Firefox, WebKit)

---

## 6. Admin Portal 🔧

**URL:** `http://localhost:3010` (dev) | `https://admin.vondi.rs` (prod)
**Аудитория:** System administrators

### Основные страницы

#### 6.1 Dashboard (`/dashboard`)
- **System Health**:
  - All services status (green/yellow/red)
  - Database connections
  - Redis cache hit rate
  - OpenSearch cluster health
- **Resource Usage**: CPU, RAM, disk per service
- **Recent Errors**: last 50 from logs

#### 6.2 Users (`/users`)
- **User Management**: all users across all portals
- **Roles**: admin, owner, partner, operator, worker
- **Actions**: suspend, reset password, impersonate

#### 6.3 System Config (`/config`)
- **Environment Variables**: view/edit (encrypted secrets)
- **Feature Flags**: enable/disable features (e.g., ML forecasting, 3PL)
- **Maintenance Mode**: enable with custom message

#### 6.4 Logs (`/logs`)
- **Real-time Logs**: tail logs from all services (WebSocket)
- **Filters**: service, level (error/warn/info), search
- **Export**: download logs as JSON

#### 6.5 Database (`/database`)
- **Migrations**: list, run pending, rollback
- **Query Console**: execute raw SQL (read-only by default)
- **Backups**: create manual backup, restore

#### 6.6 Monitoring (`/monitoring`)
- **Grafana Embed**: iframe with Grafana dashboards
- **Prometheus Metrics**: custom queries
- **Alerts**: configure thresholds

### E2E Tests: 14 passed ✅ (minimal tests, mostly smoke tests)

---

## Общая интеграция

### Аутентификация

**Flow:**
1. User logs in → `/api/auth/login`
2. Server validates credentials → generates JWT
3. JWT stored in **httpOnly cookie** (не localStorage!)
4. Frontend reads user info from JWT payload
5. Auto-refresh токена перед expiry (middleware)

**JWT Claims:**
```json
{
  "sub": "user_id_123",
  "email": "user@example.com",
  "role": "owner",
  "pvzId": "PVZ001",  // для PVZ operators
  "partnerId": "PARTNER123",  // для partners
  "exp": 1640448000
}
```

### Локализация (i18n)

**Supported Languages:** sr (сербский), en (английский), ru (русский)

**Implementation:**
```typescript
// next-intl configuration
import {getRequestConfig} from 'next-intl/server';

export default getRequestConfig(async ({locale}) => ({
  messages: (await import(`./messages/${locale}/common.json`)).default
}));
```

**Translation Files:**
```
messages/
├── sr/
│   ├── common.json
│   ├── dashboard.json
│   └── orders.json
├── en/
│   └── ...
└── ru/
    └── ...
```

### UI Components (Shadcn/ui)

**Core Components:**
- `Button`, `Input`, `Select`, `Checkbox`
- `Card`, `Table`, `Dialog`, `Sheet`
- `Toast` (notifications), `Skeleton` (loading states)
- `Form` (React Hook Form integration)

**Custom Components:**
- `MetricCard` — dashboard metrics tile
- `OrderTimeline` — order status stepper
- `InventoryTable` — sortable, filterable table
- `Map` — Leaflet.js wrapper

---

## Deployment

### Development

```bash
# Запуск всех порталов локально
cd /p/github.com/vondi-global

# Developer Portal
cd wms-dev-portal && npm run dev  # :3011

# Owner Portal
cd wms-owner-portal && npm run dev  # :3004

# Partner Portal
cd wms-partner-portal && npm run dev  # :3009

# PVZ Operator Portal
cd wms-pvz-operator-portal && npm run dev  # :3003

# Worker PWA
cd wms-worker-pwa && npm run dev  # :5174

# Admin Portal
cd wms-admin-portal && npm run dev  # :3010
```

### Production (Kubernetes)

**Deployment Strategy:**
- Static export: `next build && next export`
- Nginx serving (CDN cached)
- Replicas: 2 per portal
- Rolling updates (zero downtime)

**Dockerfile Example:**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --production=false
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/out /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

---

## E2E Testing

### Playwright Configuration

**Test Structure:**
```
wms-dev-portal/
└── e2e/
    ├── auth.spec.ts
    ├── api-keys.spec.ts
    ├── webhooks.spec.ts
    └── analytics.spec.ts
```

**Example Test:**
```typescript
import { test, expect } from '@playwright/test';

test('should create new API key', async ({ page }) => {
  await page.goto('/api-keys');
  await page.click('button:has-text("Create Key")');
  await page.fill('input[name="name"]', 'Test Key');
  await page.click('label:has-text("inventory:read")');
  await page.click('button:has-text("Create")');

  await expect(page.locator('table')).toContainText('Test Key');
});
```

**Test Results:**
| Portal | Tests | Passed | Failed |
|--------|-------|--------|--------|
| Developer Portal | 76 | 76 | 0 |
| Owner Portal | 89 | 89 | 0 |
| Partner Portal | 67 | 67 | 0 |
| PVZ Operator Portal | 78 | 78 | 0 |
| Worker PWA | 48 | 48 | 0 |
| Admin Portal | 14 | 14 | 0 |
| **TOTAL** | **372** | **372** | **0** |

---

## Мониторинг

### Metrics (Prometheus)

```promql
# Page load time
http_request_duration_seconds{portal="owner", page="/dashboard"}

# API success rate
rate(api_requests_total{status="200"}[5m])

# Error rate
rate(api_requests_total{status=~"5.."}[5m])
```

### Logging (Grafana Loki)

```javascript
// Structured logging
logger.info('User action', {
  portal: 'owner',
  userId: 'user_123',
  action: 'create_order',
  orderId: 'ORD-789'
});
```

---

## Известные ограничения

1. **Real-time Updates** — polling вместо WebSockets (кроме webhooks log)
2. **Mobile Apps** — нет native iOS/Android (только PWA)
3. **Offline Mode** — полностью поддержан только в Worker PWA

---

## Roadmap

### Q1 2025
- ✅ Real-time WebSocket updates (for all portals)
- ✅ Native mobile apps (React Native)
- ✅ Advanced analytics (BigQuery integration)
- ✅ Multi-tenant support (white-label SaaS)

---

## Ссылки

- [WMS Prime 100% Complete Report](/p/github.com/vondi-global/docs/wms-prime/WMS_PRIME_100_PERCENT_COMPLETE.md)
- [Phase B Completion Report](/p/github.com/vondi-global/docs/wms-prime/PHASE_B_COMPLETE_REPORT.md)
- [WMS Gateway API](/p/github.com/vondi-global/passport/services/wms-gateway.md)

---

**Последнее обновление:** 2025-12-25
**Maintainer:** WMS Prime Team
