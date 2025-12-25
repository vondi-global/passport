# Frontend Components & Hooks Passport

> **Статус:** Production-ready | **Компонентов:** 470+ | **Хуков:** 44 | **Версия:** 0.3.59

---

## 1. Обзор

### Архитектура компонентов

Vondi Web использует **Component-Driven Architecture** с четким разделением:
- **UI Components** - презентационные компоненты (DaisyUI + Tailwind)
- **Smart Components** - бизнес-логика + состояние (Redux + React Query)
- **Layout Components** - структура страниц
- **Feature Components** - законченные фичи (поиск, корзина, checkout)

### Технологии

| Категория | Технология | Назначение |
|-----------|-----------|------------|
| **Framework** | React 19 | UI библиотека |
| **State** | Redux Toolkit | Глобальное состояние |
| **Forms** | React Hook Form | Формы + валидация |
| **UI Kit** | DaisyUI 5.0 | Компоненты |
| **Styling** | Tailwind CSS 4.x | Utility-first CSS |
| **i18n** | next-intl 4.1 | Переводы |
| **Maps** | Mapbox GL 3.15 | Карты |

---

## 2. Структура директорий

```
components/
├── abtest/              # A/B тестирование (4)
├── address/             # Адреса, геокодирование (8)
├── admin/               # Админ панель (65+)
│   ├── categories/      # Управление категориями
│   ├── logistics/       # Логистика, доставка
│   ├── orders/          # Заказы
│   ├── payments/        # Платежи
│   ├── products/        # Товары
│   ├── storefronts/     # B2C витрины
│   ├── translations/    # AI переводы
│   └── users/           # Пользователи
├── attributes/          # Атрибуты товаров (12)
├── auth/                # Аутентификация (15)
│   ├── LoginForm.tsx
│   ├── RegisterForm.tsx
│   ├── OAuthButtons.tsx
│   └── PhoneVerification.tsx
├── B2C/                 # B2C маркетплейс (28)
│   ├── ProductVariants/ # Варианты товаров
│   ├── Storefront/      # Витрины магазинов
│   └── Checkout/        # Оформление заказа
├── balance/             # Баланс пользователя (6)
├── c2c/                 # C2C объявления (18)
├── cars/                # Автомобильный раздел (45)
│   ├── CarSelector.tsx  # Подбор авто (марка-модель)
│   ├── VinDecoder.tsx   # VIN декодер
│   ├── CarFilters.tsx   # Фильтры
│   └── Comparison.tsx   # Сравнение авто
├── cart/                # Корзина (12)
│   ├── CartItem.tsx     # Товар в корзине
│   ├── CartSummary.tsx  # Итоги корзины
│   └── EmptyCart.tsx    # Пустая корзина
├── categories/          # Категории (18)
│   ├── CategoryTree.tsx # Дерево категорий
│   ├── CategoryBreadcrumb.tsx
│   └── CategoryFilter.tsx
├── chat/                # Чат (22)
│   ├── ChatWindow.tsx   # Окно чата
│   ├── MessageList.tsx  # Список сообщений
│   └── ContactsList.tsx # Контакты
├── checkout/            # Оформление заказа (16)
│   ├── CheckoutForm.tsx
│   ├── DeliveryAddress.tsx
│   ├── PaymentMethod.tsx
│   └── OrderSummary.tsx
├── common/              # Общие UI (18)
│   ├── Button.tsx
│   ├── Modal.tsx
│   ├── Loader.tsx
│   ├── InfiniteScroll.tsx
│   └── LazyImage.tsx
├── create-listing/      # Создание объявлений (35)
│   ├── SmartListingForm.tsx # AI-powered создание
│   ├── ImageUpload.tsx
│   ├── CategoryDetection.tsx
│   └── PriceRecommendation.tsx
├── delivery/            # Доставка (24)
│   ├── AddressAutocomplete.tsx
│   ├── DeliveryMethodCard.tsx
│   └── TrackingMap.tsx
├── filters/             # Фильтры поиска (14)
├── GIS/                 # Геоинформационные системы (42)
│   ├── Map/             # Mapbox компоненты (18)
│   │   ├── MapMarker.tsx
│   │   ├── MapCluster.tsx
│   │   ├── PriceMarker.tsx
│   │   └── StorefrontMarker.tsx
│   ├── Mobile/          # Мобильные карты (8)
│   ├── SmartAddress.tsx # Умный ввод адреса
│   └── DistrictFilter.tsx
├── home/                # Главная страница (12)
│   ├── HeroSection.tsx
│   ├── Categories.tsx
│   ├── FeaturedListings.tsx
│   └── MapSection.tsx
├── import/              # Импорт данных (8)
├── invitations/         # Приглашения (4)
├── listing/             # Детали объявления (28)
│   ├── ListingDetail.tsx
│   ├── ImageGallery.tsx
│   ├── Reviews.tsx
│   └── SimilarListings.tsx
├── logos/               # Логотипы (6 вариаций)
│   ├── SveTuLogo3D.tsx
│   ├── SveTuLogoParticles.tsx
│   └── SveTuLogoStatic.tsx
├── mobile/              # Мобильные компоненты (8)
├── navigation/          # Навигация (14)
│   ├── Header.tsx
│   ├── Footer.tsx
│   ├── Sidebar.tsx
│   └── Breadcrumbs.tsx
├── orders/              # Заказы (16)
│   ├── OrderList.tsx
│   ├── OrderDetail.tsx
│   └── OrderTracking.tsx
├── payment/             # Платежи (22)
│   ├── AllSecure/       # AllSecure интеграция
│   ├── PayPal/          # PayPal
│   └── BalancePayment.tsx
├── phone/               # Телефон (4)
├── postexpress/         # Post Express (6)
├── product/             # Товары B2C (18)
├── products/            # Товары C2C/B2C (45)
│   ├── ProductWizard.tsx      # Wizard создания товара
│   ├── EditProductWizard.tsx  # Wizard редактирования товара (7 шагов)
│   ├── EditProductTabs.tsx    # Tabbed редактирование товара (6 вкладок)
│   ├── VariantGenerator.tsx
│   ├── BulkActions.tsx
│   ├── steps/                 # Шаги wizard'а
│   │   ├── EditVariantsStep.tsx    # Шаг редактирования вариантов
│   │   ├── EditBasicInfoStep.tsx
│   │   ├── EditPhotosStep.tsx
│   │   └── ...
│   └── tabs/                  # Вкладки для tabbed интерфейса
│       ├── EditProductBasicTab.tsx
│       ├── EditProductVariantsTab.tsx
│       ├── EditProductImagesTab.tsx
│       └── ...
├── reviews/             # Отзывы (10)
├── search/              # Поиск (38)
│   ├── SearchBar.tsx    # Главный поиск
│   ├── SearchAutocomplete.tsx
│   ├── SearchFilters.tsx
│   ├── ActiveFilters.tsx
│   ├── DynamicFilters.tsx
│   ├── LocationFilter.tsx
│   ├── CategoryFilter.tsx
│   └── MobileFilterDrawer.tsx
├── seo/                 # SEO (4)
├── shared/              # Общие (26)
├── staff/               # Персонал (6)
├── static/              # Статические страницы (8)
├── storefront/          # Витрины (28)
├── tracking/            # Отслеживание (6)
├── ui/                  # UI библиотека (32)
├── unified/             # Универсальные (12)
├── universal/           # Общие (8)
└── warehouse/           # WMS (45)
    ├── Dashboard.tsx
    ├── Inventory.tsx
    ├── StaffManagement.tsx
    └── OrderPicking.tsx
```

