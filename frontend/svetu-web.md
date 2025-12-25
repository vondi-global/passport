# Frontend Passport - Vondi Web Application

> **Статус:** Production-ready | **Версия:** 0.3.59 | **Технология:** Next.js 15 App Router

---

## 1. Обзор

### Технологический стек

| Категория | Технология | Версия | Назначение |
|-----------|-----------|--------|------------|
| **Framework** | Next.js | 15.3.2 | React framework с App Router |
| **React** | React | 19.0.0 | UI библиотека |
| **Language** | TypeScript | 5.x | Типизация |
| **Styling** | Tailwind CSS | 4.x | Utility-first CSS |
| **UI Kit** | DaisyUI | 5.0.42 | Компоненты |
| **State** | Redux Toolkit | 2.8.2 | Глобальное состояние |
| **Data Fetching** | TanStack Query | 5.80.7 | Server state |
| **Forms** | React Hook Form | 7.58.1 | Формы + валидация |
| **Maps** | Mapbox GL | 3.15.0 | Интерактивные карты |
| **i18n** | next-intl | 4.1.0 | Интернационализация |
| **HTTP Client** | Axios | 1.11.0 | API запросы |

### Основные характеристики

- **Port:** 3001
- **Base URL:** http://localhost:3001 (dev), https://dev.vondi.rs (prod)
- **App Router Routes:** 69+
- **Components:** 100+ (включая новые компоненты редактирования товаров)
- **Custom Hooks:** 46+
- **Redux Slices:** 18
- **Supported Locales:** en, ru, sr (Serbian Latin)
- **Package Manager:** Yarn 1.22.22

---

## 2. Структура проекта

```
frontend/
├── public/                      # Статические файлы
│   ├── images/                  # Изображения, иконки
│   ├── locales/                 # JSON переводы (legacy)
│   └── manifest.json            # PWA manifest
├── src/
│   ├── app/                     # Next.js App Router
│   │   ├── [locale]/            # Локализованные маршруты
│   │   │   ├── admin/           # Админ панель
│   │   │   ├── auth/            # Аутентификация
│   │   │   ├── cart/            # Корзина
│   │   │   ├── chat/            # Чат
│   │   │   ├── checkout/        # Оформление заказа
│   │   │   ├── create-listing/  # Создание объявлений
│   │   │   ├── delivery/        # Доставка
│   │   │   ├── favorites/       # Избранное
│   │   │   ├── franchise/       # Франшиза
│   │   │   ├── my-warehouse/    # Склад (WMS)
│   │   │   ├── orders/          # Заказы
│   │   │   ├── profile/         # Профиль
│   │   │   ├── search/          # Поиск
│   │   │   └── ...              # Остальные маршруты
│   │   └── api/                 # API Routes (BFF)
│   │       ├── v2/[...path]/    # BFF Proxy (основной)
│   │       ├── auth/            # Auth endpoints
│   │       ├── ai/              # AI endpoints
│   │       └── admin/           # Admin API
│   ├── components/              # React компоненты
│   │   ├── admin/               # Админка
│   │   ├── auth/                # Аутентификация
│   │   ├── cart/                # Корзина
│   │   ├── categories/          # Категории
│   │   ├── chat/                # Чат
│   │   ├── checkout/            # Checkout
│   │   ├── common/              # Общие компоненты
│   │   ├── delivery/            # Доставка
│   │   ├── listings/            # Объявления
│   │   ├── maps/                # Карты
│   │   ├── profile/             # Профиль
│   │   ├── search/              # Поиск
│   │   └── wms/                 # WMS компоненты
│   ├── services/                # API клиенты
│   │   ├── api-client.ts        # Основной HTTP клиент (BFF)
│   │   ├── auth.ts              # Аутентификация
│   │   ├── listings.ts          # Объявления
│   │   ├── delivery.ts          # Доставка
│   │   ├── cart.ts              # Корзина
│   │   ├── payment/             # Платежи
│   │   └── ...                  # Остальные сервисы
│   ├── store/                   # Redux Store
│   │   ├── slices/              # Redux Slices
│   │   ├── hooks.ts             # Типизированные хуки
│   │   └── index.ts             # Store конфигурация
│   ├── hooks/                   # Custom React Hooks
│   ├── contexts/                # React Contexts
│   ├── types/                   # TypeScript типы
│   ├── utils/                   # Утилиты
│   ├── lib/                     # Библиотеки (Mapbox, Redis, etc)
│   ├── messages/                # i18n переводы (next-intl)
│   │   ├── en/                  # English
│   │   ├── ru/                  # Russian
│   │   └── sr/                  # Serbian
│   ├── config/                  # Конфигурация
│   ├── constants/               # Константы
│   ├── i18n/                    # i18n setup
│   ├── middleware.ts            # Next.js middleware (locale, auth)
│   └── providers/               # React providers
├── scripts/                     # Утилиты сборки
├── .env.example                 # Пример переменных окружения
├── next.config.ts               # Next.js конфигурация
├── tailwind.config.ts           # Tailwind конфигурация
├── tsconfig.json                # TypeScript конфигурация
└── package.json                 # Dependencies
```

