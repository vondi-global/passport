# Инфраструктура и конфигурация

> Часть паспорта [vondi-ai-landing](./README.md)

## Сетевая конфигурация

| Параметр | Значение |
|----------|----------|
| **Production URL** | https://solutions.vondi.rs |
| **Внутренний порт** | 3008 |
| **Протокол** | HTTPS (Let's Encrypt) |
| **Reverse proxy** | Nginx |

---

## Environment Variables

### Обязательные

```bash
# Anthropic API для AI демо
ANTHROPIC_API_KEY=sk-ant-api03-...

# URL сайта
NEXT_PUBLIC_SITE_URL=https://solutions.vondi.rs

# Порт приложения
PORT=3008
```

### Опциональные (Zoho Mail)

```bash
# OAuth2 credentials для отправки email
ZOHO_CLIENT_ID=1000.WYUN6SHOC65CCO5VOL6CP307TG3JWL
ZOHO_CLIENT_SECRET=<secret>
ZOHO_REFRESH_TOKEN=<refresh_token>
ZOHO_ACCOUNT_ID=<account_id>

# Домены Zoho (по умолчанию EU)
ZOHO_ACCOUNTS_DOMAIN=accounts.zoho.eu
ZOHO_MAIL_DOMAIN=mail.zoho.eu
ZOHO_FROM_EMAIL=mail@vondi.rs

# Email для нотификаций о chat сессиях
CHAT_NOTIFY_EMAIL=mail@vondi.rs
```

### Rate Limiting

```bash
RATE_LIMIT_PER_MINUTE=10
RATE_LIMIT_PER_HOUR=50
RATE_LIMIT_PER_DAY=100
```

---

## package.json

### Dependencies

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.71.2",
    "@hookform/resolvers": "^5.2.2",
    "framer-motion": "^12.23.26",
    "lucide-react": "^0.560.0",
    "next": "16.0.8",
    "next-intl": "^4.5.8",
    "react": "19.2.1",
    "react-dom": "19.2.1",
    "react-hook-form": "^7.68.0",
    "zod": "^4.1.13"
  },
  "devDependencies": {
    "@playwright/test": "^1.57.0",
    "@tailwindcss/postcss": "^4",
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "eslint": "^9",
    "eslint-config-next": "16.0.8",
    "typescript": "^5"
  }
}
```

### Scripts

```json
{
  "scripts": {
    "dev": "next dev -p 3008",
    "build": "next build",
    "start": "next start -p 3008",
    "lint": "next lint",
    "test": "playwright test",
    "test:e2e": "playwright test"
  }
}
```

---

## Next.js Configuration

```typescript
// next.config.ts
import createNextIntlPlugin from 'next-intl/plugin';

const withNextIntl = createNextIntlPlugin();

const nextConfig = {
  output: 'standalone',

  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 's3.vondi.rs',
        pathname: '/**',
      },
    ],
  },
};