---

## 3. Основные компоненты по категориям

### 3.1 Аутентификация (auth/)

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **LoginForm** | `onSuccess?: () => void` | Форма входа (email/пароль) |
| **RegisterForm** | `onSuccess?: () => void` | Регистрация нового пользователя |
| **OAuthButtons** | `provider: 'google' \| 'facebook'` | OAuth кнопки |
| **PhoneVerification** | `phone: string, onVerify: (code) => void` | Firebase SMS верификация |
| **ForgotPassword** | - | Восстановление пароля |
| **AuthModal** | `isOpen: boolean, onClose: () => void` | Модальное окно auth |

**Используемые хуки:**
- `useAuthForm` - управление формами
- `useAuth` - контекст пользователя
- `useFormValidation` - валидация

**Пример:**
```tsx
import { LoginForm } from '@/components/auth/LoginForm';

<LoginForm onSuccess={() => router.push('/profile')} />
```

---

### 3.2 Поиск и фильтры (search/)

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **SearchBar** | `initialQuery?: string, onSearch: (q) => void` | Главная строка поиска |
| **SearchAutocomplete** | `query: string` | Автодополнение (подсказки) |
| **SearchFilters** | `filters: Filters, onChange: (f) => void` | Боковая панель фильтров |
| **ActiveFiltersChips** | `filters: Filters, onRemove: (key) => void` | Активные фильтры (чипсы) |
| **DynamicFilters** | `categoryId: number` | Динамические атрибуты категории |
| **LocationFilter** | `value?: Location, onChange: (l) => void` | Фильтр по локации |
| **DistrictMapSelector** | `onSelect: (district) => void` | Выбор района на карте |
| **SearchResultCard** | `listing: Listing` | Карточка в результатах |
| **SearchSorting** | `value: string, onChange: (s) => void` | Сортировка результатов |
| **MobileFilterDrawer** | `isOpen: boolean, onClose: () => void` | Фильтры на мобильных |
| **ElectronicsFilters** | `onChange: (f) => void` | Фильтры для электроники |
| **RealEstateFilters** | `onChange: (f) => void` | Фильтры для недвижимости |

**Используемые хуки:**
- `useSearchFilters` - управление фильтрами (URL sync)
- `useDebounce` - debounce для поиска
- `useVoiceSearch` - голосовой поиск
- `useCategoryFilters` - фильтры по категориям
- `useFilterPersistence` - сохранение фильтров

**Архитектура фильтров:**
```typescript
interface SearchFilters {
  query?: string;
  categories?: string[];
  priceRange?: { min?: number; max?: number };
  brands?: string[];
  colors?: string[];
  sizes?: string[];
  conditions?: string[];
  attributes?: Record<string, string[]>; // Динамические
  inStockOnly?: boolean;
  sort?: 'relevance' | 'price_asc' | 'price_desc' | 'date';
  page?: number;
}
```