---

## 3. App Router - Маршруты

### Public Routes (без аутентификации)

| Маршрут | Файл | Описание |
|---------|------|----------|
| `/` | `app/[locale]/page.tsx` | Главная страница |
| `/search` | `app/[locale]/search/page.tsx` | Поиск объявлений |
| `/listing/[id]` | `app/[locale]/listing/[id]/page.tsx` | Детали объявления |
| `/auth/login` | `app/[locale]/auth/login/page.tsx` | Вход |
| `/auth/register` | `app/[locale]/auth/register/page.tsx` | Регистрация |
| `/privacy` | `app/[locale]/privacy/page.tsx` | Политика конфиденциальности |
| `/rules` | `app/[locale]/rules/page.tsx` | Правила платформы |
| `/how-to-buy` | `app/[locale]/how-to-buy/page.tsx` | Как покупать |
| `/how-to-sell` | `app/[locale]/how-to-sell/page.tsx` | Как продавать |

### Protected Routes (требуют аутентификацию)

| Маршрут | Файл | Описание |
|---------|------|----------|
| `/profile` | `app/[locale]/profile/page.tsx` | Профиль пользователя |
| `/profile/settings` | `app/[locale]/profile/settings/page.tsx` | Настройки профиля |
| `/create-listing` | `app/[locale]/create-listing/page.tsx` | Создание объявления |
| `/create-listing-smart` | `app/[locale]/create-listing-smart/page.tsx` | AI-powered создание |
| `/my-listings` | `app/[locale]/my-listings/page.tsx` | Мои объявления |
| `/favorites` | `app/[locale]/favorites/page.tsx` | Избранное |
| `/cart` | `app/[locale]/cart/page.tsx` | Корзина |
| `/checkout` | `app/[locale]/checkout/page.tsx` | Оформление заказа |
| `/orders` | `app/[locale]/orders/page.tsx` | Заказы |
| `/chat` | `app/[locale]/chat/page.tsx` | Чат |
| `/balance` | `app/[locale]/balance/page.tsx` | Баланс |
| `/subscription` | `app/[locale]/subscription/page.tsx` | Подписки |

### Admin Routes (требуют роль admin)

| Маршрут | Файл | Описание |
|---------|------|----------|
| `/admin/dashboard` | `app/[locale]/admin/dashboard/page.tsx` | Админ панель |
| `/admin/users` | `app/[locale]/admin/users/page.tsx` | Управление пользователями |
| `/admin/categories` | `app/[locale]/admin/categories/page.tsx` | Управление категориями |
| `/admin/listings` | `app/[locale]/admin/listings/page.tsx` | Модерация объявлений |
| `/admin/orders` | `app/[locale]/admin/orders/page.tsx` | Управление заказами |
| `/admin/payments` | `app/[locale]/admin/payments/page.tsx` | Платежи |
| `/admin/logistics` | `app/[locale]/admin/logistics/page.tsx` | Логистика |
| `/admin/storefronts` | `app/[locale]/admin/storefronts/page.tsx` | Витрины B2C |