export default withNextIntl(nextConfig);
```

**Ключевые настройки:**
- `output: 'standalone'` - для Docker (минимальный образ)
- `images.remotePatterns` - разрешение изображений с S3
- `withNextIntl` - плагин для интернационализации

---

## TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

**Path aliases:** `@/*` → `./src/*`

---

## Tailwind CSS Configuration

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss';

const config: Config = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          900: '#1a1a2e',  // Тёмный фон
          800: '#16213e',
          700: '#2d2d44',
          600: '#3d3d5c',
          500: '#4a4a6a',
        },
        accent: {
          600: '#3d5af1',
          500: '#4a6cf7',  // Основной акцент (синий)
          400: '#6b8cff',
          300: '#94aeff',
        },
        success: {
          500: '#10b981',
          400: '#34d399',
        },
        warning: {
          500: '#f59e0b',
          400: '#fbbf24',
        },
      },
      fontFamily: {
        sans: ['Inter', 'SF Pro Display', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'Fira Code', 'monospace'],
      },
    },
  },
  plugins: [],
};

export default config;
```

### Цветовая схема

| Цвет | Hex | Использование |
|------|-----|---------------|
| Primary 900 | `#1a1a2e` | Тёмный фон секций |
| Primary 700 | `#2d2d44` | Вторичный фон |
| Accent 500 | `#4a6cf7` | CTA кнопки, ссылки |
| Accent 400 | `#6b8cff` | Hover состояния |
| Success 500 | `#10b981` | Иконки галочек, успех |
| Warning 500 | `#f59e0b` | Предупреждения |

---

## Docker Configuration

### Dockerfile (Production)

```dockerfile
# Base stage
FROM node:20-alpine AS base
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Dependencies stage
FROM base AS deps
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile --production=false

# Builder stage
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN yarn build

# Runner stage
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3008

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3008

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3008/api/health || exit 1

CMD ["node", "server.js"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  vondi-ai-landing:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    container_name: vondi-ai-landing
    ports:
      - "3008:3008"
    environment:
      - NODE_ENV=production
      - PORT=3008
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - NEXT_PUBLIC_SITE_URL=${NEXT_PUBLIC_SITE_URL}
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3008"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## Nginx Configuration

```nginx
# /etc/nginx/sites-available/solutions.vondi.rs.conf

upstream vondi_ai_landing {
    server 127.0.0.1:3008;
    keepalive 64;
}

server {
    listen 80;
    server_name solutions.vondi.rs;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name solutions.vondi.rs;

    # SSL certificates (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/solutions.vondi.rs/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/solutions.vondi.rs/privkey.pem;

    # SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript;

    # Static files caching
    location /_next/static {
        proxy_pass http://vondi_ai_landing;
        proxy_cache_valid 200 365d;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    # Public assets
    location /images {
        proxy_pass http://vondi_ai_landing;
        proxy_cache_valid 200 30d;
        add_header Cache-Control "public, max-age=2592000";
    }

    # API routes (no cache)
    location /api {
        proxy_pass http://vondi_ai_landing;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # All other requests
    location / {
        proxy_pass http://vondi_ai_landing;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## PM2 Configuration

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'vondi-ai-landing',
      script: 'node_modules/.bin/next',
      args: 'start -p 3008',
      cwd: '/opt/vondi-ai-landing',
      instances: 1,
      autorestart: true,
      watch: false,
      max_memory_restart: '1G',
      env: {
        NODE_ENV: 'production',
        PORT: 3008,
      },
      error_file: '/var/log/pm2/vondi-ai-landing-error.log',
      out_file: '/var/log/pm2/vondi-ai-landing-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    },
  ],
};
```

### PM2 команды

```bash
# Запуск
pm2 start ecosystem.config.js

# Статус
pm2 status vondi-ai-landing

# Логи
pm2 logs vondi-ai-landing --lines 100

# Перезапуск
pm2 restart vondi-ai-landing

# Остановка
pm2 stop vondi-ai-landing
```

---

## Scripts

### deploy.sh

```bash
#!/bin/bash
set -e

echo "Deploying vondi-ai-landing to solutions.vondi.rs..."

# Build
yarn build

# Sync to server
rsync -avz --delete \
  --exclude 'node_modules' \
  --exclude '.git' \
  --exclude '.env.local' \
  ./ vondi:/opt/vondi-ai-landing/

# Install deps and restart on server
ssh vondi "cd /opt/vondi-ai-landing && yarn install --production && pm2 restart vondi-ai-landing"

echo "Deploy complete!"
```

### setup-ssl.sh

```bash
#!/bin/bash
# Setup SSL certificate with Let's Encrypt

sudo certbot certonly --nginx \
  -d solutions.vondi.rs \
  --non-interactive \
  --agree-tos \
  -m mail@vondi.rs

# Auto-renewal
sudo systemctl enable certbot.timer
```

---

## Мониторинг

### Health Check

```bash
# Проверка health endpoint
curl -s http://localhost:3008 | head -1

# Проверка API
curl -s -X POST http://localhost:3008/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"test","locale":"en"}' | jq .
```

### Логи

```bash
# PM2 логи
pm2 logs vondi-ai-landing

# Docker логи
docker logs -f vondi-ai-landing

# Nginx логи
tail -f /var/log/nginx/access.log | grep solutions.vondi.rs
```

---

## Зависимости внешних сервисов

| Сервис | Назначение | Критичность |
|--------|-----------|-------------|
| **Anthropic API** | AI Chat демо | Критичный |
| **Zoho Mail** | Email нотификации | Опциональный |
| **S3 (s3.vondi.rs)** | Хранение изображений | Опциональный |
| **Plausible** | Аналитика | Опциональный |

### Fallback поведение

- **Без Anthropic API:** Chat API возвращает 503 ошибку
- **Без Zoho Mail:** Email логируется в консоль (dev mode)
- **Без S3:** Используются локальные изображения из /public
- **Без Plausible:** Аналитика не работает, приложение функционирует