**Пример:**
```tsx
import { SearchBar } from '@/components/search/SearchBar';
import { SearchFilters } from '@/components/search/SearchFilters';
import { useSearchFilters } from '@/hooks/useSearchFilters';

function SearchPage() {
  const { filters, setFilters } = useSearchFilters();

  return (
    <div className="grid grid-cols-4 gap-4">
      <aside className="col-span-1">
        <SearchFilters filters={filters} onChange={setFilters} />
      </aside>
      <main className="col-span-3">
        <SearchBar initialQuery={filters.query} />
        {/* Results */}
      </main>
    </div>
  );
}
```

---

### 3.3 Корзина и Checkout (cart/, checkout/)

#### Корзина

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **CartItem** | `item: CartItem, onUpdate: (qty) => void` | Товар в корзине |
| **CartSummary** | `items: CartItem[], total: number` | Итоги корзины |
| **EmptyCart** | - | Пустая корзина (CTA) |
| **CartDrawer** | `isOpen: boolean` | Боковая панель корзины |

#### Checkout

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **CheckoutForm** | `cartId: number` | Главная форма оформления |
| **DeliveryAddressForm** | `onSubmit: (addr) => void` | Адрес доставки |
| **PaymentMethodSelector** | `methods: Method[], onSelect: (m) => void` | Выбор метода оплаты |
| **OrderSummary** | `items: Item[], delivery: DeliveryInfo` | Итоговая информация |
| **PromoCodeInput** | `onApply: (code) => void` | Промокод |

**Используемые хуки:**
- `useCartSync` - синхронизация корзины (server ↔ local)
- `useAllSecurePayment` - AllSecure интеграция
- `useBalance` - баланс пользователя

**Пример:**
```tsx
import { CartSummary } from '@/components/cart/CartSummary';
import { useAppSelector } from '@/store/hooks';

function Cart() {
  const cartItems = useAppSelector(state => state.cart.items);
  const total = useAppSelector(state => state.cart.total);

  return <CartSummary items={cartItems} total={total} />;
}
```

---

### 3.4 Категории (categories/)

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **CategoryTree** | `categories: Category[], onSelect: (c) => void` | Дерево категорий |
| **CategoryBreadcrumb** | `categoryId: number` | Хлебные крошки |
| **CategoryFilter** | `selected?: number[], onChange: (ids) => void` | Фильтр по категориям |
| **CategoryCard** | `category: Category` | Карточка категории |
| **NestedCategorySelector** | `level: number, onSelect: (c) => void` | Вложенный селектор |

**Структура Category:**
```typescript
interface Category {
  id: number;
  name: string;
  slug: string;
  parent_id?: number;
  icon?: string;
  attributes?: Attribute[]; // Динамические атрибуты
  children?: Category[];
}
```

**Пример:**
```tsx
import { CategoryTree } from '@/components/categories/CategoryTree';
import { useCategoryFilters } from '@/hooks/useCategoryFilters';

function CategoriesPage() {
  const { categories, selectedId, selectCategory } = useCategoryFilters();

  return (
    <CategoryTree
      categories={categories}
      onSelect={selectCategory}
    />
  );
}
```

---

### 3.5 Карты и геолокация (GIS/)

#### Map Components

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **MapMarker** | `lat: number, lng: number, onClick?: () => void` | Маркер на карте |
| **MapCluster** | `listings: Listing[]` | Кластеризация маркеров |
| **PriceMarker** | `price: number, position: [lat, lng]` | Маркер с ценой |
| **StorefrontMarker** | `store: Store, position: [lat, lng]` | Маркер магазина |
| **MapPopup** | `listing: Listing` | Popup с деталями |
| **MapControls** | `onZoom?: (z) => void` | Контролы карты |

#### Address & Location

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **SmartAddressInput** | `value?: string, onSelect: (addr) => void` | Умный ввод адреса (Mapbox) |
| **AddressConfirmationMap** | `address: Address` | Подтверждение адреса |
| **DistrictFilter** | `selected?: string[], onChange: (d) => void` | Фильтр по районам |
| **RadiusSearchControl** | `center: [lat, lng], radius: number` | Поиск в радиусе |

**Используемые хуки:**
- `useAddressGeocoding` - геокодирование Mapbox
- `useMultilingualGeocoding` - мультиязычное геокодирование
- `useAdvancedGeoFilters` - geo фильтры (polygon, radius)
- `useMapCache` - кэширование карт (IndexedDB)

**Пример:**
```tsx
import { SmartAddressInput } from '@/components/GIS/SmartAddressInput';
import { useAddressGeocoding } from '@/hooks/useAddressGeocoding';

function DeliveryForm() {
  const { geocode, loading } = useAddressGeocoding();

  const handleAddressSelect = async (address: string) => {
    const coords = await geocode(address);
    console.log('Coordinates:', coords);
  };

  return (
    <SmartAddressInput
      value=""
      onSelect={handleAddressSelect}
    />
  );
}
```

---

### 3.6 Админ панель (admin/)

#### Управление

| Компонент | Назначение |
|-----------|------------|
| **AdminDashboard** | Главная панель (статистика) |
| **UserManagement** | Управление пользователями |
| **CategoryEditor** | Редактор категорий (дерево) |
| **LogisticsPanel** | Логистика, доставки |
| **OrdersManagement** | Управление заказами |
| **PaymentsManagement** | Платежи, транзакции |
| **StorefrontManager** | Управление B2C витринами |
| **TranslationEditor** | AI переводы (bulk) |

#### Специфичные компоненты