### WMS Routes (Warehouse Management)

| Маршрут | Файл | Описание |
|---------|------|----------|
| `/my-warehouse` | `app/[locale]/my-warehouse/page.tsx` | Dashboard владельца |
| `/my-warehouse/inventory` | `app/[locale]/my-warehouse/inventory/page.tsx` | Инвентаризация |
| `/my-warehouse/staff` | `app/[locale]/my-warehouse/staff/page.tsx` | Управление сотрудниками |
| `/my-warehouse/orders` | `app/[locale]/my-warehouse/orders/page.tsx` | Заказы склада |

### B2C Routes (Storefront)

| Маршрут | Файл | Описание |
|---------|------|----------|
| `/b2c` | `app/[locale]/b2c/page.tsx` | B2C маркетплейс |
| `/b2c/store/[id]` | `app/[locale]/b2c/store/[id]/page.tsx` | Страница магазина |
| `/b2c/product/[id]` | `app/[locale]/b2c/product/[id]/page.tsx` | Страница товара |

### C2C Routes (Customer-to-Customer)

| Маршрут | Файл | Описание |
|---------|------|----------|
| `/c2c` | `app/[locale]/c2c/page.tsx` | C2C объявления |
| `/c2c/cars` | `app/[locale]/cars/page.tsx` | Автомобили |

### Specialty Routes

| Маршрут | Файл | Описание |
|---------|------|----------|
| `/franchise` | `app/[locale]/franchise/page.tsx` | Франшиза PVZ |
| `/delivery` | `app/[locale]/delivery/page.tsx` | Отслеживание доставки |
| `/track` | `app/[locale]/track/page.tsx` | Track заказа |

---

## 4. BFF Proxy Architecture (Backend-for-Frontend)

### Схема работы

```
Browser → /api/v2/* (Next.js BFF) → /api/v1/* (Backend)
         └─ httpOnly cookies     └─ Authorization: Bearer <JWT>
```

### Основной прокси: `/api/v2/[...path]/route.ts`

**Назначение:** Перенаправление всех API запросов на backend с автоматической авторизацией.

**Преимущества:**
- ✅ JWT токены в httpOnly cookies (безопасность)
- ✅ Нет CORS проблем (same-origin)
- ✅ Централизованная авторизация
- ✅ Простота использования (не нужно управлять токенами в frontend)

**Пример использования:**

```typescript
// ✅ ПРАВИЛЬНО - через apiClient
import { apiClient } from '@/services/api-client';

// Запрос автоматически идет через /api/v2/* → backend /api/v1/*
const response = await apiClient.get('/admin/categories');
const response = await apiClient.post('/marketplace/listings', data);

// ❌ НЕПРАВИЛЬНО - прямые запросы к backend
fetch('http://localhost:3000/api/v1/...')  // Никогда не делай так!
```

### API Client: `services/api-client.ts`

```typescript
// Базовая конфигурация
const apiClient = axios.create({
  baseURL: '/api/v2',  // BFF proxy
  withCredentials: true,  // Включить cookies
  headers: {
    'Content-Type': 'application/json',
  },
});

// Использование
const { data } = await apiClient.get<Listing[]>('/marketplace/listings');
```

### Auth Endpoints (API Routes)

