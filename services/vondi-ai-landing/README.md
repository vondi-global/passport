# VONDI AI Landing - Паспорт проекта

> **Версия:** 0.1.0
> **Домен:** https://solutions.vondi.rs
> **Порт:** 3008
> **Дата актуализации:** 2025-12-12

## Общая информация

**vondi-ai-landing** - лендинг страница для AI-решений компании VONDI GLOBAL D.O.O. с интерактивной демонстрацией AI-чата на базе Claude Haiku.

### Назначение

- Презентация AI-решений для государственного и корпоративного сектора Балкан
- Интерактивная демонстрация AI-чатбота
- Сбор лидов через контактную форму
- SEO-оптимизированный контент для региона (Сербия, Черногория, Босния и Герцеговина, Хорватия, Словения, Северная Македония, Албания, Косово)

### Технологический стек

| Категория | Технологии |
|-----------|-----------|
| **Frontend** | Next.js 16.0.8, React 19.2.1, TypeScript 5, Tailwind CSS 4 |
| **Анимации** | Framer Motion 12.23.26 |
| **Формы** | React Hook Form 7.68.0 + Zod 4.1.13 |
| **i18n** | next-intl 4.5.8 (SR/EN/RU) |
| **AI** | Anthropic SDK (Claude 3 Haiku) |
| **Email** | Zoho Mail API (OAuth2) |
| **Testing** | Playwright 1.57.0 |
| **Deploy** | Docker, PM2, Nginx |

---

## Структура паспорта

Паспорт разделён на отдельные файлы для удобства навигации:

| Файл | Описание |
|------|----------|
| [01-infrastructure.md](./01-infrastructure.md) | Инфраструктура, конфигурация, деплой |
| [02-components.md](./02-components.md) | Frontend компоненты и их API |
| [03-api-services.md](./03-api-services.md) | API endpoints и backend сервисы |
| [04-seo-i18n.md](./04-seo-i18n.md) | SEO оптимизация и интернационализация |

---

## Структура директорий

```
vondi-ai-landing/
├── src/
│   ├── app/                          # Next.js App Router
│   │   ├── [locale]/                 # Динамические маршруты по языкам
│   │   │   ├── layout.tsx            # Layout с SEO метаданными
│   │   │   ├── page.tsx              # Главная страница
│   │   │   ├── about/                # О компании
│   │   │   ├── features/             # Возможности
│   │   │   ├── solutions/            # Решения
│   │   │   ├── pricing/              # Цены
│   │   │   ├── demo/                 # AI демо (fullscreen)
│   │   │   ├── contact/              # Контактная форма
│   │   │   ├── privacy/              # Политика конфиденциальности
│   │   │   └── terms/                # Условия использования
│   │   ├── api/
│   │   │   ├── chat/route.ts         # AI Chat API
│   │   │   └── contact/route.ts      # Contact Form API
│   │   ├── sitemap.ts                # Карта сайта
│   │   └── globals.css               # Глобальные стили
│   │
│   ├── components/
│   │   ├── layout/                   # Header, Footer, LanguageSwitcher
│   │   ├── sections/                 # Секции лендинга (Hero, Features, CTA...)
│   │   ├── chat/                     # AI чат компоненты
│   │   ├── forms/                    # Формы (ContactForm)
│   │   ├── ui/                       # UI компоненты (Button, Card, Badge, Input)
│   │   └── seo/                      # SEO компоненты (StructuredData)
│   │
│   ├── lib/                          # Утилиты и сервисы
│   │   ├── ai-client.ts              # Anthropic Claude клиент
│   │   ├── zoho-mail.ts              # Zoho Mail интеграция
│   │   ├── email.ts                  # Email сервис
│   │   ├── rate-limiter.ts           # Rate limiting
│   │   ├── analytics.ts              # Plausible Analytics
│   │   └── utils.ts                  # Общие утилиты
│   │
│   ├── hooks/                        # React хуки
│   │   ├── useIntersectionObserver.ts
│   │   └── useScrollAnimation.ts
│   │
│   ├── i18n/                         # Конфигурация i18n
│   │   ├── routing.ts
│   │   └── navigation.ts
│   │
│   ├── messages/                     # Переводы (JSON)
│   │   ├── sr.json                   # Сербский
│   │   ├── en.json                   # Английский
│   │   └── ru.json                   # Русский
│   │
│   ├── types/                        # TypeScript типы
│   │   └── index.ts
│   │
│   ├── middleware.ts                 # Next.js middleware для i18n
│   └── i18n.ts                       # Конфигурация next-intl
│
├── public/
│   ├── images/                       # Изображения и иконки
│   └── manifest.json                 # PWA манифест
│
├── docker/                           # Docker конфигурация
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   └── docker-compose.yml
│
├── nginx/                            # Nginx конфигурация
│   └── solutions.vondi.rs.conf
│
├── scripts/                          # Скрипты развёртывания
│   ├── deploy.sh
│   ├── setup-ssl.sh
│   └── test-pages.sh
│
└── docs/                             # Документация
    ├── PROGRESS.md
    ├── DEPLOYMENT_CHECKLIST.md
    └── SEO_CHECKLIST.md
```

---

## Быстрый старт

### Локальный запуск

```bash
cd /p/github.com/vondi-global/vondi-ai-landing

# Установка зависимостей
yarn install

# Создать .env.local из примера
cp .env.example .env.local
# Добавить ANTHROPIC_API_KEY

# Запуск dev сервера
yarn dev

# Открыть http://localhost:3008
```

### Production деплой

```bash
# Сборка
yarn build

# Запуск через PM2
pm2 start ecosystem.config.js

# Или через Docker
docker compose -f docker/docker-compose.yml up -d
```

---

## Ключевые URL

| Маршрут | Описание |
|---------|----------|
| `/{locale}` | Главная страница |
| `/{locale}/features` | Возможности AI |
| `/{locale}/solutions` | Решения по индустриям |
| `/{locale}/pricing` | Тарифные планы |
| `/{locale}/demo` | Полноэкранный AI демо |
| `/{locale}/contact` | Контактная форма |
| `/{locale}/about` | О компании |
| `/{locale}/privacy` | Политика конфиденциальности |
| `/{locale}/terms` | Условия использования |
| `/api/chat` | AI Chat API |
| `/api/contact` | Contact Form API |

**Локали:** `sr` (сербский, default), `en` (английский), `ru` (русский)

---

## Контакты

**Компания:** VONDI GLOBAL D.O.O.
**Адрес:** Белград, Сербия
**Email:** mail@vondi.rs
**Генеральный директор:** Малден Симић (Malden Simić)