| Компонент | Назначение |
|-----------|------------|
| **AITranslations** | Массовый AI перевод (Claude) |
| **BulkTranslationManager** | Управление bulk переводами |
| **AuditLogViewer** | Просмотр логов действий |
| **LogisticsFieldsSection** | Логистические поля товаров |

**Требования:**
- Роль: `admin` (middleware: `RequireAuth("admin")`)
- JWT токен через BFF proxy

**Пример:**
```tsx
// Защита роута через middleware (в app/[locale]/admin/layout.tsx)
import { redirect } from 'next/navigation';
import { getSession } from '@/services/auth';

export default async function AdminLayout({ children }) {
  const session = await getSession();

  if (!session?.roles?.includes('admin')) {
    redirect('/');
  }

  return <>{children}</>;
}
```

---

### 3.7 WMS - Warehouse Management (warehouse/)

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **WarehouseDashboard** | `warehouseId: number` | Dashboard владельца склада |
| **InventoryTable** | `items: InventoryItem[]` | Таблица инвентаризации |
| **StaffManagement** | `warehouseId: number` | Управление персоналом |
| **OrderPickingQueue** | `orders: Order[]` | Очередь сборки заказов |
| **PackingStation** | `orderId: number` | Станция упаковки |
| **WarehouseMap** | `zones: Zone[]` | Карта склада (зоны) |
| **StockLevelIndicator** | `current: number, min: number, max: number` | Индикатор остатков |

**Используемые сервисы:**
- `wms.ts` - API клиент WMS
- gRPC: `warehouse:50051` (локально), `warehouse.vondi.rs:50051` (prod)

**Роли:**
- `warehouse_owner` - владелец склада
- `warehouse_worker` - работник
- `warehouse_manager` - менеджер

**Пример:**
```tsx
import { InventoryTable } from '@/components/warehouse/InventoryTable';
import { useApi } from '@/hooks/useApi';
import { getWarehouseInventory } from '@/services/wms';

function InventoryPage({ warehouseId }: { warehouseId: number }) {
  const { data: inventory, loading } = useApi(
    () => getWarehouseInventory(warehouseId),
    { immediate: true }
  );

  if (loading) return <Loader />;

  return <InventoryTable items={inventory || []} />;
}
```

---

### 3.8 Автомобильный раздел (cars/)

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **CarSelector** | `onSelect: (car) => void` | Каскадный селектор (марка→модель→год) |
| **CarSelectorCompact** | `value?: Car` | Компактная версия селектора |
| **VinDecoder** | `onDecode: (vin) => void` | VIN декодер (API) |
| **CarFiltersDrawer** | `filters: CarFilters, onChange: (f) => void` | Фильтры для авто |
| **CarAttributesForm** | `carId: number` | Форма характеристик (двигатель, КПП) |
| **ComparisonTable** | `cars: Car[]` | Сравнение авто (таблица) |
| **ComparisonBar** | `selectedCars: Car[]` | Floating bar сравнения |
| **CarBadges** | `car: Car` | Бейджи (новое, проверено, VIP) |
| **CarQuickViewModal** | `car: Car, isOpen: boolean` | Quick view модал |
| **AutocompleteSearch** | `query: string` | Поиск по маркам/моделям |

**Используемые хуки:**
- `useCarData` - данные автомобилей (brands, models)

**Структура Car:**
```typescript
interface Car {
  id: number;
  brand: string;
  model: string;
  year: number;
  engine?: string; // "2.0 TDI"
  transmission?: 'manual' | 'automatic';
  fuel?: 'petrol' | 'diesel' | 'electric' | 'hybrid';
  mileage?: number;
  price: number;
  vin?: string;
  images: string[];
}
```

**Пример:**
```tsx
import { CarSelector } from '@/components/cars/CarSelector';
import { useCarData } from '@/hooks/useCarData';

function CreateCarListing() {
  const { brands, models, loading } = useCarData();

  const handleCarSelect = (car: { brand: string, model: string, year: number }) => {
    console.log('Selected:', car);
  };

  return (
    <CarSelector
      onSelect={handleCarSelect}
    />
  );
}
```

---

### 3.9 B2C Маркетплейс (B2C/)

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **ProductCard** | `product: Product` | Карточка товара B2C |
| **StorefrontPage** | `storeId: number` | Страница магазина |
| **VariantSelector** | `variants: Variant[], onSelect: (v) => void` | Выбор варианта (цвет, размер) |
| **VariantGenerator** | `product: Product` | Генератор вариантов |
| **VariantManager** | `productId: number` | Управление вариантами |
| **StockIndicator** | `quantity: number, lowStockThreshold: number` | Индикатор остатков |
| **ProductReviews** | `productId: number` | Отзывы на товар |
| **RelatedProducts** | `productId: number` | Похожие товары |

#### Edit Product Components (products/)

**Wizard Mode (7 шагов):**

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **EditProductWizard** | `productId: number` | Wizard редактирования товара |
| **EditBasicInfoStep** | - | Шаг 1: Название, описание |
| **EditCategoryStep** | - | Шаг 2: Выбор категории |
| **EditAttributesStep** | - | Шаг 3: Атрибуты категории (загрузка из API) |
| **EditPhotosStep** | - | Шаг 4: Изображения товара |
| **EditVariantsStep** | - | Шаг 5: Варианты (цвет, размер) |
| **EditPricingStep** | - | Шаг 6: Цены и скидки |
| **EditPreviewStep** | - | Шаг 7: Предпросмотр |