| Endpoint | Файл | Описание |
|----------|------|----------|
| `/api/auth/login` | `app/api/auth/login/route.ts` | Логин |
| `/api/auth/register` | `app/api/auth/register/route.ts` | Регистрация |
| `/api/auth/logout` | `app/api/auth/logout/route.ts` | Выход |
| `/api/auth/refresh` | `app/api/auth/refresh/route.ts` | Обновление токена |
| `/api/auth/session` | `app/api/auth/session/route.ts` | Текущая сессия |
| `/api/auth/google` | `app/api/auth/google/route.ts` | OAuth Google |

### AI Endpoints

| Endpoint | Файл | Описание |
|----------|------|----------|
| `/api/ai/description` | `app/api/ai/description/route.ts` | Генерация описаний |
| `/api/ai/translate` | `app/api/ai/translate/route.ts` | Перевод |
| `/api/ai/analyze` | `app/api/ai/analyze/route.ts` | Анализ контента |

### Переменные окружения

```bash
# Backend URL для BFF proxy (server-side)
BACKEND_INTERNAL_URL=http://localhost:3000

# Public API URL (для browser-side, не используется с BFF)
NEXT_PUBLIC_API_URL=http://localhost:3000
```

---

## 5. State Management - Redux Toolkit

### Redux Store Structure

**Конфигурация:** `src/store/index.ts`

**Основные слайсы:** (18 total)

| Slice | Файл | Назначение |
|-------|------|------------|
| `cart` | `cartSlice.ts` | Корзина (server-synced) |
| `localCart` | `localCartSlice.ts` | Локальная корзина (гостевой режим) |
| `checkout` | `checkoutSlice.ts` | Оформление заказа |
| `categories` | `categoriesSlice.ts` | Категории маркетплейса |
| `favorites` | `favoritesSlice.ts` | Избранные объявления |
| `product` | `productSlice.ts` | Детали товара |
| `delivery` | `deliverySlice.ts` | Доставка (адреса, методы) |
| `payment` | `paymentSlice.ts` | Платежные методы |
| `chat` | `chatSlice.ts` | Чат (сообщения, статус) |
| `reviews` | `reviewsSlice.ts` | Отзывы |
| `b2cStore` | `b2cStoreSlice.ts` | B2C витрины |
| `savedSearches` | `savedSearchesSlice.ts` | Сохраненные поиски |
| `universalCompare` | `universalCompareSlice.ts` | Сравнение товаров |
| `import` | `importSlice.ts` | Импорт данных |
| `categoryProposals` | `categoryProposalsSlice.ts` | Предложения категорий |

### Типизированные хуки

```typescript
// src/store/hooks.ts
import { useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './index';

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

### Примеры использования

```typescript
// Чтение состояния
const cartItems = useAppSelector(state => state.cart.items);

// Dispatch действий
const dispatch = useAppDispatch();
dispatch(addToCart({ listingId: 123, quantity: 1 }));

// Async thunks
dispatch(fetchCategories());
```

---

## 6. API Клиенты

### Основной клиент: `api-client.ts`

```typescript
// BFF-based HTTP клиент
const apiClient = axios.create({
  baseURL: '/api/v2',
  withCredentials: true,
});

// Interceptors для обработки ошибок
apiClient.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      // Redirect to login
    }
    return Promise.reject(error);
  }
);
```

### Специализированные API клиенты

| Файл | Назначение |
|------|------------|
| `auth.ts` | Аутентификация (login, register, OAuth) |
| `listings.ts` | Объявления (CRUD, search, images) |
| `delivery.ts` | Доставка (адреса, методы, расчет стоимости) |
| `cart.ts` | Корзина (add, remove, sync) |
| `orders.ts` | Заказы (create, list, details) |
| `payment/` | Платежи (AllSecure, PayPal) |
| `category.ts` | Категории (tree, attributes, filters) |
| `chat.ts` | Чат (messages, rooms, typing) |
| `reviews.ts` | Отзывы (list, create, moderate) |
| `admin.ts` | Админка (users, moderation, analytics) |
| `b2cStoreApi.ts` | B2C витрины |
| `wms.ts` | Warehouse Management System |
| `franchise.ts` | Франшиза PVZ |
| `ai/` | AI сервисы (Claude, image analysis) |

### Примеры вызовов

```typescript
// Listings
import { getListingById, createListing } from '@/services/listings';

const listing = await getListingById(123);
const newListing = await createListing({ title: 'iPhone 15', price: 999 });

// Cart
import { addToCart, getCart } from '@/services/cart';

await addToCart({ listingId: 456, quantity: 2 });
const cart = await getCart();

// Delivery
import { calculateDeliveryPrice } from '@/services/delivery';

const price = await calculateDeliveryPrice({
  from: { lat: 44.8, lng: 20.4 },
  to: { lat: 44.9, lng: 20.5 },
});
```

---

## 7. Интернационализация (i18n)

### Поддерживаемые локали

- **en** - English
- **ru** - Russian (Русский)
- **sr** - Serbian Latin (Srpski latinica)

### Архитектура переводов

**Библиотека:** `next-intl` v4.1.0

**Структура файлов:**

```
messages/
├── en/
│   ├── common.json          # Общие переводы
│   ├── auth.json            # Аутентификация
│   ├── marketplace.json     # Маркетплейс
│   ├── cart.json            # Корзина
│   ├── checkout.json        # Checkout
│   ├── delivery.json        # Доставка
│   ├── admin.json           # Админка
│   ├── errors.json          # Ошибки
│   └── ...
├── ru/                      # То же для русского
└── sr/                      # То же для сербского
```

### Принцип работы

**Backend → Frontend flow:**

```javascript
// Backend возвращает placeholder (ключ перевода):
{ "error": "storefronts.no_image_file" }

// Frontend переводит через next-intl:
import { useTranslations } from 'next-intl';

const t = useTranslations('storefronts');
t('no_image_file')  // → "Файл изображения не найден" (ru)
```

### Middleware для локали

```typescript
// src/middleware.ts
import createMiddleware from 'next-intl/middleware';

export default createMiddleware({
  locales: ['en', 'ru', 'sr'],
  defaultLocale: 'sr',
  localePrefix: 'always',  // Всегда префикс: /sr/*, /en/*, /ru/*
});
```

### Примеры использования

```typescript
// В компонентах
import { useTranslations } from 'next-intl';

function MyComponent() {
  const t = useTranslations('common');

  return (
    <div>
      <h1>{t('welcome')}</h1>
      <p>{t('description', { count: 5 })}</p>
    </div>
  );
}

// В серверных компонентах
import { getTranslations } from 'next-intl/server';

async function ServerComponent() {
  const t = await getTranslations('marketplace');
  return <h1>{t('title')}</h1>;
}
```

### Переключение языка

```typescript
import { useRouter, usePathname } from 'next/navigation';