**Tabbed Mode (6 вкладок):**

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **EditProductTabs** | `productId: number` | Tabbed редактирование |
| **EditProductBasicTab** | - | Основная информация |
| **EditProductImagesTab** | - | Управление изображениями |
| **EditProductVariantsTab** | - | Управление вариантами |
| **EditProductAttributesTab** | - | Редактирование атрибутов категории |
| **EditProductPricingTab** | - | Цены и скидки |
| **EditProductSettingsTab** | - | Настройки публикации |

**Контекст EditProductContext:**

```typescript
interface EditProductState {
  step: number;
  isSubmitting: boolean;
  isModified: boolean;
  productId: number;

  // Данные товара
  title: string;
  description: string;
  category: Category | null;
  images: ProductImage[];
  variants: ProductVariant[];
  attributes: Record<number, any>;  // categoryId -> value

  // Цены
  price: number;
  originalPrice?: number;
  discountPercent?: number;

  // Ошибки валидации
  errors: Record<string, string>;
}

// Методы контекста
interface EditProductContextValue {
  state: EditProductState;
  setState: (state: Partial<EditProductState>) => void;
  setAttribute: (id: number, value: any) => void;
  setError: (field: string, message: string) => void;
  clearError: (field: string) => void;
  nextStep: () => void;
  prevStep: () => void;
  saveProduct: () => Promise<void>;
}
```

**Пример использования EditAttributesStep:**
```tsx
// EditAttributesStep загружает атрибуты категории из API
// GET /api/v1/marketplace/categories/${categoryId}/attributes

import { useEditProduct } from '@/contexts/EditProductContext';
import { apiClient } from '@/services/api-client';

function EditAttributesStep() {
  const { state, setAttribute, clearError } = useEditProduct();
  const [attributes, setAttributes] = useState<CategoryAttribute[]>([]);

  // Загрузка атрибутов при смене категории
  useEffect(() => {
    if (state.category?.id) {
      apiClient.get(`/api/v1/marketplace/categories/${state.category.id}/attributes`)
        .then(response => setAttributes(response.data.data || []));
    }
  }, [state.category?.id]);

  // Группировка атрибутов по логическим группам
  const groupAttributes = (): AttributeGroup[] => {
    // basic, technical, condition, accessories, dimensions, other
  };

  // Рендеринг атрибутов по типу
  const renderAttribute = (attr: CategoryAttribute) => {
    switch (attr.attribute_type) {
      case 'text': return <input ... />;
      case 'number': return <input type="number" ... />;
      case 'select': return <select ... />;
      case 'boolean': return <input type="checkbox" ... />;
      case 'multiselect': return <MultiSelectAttribute ... />;
      case 'range': return <RangeAttribute ... />;
    }
  };
}
```

**Структура Product Variant:**
```typescript
interface ProductVariant {
  id: number;
  product_id: number;
  sku: string; // "IPHONE-15-PRO-256-BLUE"
  attributes: {
    color?: string;
    size?: string;
    storage?: string;
  };
  price: number;
  stock_quantity: number;
  images?: string[];
}
```

**Используемые хуки:**
- `useB2CStores` - B2C витрины

**Пример:**
```tsx
import { VariantSelector } from '@/components/B2C/ProductVariants/VariantSelector';

function ProductPage({ product }: { product: Product }) {
  const [selectedVariant, setSelectedVariant] = useState<Variant | null>(null);

  return (
    <div>
      <h1>{product.name}</h1>
      <VariantSelector
        variants={product.variants}
        onSelect={setSelectedVariant}
      />
      {selectedVariant && (
        <p>Price: ${selectedVariant.price}</p>
      )}
    </div>
  );
}
```

---

### 3.10 Общие UI компоненты (common/, ui/)

| Компонент | Props | Назначение |
|-----------|-------|------------|
| **Button** | `variant?: 'primary' \| 'secondary' \| 'danger', loading?: boolean` | Универсальная кнопка |
| **Modal** | `isOpen: boolean, onClose: () => void, title?: string` | Модальное окно |
| **Loader** | `size?: 'sm' \| 'md' \| 'lg', text?: string` | Индикатор загрузки |
| **LazyImage** | `src: string, alt: string, placeholder?: string` | Ленивая загрузка изображений |
| **InfiniteScrollTrigger** | `onLoadMore: () => void, hasMore: boolean` | Бесконечный скролл |
| **VirtualizedList** | `items: T[], renderItem: (item) => ReactNode` | Виртуализация списков |
| **MobileBottomSheet** | `isOpen: boolean, onClose: () => void` | Bottom sheet для мобильных |
| **Pagination** | `total: number, page: number, onChange: (p) => void` | Пагинация |
| **Tabs** | `tabs: Tab[], active: string, onChange: (t) => void` | Табы |
| **Dropdown** | `options: Option[], value?: string, onChange: (o) => void` | Выпадающий список |
| **Toast** | - | Уведомления (react-hot-toast) |
| **ErrorBoundary** | `fallback?: ReactNode` | Обработка ошибок React |
| **GridColumnsToggle** | `columns: number, onChange: (c) => void` | Переключатель колонок грида |
| **ViewToggle** | `view: 'grid' \| 'list', onChange: (v) => void` | Переключатель вида |