function LanguageSwitcher() {
  const router = useRouter();
  const pathname = usePathname();

  const switchLocale = (locale: string) => {
    const newPath = pathname.replace(/^\/(en|ru|sr)/, `/${locale}`);
    router.push(newPath);
  };

  return (
    <select onChange={(e) => switchLocale(e.target.value)}>
      <option value="sr">Srpski</option>
      <option value="en">English</option>
      <option value="ru">Русский</option>
    </select>
  );
}
```

---

## 8. Компоненты

### Основные категории компонентов

```
components/
├── admin/                   # Админ панель
│   ├── AdminDashboard.tsx
│   ├── UserManagement.tsx
│   ├── CategoryEditor.tsx
│   └── LogisticsPanel.tsx
├── auth/                    # Аутентификация
│   ├── AuthModal.tsx
│   ├── LoginForm.tsx
│   ├── RegisterForm.tsx
│   └── OAuthButtons.tsx
├── cart/                    # Корзина
│   ├── CartItem.tsx
│   ├── CartSummary.tsx
│   └── EmptyCart.tsx
├── categories/              # Категории
│   ├── CategoryTree.tsx
│   ├── CategoryBreadcrumb.tsx
│   └── CategoryFilter.tsx
├── checkout/                # Checkout
│   ├── CheckoutForm.tsx
│   ├── DeliveryAddressForm.tsx
│   ├── PaymentMethodSelector.tsx
│   └── OrderSummary.tsx
├── common/                  # Общие компоненты
│   ├── Button.tsx
│   ├── Input.tsx
│   ├── Modal.tsx
│   ├── Loader.tsx
│   ├── ErrorBoundary.tsx
│   └── UnifiedImageUploader/     # Загрузка изображений
│       ├── UnifiedImageUploaderV2.tsx  # Главный компонент
│       ├── ImageGridDndKit.tsx         # @dnd-kit grid для перетаскивания
│       ├── SortableImageCard.tsx       # Sortable wrapper
│       └── UploadProgress.tsx          # Прогресс загрузки
├── delivery/                # Доставка
│   ├── AddressAutocomplete.tsx
│   ├── MapPicker.tsx
│   └── DeliveryMethodCard.tsx
├── listings/                # Объявления
│   ├── ListingCard.tsx
│   ├── ListingDetail.tsx
│   ├── ImageGallery.tsx
│   └── CreateListingForm.tsx
├── products/                # B2C товары (редактирование)
│   ├── EditProductWizard.tsx     # 7-шаговый wizard редактирования
│   ├── EditProductTabs.tsx       # Tabbed interface (6 вкладок)
│   ├── steps/                    # Шаги wizard'а
│   │   ├── EditVariantsStep.tsx  # Управление вариантами товара
│   │   ├── EditBasicInfoStep.tsx
│   │   ├── EditPhotosStep.tsx
│   │   ├── EditPreviewStep.tsx
│   │   └── ...
│   └── tabs/                     # Вкладки tabbed интерфейса
│       ├── EditProductBasicTab.tsx     # Inline редактирование
│       ├── EditProductVariantsTab.tsx  # Simple/Advanced режимы
│       ├── EditProductImagesTab.tsx    # Загрузка изображений
│       ├── EditProductLocationTab.tsx  # Локация товара
│       └── EditProductPreviewTab.tsx   # Предпросмотр
├── maps/                    # Карты (Mapbox)
│   ├── InteractiveMap.tsx
│   ├── MarkerCluster.tsx
│   └── RouteDisplay.tsx
├── profile/                 # Профиль
│   ├── ProfileCard.tsx
│   ├── SettingsForm.tsx
│   └── BalanceWidget.tsx
├── search/                  # Поиск
│   ├── SearchBar.tsx
│   ├── FilterSidebar.tsx
│   ├── SearchResults.tsx
│   └── AttributeFilters.tsx
└── wms/                     # WMS (Warehouse)
    ├── InventoryTable.tsx
    ├── StaffManagement.tsx
    └── OrderPickingQueue.tsx
```

### Универсальные компоненты (common/)

| Компонент | Назначение |
|-----------|------------|
| `Button.tsx` | Кнопки (primary, secondary, danger) |
| `Input.tsx` | Поля ввода (text, email, password) |
| `Modal.tsx` | Модальные окна |
| `Loader.tsx` | Индикаторы загрузки |
| `ErrorBoundary.tsx` | Обработка ошибок React |
| `Pagination.tsx` | Пагинация |
| `Tabs.tsx` | Табы |
| `Dropdown.tsx` | Выпадающие списки |
| `Toast.tsx` | Уведомления (react-hot-toast) |

---

## 9. Кастомные хуки (Custom Hooks)

### Data Fetching Hooks

| Hook | Назначение |
|------|------------|
| `useApi.ts` | Универсальный API хук (fetch, loading, error) |
| `usePaginatedData.ts` | Пагинированные данные |
| `useInfiniteScroll.ts` | Бесконечный скролл |
| `useWebSocket.ts` | WebSocket подключение (чат, уведомления) |

### Form Hooks

| Hook | Назначение |
|------|------------|
| `useAuthForm.ts` | Формы аутентификации |
| `useFormValidation.ts` | Валидация форм |

### Search & Filters

| Hook | Назначение |
|------|------------|
| `useSearchFilters.ts` | Фильтры поиска (category, price, location) |
| `useCategoryFilters.ts` | Фильтры по категориям |
| `useFilterPersistence.ts` | Сохранение фильтров (URL + localStorage) |
| `useSearchWithAttributes.ts` | Поиск с атрибутами (dynamic filters) |
| `useVoiceSearch.ts` | Голосовой поиск |

### Category Detection

| Hook | Назначение |
|------|------------|
| `useCategoryDetection.ts` | AI определение категории по описанию/фото |
| `useAttributeAutocomplete.ts` | Автодополнение атрибутов |
| `useAttributesPagination.ts` | Пагинация атрибутов |

### Maps & Location

| Hook | Назначение |
|------|------------|
| `useAddressGeocoding.ts` | Геокодирование адресов (Mapbox) |
| `useMultilingualGeocoding.ts` | Мультиязычное геокодирование |
| `useAdvancedGeoFilters.ts` | Geo фильтры (radius, polygon) |
| `useMapCache.ts` | Кэширование карт (IndexedDB) |

### Commerce

| Hook | Назначение |
|------|------------|
| `useCartSync.ts` | Синхронизация корзины (server ↔ local) |
| `useB2CStores.ts` | B2C витрины |
| `useBalance.ts` | Баланс пользователя |
| `useAllSecurePayment.ts` | AllSecure интеграция |
| `useAllSecurePaymentHelpers.ts` | Вспомогательные функции AllSecure |

### Chat & Messaging

| Hook | Назначение |
|------|------------|
| `useChat.ts` | Чат (messages, rooms, typing indicators) |
| `useIncomingContactRequests.ts` | Входящие контакты |

### User Engagement

| Hook | Назначение |
|------|------------|
| `useReviews.ts` | Отзывы (list, create, moderate) |
| `useSubscription.ts` | Подписки |
| `useAnalytics.ts` | Аналитика (события) |
| `useBehaviorTracking.ts` | Отслеживание поведения пользователя |
| `useABTest.ts` | A/B тестирование |

### UI/UX Hooks

| Hook | Назначение |
|------|------------|
| `useDebounce.ts` | Debounce (поиск, автодополнение) |
| `useLocalStorage.ts` | localStorage с типизацией |
| `useObjectURL.ts` | Создание blob URLs (preview изображений) |
| `useViewPreference.ts` | Сохранение view preference (grid/list) |
| `useGridColumns.ts` | Адаптивные grid columns |
| `useCountAnimation.ts` | Анимация чисел (counter) |
| `useProgressiveLoading.ts` | Прогрессивная загрузка изображений |

### Mobile Optimization

| Hook | Назначение |
|------|------------|
| `useMobileOptimization.ts` | Мобильные оптимизации |
| `useTouchGestures.ts` | Жесты (swipe, pinch, zoom) |
| `useHapticFeedback.ts` | Haptic feedback (вибрация) |
| `useNativeShare.ts` | Native share API |

### Data Management

| Hook | Назначение |
|------|------------|
| `useListingDraft.ts` | Черновики объявлений (auto-save) |
| `useCarData.ts` | Автомобильные данные (brands, models) |

### Configuration

| Hook | Назначение |
|------|------------|
| `useConfig.ts` | Runtime конфигурация (env variables) |

---

## 10. Стилизация (Tailwind CSS)

### Конфигурация Tailwind

**Файл:** `tailwind.config.ts`

```typescript
import type { Config } from 'tailwindcss';
import daisyui from 'daisyui';