**Используемые хуки:**
- `useInfiniteScroll` - бесконечный скролл
- `useViewPreference` - сохранение view preference
- `useGridColumns` - адаптивные grid columns
- `useObjectURL` - создание blob URLs

**Пример:**
```tsx
import { Button } from '@/components/common/Button';
import { Modal } from '@/components/common/Modal';
import { useState } from 'react';

function MyComponent() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      <Button variant="primary" onClick={() => setIsOpen(true)}>
        Открыть модал
      </Button>

      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)} title="Заголовок">
        <p>Содержимое модального окна</p>
      </Modal>
    </>
  );
}
```

---

## 4. Custom Hooks (hooks/)

### 4.1 Data Fetching

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useApi** | `apiCall: () => Promise<T>, options?` | `{ data, error, loading, execute, reset }` | Универсальный API хук |
| **usePaginatedData** | `fetchFn: (page) => Promise<T[]>, initialPage?` | `{ data, page, loading, hasMore, loadMore }` | Пагинация |
| **useInfiniteScroll** | `loadMore: () => void, hasMore: boolean` | `{ ref, loading }` | Бесконечный скролл |
| **useWebSocket** | `url: string, options?` | `{ send, close, lastMessage, readyState }` | WebSocket подключение |

**Пример useApi:**
```tsx
import { useApi } from '@/hooks/useApi';
import { getCategories } from '@/services/category';

function CategoriesPage() {
  const { data: categories, loading, error, execute } = useApi(
    getCategories,
    { immediate: true } // Загрузить сразу
  );

  if (loading) return <Loader />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <ul>
      {categories?.map(cat => <li key={cat.id}>{cat.name}</li>)}
    </ul>
  );
}
```

---

### 4.2 Forms & Validation

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useAuthForm** | `type: 'login' \| 'register'` | `{ values, errors, handleChange, handleSubmit }` | Формы аутентификации |
| **useFormValidation** | `schema: ValidationSchema` | `{ validate, errors }` | Валидация форм |

---

### 4.3 Search & Filters

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useSearchFilters** | - | `{ filters, setFilters, reset }` | Фильтры поиска (URL sync) |
| **useCategoryFilters** | `categoryId?: number` | `{ categories, selectedId, selectCategory }` | Фильтры по категориям |
| **useFilterPersistence** | `key: string` | `{ filters, saveFilters, loadFilters }` | Сохранение фильтров (localStorage) |
| **useSearchWithAttributes** | `categoryId?: number` | `{ attributes, filters, applyFilter }` | Поиск с динамическими атрибутами |
| **useVoiceSearch** | `onResult: (text) => void` | `{ start, stop, isListening, transcript }` | Голосовой поиск (Web Speech API) |

**Пример useSearchFilters:**
```tsx
import { useSearchFilters } from '@/hooks/useSearchFilters';

function SearchPage() {
  const { filters, setFilters } = useSearchFilters();

  const handlePriceChange = (min: number, max: number) => {
    setFilters({ ...filters, priceRange: { min, max } });
  };

  return (
    <div>
      <p>Query: {filters.query}</p>
      <p>Price: {filters.priceRange?.min} - {filters.priceRange?.max}</p>
    </div>
  );
}
```

---

### 4.4 Category Detection & AI

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useCategoryDetection** | `description?: string, images?: File[]` | `{ detectCategory, suggestedCategory, loading }` | AI определение категории |
| **useAttributeAutocomplete** | `categoryId: number, field: string` | `{ suggestions, loading, search }` | Автодополнение атрибутов |
| **useAttributesPagination** | `attributes: Attribute[], perPage?` | `{ visibleAttrs, page, nextPage, prevPage }` | Пагинация атрибутов |

**Пример useCategoryDetection:**
```tsx
import { useCategoryDetection } from '@/hooks/useCategoryDetection';

function SmartListingForm() {
  const { detectCategory, suggestedCategory, loading } = useCategoryDetection();

  const handleDescriptionChange = async (desc: string) => {
    const category = await detectCategory(desc);
    console.log('Suggested category:', category);
  };

  return (
    <textarea onChange={(e) => handleDescriptionChange(e.target.value)} />
  );
}
```

---

### 4.5 Maps & Location

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useAddressGeocoding** | - | `{ geocode, reverse, loading, error }` | Геокодирование Mapbox |
| **useMultilingualGeocoding** | `locale: string` | `{ geocode, suggestions }` | Мультиязычное геокодирование |
| **useAdvancedGeoFilters** | - | `{ radius, polygon, applyFilter }` | Geo фильтры (radius, polygon) |
| **useMapCache** | `cacheKey: string` | `{ getCached, setCache, clearCache }` | Кэширование карт (IndexedDB) |

**Пример useAddressGeocoding:**
```tsx
import { useAddressGeocoding } from '@/hooks/useAddressGeocoding';

function AddressForm() {
  const { geocode, reverse, loading } = useAddressGeocoding();

  const handleGeocode = async (address: string) => {
    const coords = await geocode(address);
    console.log('Coordinates:', coords); // { lat: 44.8, lng: 20.4 }
  };

  const handleReverse = async (lat: number, lng: number) => {
    const address = await reverse(lat, lng);
    console.log('Address:', address); // "Bulevar Mihajla Pupina 10, Beograd"
  };

  return (
    <input
      type="text"
      placeholder="Enter address"
      onChange={(e) => handleGeocode(e.target.value)}
    />
  );
}
```

---

### 4.6 Commerce & Payments

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useCartSync** | - | Автоматическая синхронизация | Синхронизация корзины (server ↔ local) |
| **useB2CStores** | - | `{ stores, loading, fetchStores }` | B2C витрины |
| **useBalance** | - | `{ balance, loading, topUp, withdraw }` | Баланс пользователя |
| **useAllSecurePayment** | `orderId: number` | `{ initiatePayment, checkStatus }` | AllSecure интеграция |
| **useAllSecurePaymentHelpers** | - | `{ formatAmount, validateCard }` | Вспомогательные функции AllSecure |

**Пример useCartSync:**
```tsx
import { useCartSync } from '@/hooks/useCartSync';
import { useAuth } from '@/contexts/AuthContext';

function App() {
  const { isAuthenticated } = useAuth();

  // Автоматически синхронизирует корзину при логине
  useCartSync();

  return <div>App content</div>;
}
```

---

### 4.7 Chat & Messaging

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useChat** | `roomId?: number` | `{ messages, send, typing, participants }` | Чат (WebSocket) |
| **useIncomingContactRequests** | - | `{ requests, accept, reject, loading }` | Входящие контакты |

---

### 4.8 User Engagement

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useReviews** | `listingId: number` | `{ reviews, addReview, loading }` | Отзывы |
| **useSubscription** | - | `{ plan, upgrade, cancel }` | Подписки |
| **useAnalytics** | - | `{ track, page, identify }` | Аналитика (события) |
| **useBehaviorTracking** | - | `{ trackClick, trackView }` | Отслеживание поведения |
| **useABTest** | `testName: string` | `{ variant, isVariant }` | A/B тестирование |

**Пример useABTest:**
```tsx
import { useABTest } from '@/hooks/useABTest';

function HomePage() {
  const { variant, isVariant } = useABTest('hero-redesign');

  return (
    <div>
      {isVariant('A') && <HeroOld />}
      {isVariant('B') && <HeroNew />}
    </div>
  );
}
```

---

### 4.9 UI/UX Utilities

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useDebounce** | `value: T, delay?: number` | `debouncedValue: T` | Debounce значений |
| **useLocalStorage** | `key: string, initialValue: T` | `[value, setValue]` | localStorage с типизацией |
| **useObjectURL** | `file?: File` | `url: string \| null` | Создание blob URLs (preview) |
| **useViewPreference** | `key: string` | `{ view, setView }` | Сохранение view preference (grid/list) |
| **useGridColumns** | `minWidth: number` | `columns: number` | Адаптивные grid columns |
| **useCountAnimation** | `end: number, duration?: number` | `count: number` | Анимация чисел (counter) |
| **useProgressiveLoading** | `images: string[]` | `{ loadedImages, progress }` | Прогрессивная загрузка изображений |

**Пример useDebounce:**
```tsx
import { useDebounce } from '@/hooks/useDebounce';
import { useState } from 'react';

function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500); // 500ms задержка

  useEffect(() => {
    if (debouncedQuery) {
      // API call только после 500ms без ввода
      searchAPI(debouncedQuery);
    }
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

---

### 4.10 Mobile Optimization

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useMobileOptimization** | - | `{ isMobile, isTablet, isDesktop }` | Определение устройства |
| **useTouchGestures** | `ref: RefObject` | `{ onSwipe, onPinch }` | Жесты (swipe, pinch) |
| **useHapticFeedback** | - | `{ vibrate, light, medium, heavy }` | Haptic feedback (вибрация) |
| **useNativeShare** | - | `{ share, canShare }` | Native share API |

**Пример useNativeShare:**
```tsx
import { useNativeShare } from '@/hooks/useNativeShare';

function ListingPage({ listing }: { listing: Listing }) {
  const { share, canShare } = useNativeShare();

  const handleShare = async () => {
    if (canShare) {
      await share({
        title: listing.title,
        text: listing.description,
        url: window.location.href,
      });
    }
  };

  return (
    <button onClick={handleShare} disabled={!canShare}>
      Поделиться
    </button>
  );
}
```

---

### 4.11 Data Management

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useListingDraft** | `listingId?: number` | `{ draft, saveDraft, clearDraft }` | Черновики объявлений (auto-save) |
| **useCarData** | - | `{ brands, models, loading }` | Автомобильные данные |

---

### 4.12 Configuration

| Hook | Параметры | Возвращает | Назначение |
|------|-----------|-----------|------------|
| **useConfig** | - | `{ apiUrl, s3Url, mapboxToken }` | Runtime конфигурация (env) |

---

## 5. Best Practices

### 5.1 Компоненты

**✅ DO:**
- Разделяй компоненты по принципу Single Responsibility
- Используй TypeScript для props
- Добавляй JSDoc комментарии для сложных компонентов
- Мемоизируй тяжелые вычисления через `useMemo`
- Используй `React.memo` для предотвращения лишних рендеров

**❌ DON'T:**
- Не создавай компоненты > 500 строк (разбивай!)
- Не используй inline стили (только Tailwind classes)
- Не дублируй логику (выноси в хуки)
- Не забывай про accessibility (aria-labels)

### 5.2 Hooks

**✅ DO:**
- Всегда проверяй зависимости в useEffect
- Используй `useCallback` для функций в зависимостях
- Добавляй cleanup функции в useEffect
- Типизируй возвращаемые значения

**❌ DON'T:**
- Не вызывай хуки условно
- Не забывай про cleanup (unsubscribe, clearTimeout)
- Не создавай бесконечные циклы в useEffect

### 5.3 Стилизация

**Tailwind classes приоритет:**
```tsx
// ✅ Правильно - Tailwind utility classes
<div className="flex items-center gap-4 p-4 bg-white rounded-lg shadow">