const config: Config = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: '#6366f1',    // Indigo
        secondary: '#ec4899',  // Pink
        accent: '#f59e0b',     // Amber
      },
    },
  },
  plugins: [daisyui],
  daisyui: {
    themes: ['light', 'dark', 'cupcake'],
    darkTheme: 'dark',
  },
};
```

### DaisyUI компоненты

**Используемые компоненты:**
- `btn` - Кнопки
- `card` - Карточки
- `modal` - Модальные окна
- `badge` - Бейджи
- `alert` - Уведомления
- `dropdown` - Выпадающие меню
- `tabs` - Табы
- `table` - Таблицы
- `form-control`, `input`, `textarea` - Формы

### Пример использования

```tsx
// Tailwind utility classes + DaisyUI
<button className="btn btn-primary">
  Добавить в корзину
</button>

<div className="card bg-base-100 shadow-xl">
  <div className="card-body">
    <h2 className="card-title">Заголовок</h2>
    <p>Описание</p>
  </div>
</div>
```

### Адаптивный дизайн

```tsx
// Responsive utilities
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  {/* Mobile: 1 колонка, Tablet: 2 колонки, Desktop: 3 колонки */}
</div>

<div className="text-sm md:text-base lg:text-lg">
  {/* Адаптивный размер шрифта */}
</div>
```

---

## 11. Полезные команды

### Быстрый старт

```bash
# Установка зависимостей
yarn install

# Создать .env.local
cp .env.example .env.local

# Запустить dev сервер
yarn dev
```

### Development

```bash
# Запуск dev сервера (Turbopack)
yarn dev

# Запуск с Webpack (для отладки)
yarn dev:webpack

# Проверка environment variables
yarn env:check
```

### Build & Production

```bash
# Production build
yarn build

# Запуск production сервера
yarn start

# Preview build локально
yarn build && yarn start
```

### Code Quality

```bash
# Linting
yarn lint

# Lint + автофикс
yarn lint:fix

# Форматирование (Prettier)
yarn format

# Проверка форматирования
yarn format:check
```

### Testing

```bash
# Запуск тестов (Jest)
yarn test

# Тесты с watch mode
yarn test:watch

# Coverage отчет
yarn test:coverage
```

### Type Generation

```bash
# Генерация TypeScript типов из OpenAPI
yarn generate:types
```

### Version Management

```bash
# Bump patch версии (0.3.58 → 0.3.59)
yarn version:bump

# Bump minor версии (0.3.58 → 0.4.0)
yarn version:bump:minor

# Bump major версии (0.3.58 → 1.0.0)
yarn version:bump:major
```

---

## 12. Deployment

### Local Development

```bash
# Запуск frontend (порт 3001)
cd /p/github.com/vondi-global/vondi/frontend
yarn dev
```

### Production Build

```bash
# Build для production
yarn build

# Проверка production build
yarn start
```

### Deployment на dev.vondi.rs

```bash
# Деплой всего проекта (backend + frontend)
cd /p/github.com/vondi-global/vondi
./deploy-to-dev.sh

# Только frontend rebuild
ssh vondi "cd /opt/vondi-dev/frontend && yarn build && pm2 restart vondi-frontend"
```

---

## 13. Контакты и документация

### Документация

- **Главная инструкция:** `/p/github.com/vondi-global/CLAUDE.md`
- **Frontend правила:** `/p/github.com/vondi-global/vondi/.ai/frontend.md`
- **System Passport:** `/p/github.com/vondi-global/SYSTEM_PASSPORT.md`

### Репозиторий

- **GitHub:** `vondi-global/vondi`
- **Branch:** `main`
- **PR Workflow:** Все изменения только через Pull Request

### Deployment

- **Dev:** https://dev.vondi.rs
- **API:** https://devapi.vondi.rs
- **S3:** https://s3.vondi.rs

---

**Дата создания:** 2025-12-21
**Версия документа:** 1.0
**Статус:** Production-ready