// ❌ Неправильно - inline styles
<div style={{ display: 'flex', padding: '16px', background: 'white' }}>
```

**DaisyUI компоненты:**
```tsx
// ✅ Используй DaisyUI где возможно
<button className="btn btn-primary">Click</button>

// ❌ Не создавай кастомные кнопки без необходимости
<button className="px-4 py-2 bg-blue-500 text-white rounded">Click</button>
```

### 5.4 API Calls

**Всегда через BFF proxy:**
```typescript
// ✅ Правильно
import { apiClient } from '@/services/api-client';
await apiClient.get('/marketplace/listings');

// ❌ НЕПРАВИЛЬНО - прямой вызов backend
fetch('http://localhost:3000/api/v1/marketplace/listings');
```

---

## 6. Testing

### 6.1 Component Tests

```tsx
// src/components/common/__tests__/Button.test.tsx
import { render, screen } from '@testing-library/react';
import { Button } from '../Button';

describe('Button', () => {
  it('renders with text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('calls onClick when clicked', () => {
    const onClick = jest.fn();
    render(<Button onClick={onClick}>Click</Button>);

    screen.getByText('Click').click();
    expect(onClick).toHaveBeenCalledTimes(1);
  });
});
```

### 6.2 Hook Tests

```tsx
// src/hooks/__tests__/useDebounce.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { useDebounce } from '../useDebounce';

describe('useDebounce', () => {
  it('debounces value changes', async () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: 'initial' } }
    );

    expect(result.current).toBe('initial');

    rerender({ value: 'updated' });
    expect(result.current).toBe('initial'); // Еще не обновлено

    await waitFor(() => {
      expect(result.current).toBe('updated'); // Обновлено после задержки
    }, { timeout: 600 });
  });
});
```

---

## 7. Performance Optimization

### 7.1 Lazy Loading

```tsx
import dynamic from 'next/dynamic';

// Lazy load тяжелых компонентов
const MapComponent = dynamic(() => import('@/components/GIS/Map/MapComponent'), {
  ssr: false, // Отключить SSR для карт
  loading: () => <Loader />,
});
```

### 7.2 Image Optimization

```tsx
import Image from 'next/image';

// Next.js Image компонент (автоматическая оптимизация)
<Image
  src="/images/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority // Для hero изображений
/>

// Или LazyImage для галерей
import { LazyImage } from '@/components/common/LazyImage';

<LazyImage
  src={listing.image}
  alt={listing.title}
  placeholder="/images/placeholder.png"
/>
```

### 7.3 Мемоизация

```tsx
import { memo, useMemo, useCallback } from 'react';

// Мемоизация компонента
const ListingCard = memo(({ listing }: { listing: Listing }) => {
  return <div>{listing.title}</div>;
});

// Мемоизация вычислений
const filteredListings = useMemo(() => {
  return listings.filter(l => l.price > 100);
}, [listings]);

// Мемоизация функций
const handleClick = useCallback(() => {
  console.log('Clicked');
}, []);
```

---

## 8. Internationalization (i18n)

### 8.1 Использование переводов

```tsx
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
```

### 8.2 Структура переводов

```json
// messages/en/common.json
{
  "welcome": "Welcome to Vondi",
  "description": "You have {count} items"
}

// messages/sr/common.json
{
  "welcome": "Dobrodošli na Vondi",
  "description": "Imate {count} stavki"
}
```

---

## 9. Accessibility (a11y)

### 9.1 ARIA Labels

```tsx
// ✅ Правильно - с aria-label
<button aria-label="Close modal" onClick={onClose}>
  <XIcon />
</button>

// ✅ Правильно - с aria-describedby
<input
  id="email"
  aria-describedby="email-help"
/>
<span id="email-help">Введите ваш email</span>
```

### 9.2 Keyboard Navigation

```tsx
// Поддержка Escape для модалов
useEffect(() => {
  const handleEscape = (e: KeyboardEvent) => {
    if (e.key === 'Escape') onClose();
  };

  document.addEventListener('keydown', handleEscape);
  return () => document.removeEventListener('keydown', handleEscape);
}, [onClose]);
```

---

## 10. Troubleshooting

### 10.1 Общие проблемы

**Компонент не обновляется:**
- Проверь зависимости в `useEffect`
- Используй `useCallback` для функций

**Бесконечный цикл в useEffect:**
- Проверь массив зависимостей
- Используй `useMemo` для объектов/массивов

**Hydration mismatch:**
- Проверь SSR vs CSR различия
- Используй `useEffect` для client-only кода

---

**Дата создания:** 2025-12-21
**Дата обновления:** 2025-12-21
**Версия документа:** 1.1
**Компонентов:** 470+
**Хуков:** 44

**Changelog v1.1:**
- Добавлена документация по Edit Product Components (Wizard и Tabbed режимы)
- Добавлено описание EditProductContext
- Описаны атрибутные компоненты: EditAttributesStep, EditProductAttributesTab
