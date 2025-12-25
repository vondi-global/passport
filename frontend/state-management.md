# Frontend State Management - Vondi Platform

**Документ:** Архитектура управления состоянием во frontend приложении
**Версия:** 1.0
**Дата:** 2025-12-21
**Статус:** Production Ready

---

## Оглавление

1. [Архитектура](#архитектура)
2. [Redux Store](#redux-store)
3. [Слайсы Redux](#слайсы-redux)
4. [API Client](#api-client)
5. [Middleware](#middleware)
6. [Паттерны использования](#паттерны-использования)
7. [Синхронизация состояний](#синхронизация-состояний)

---

## Архитектура

### Гибридная модель: Redux + Server State

**Философия:**
```
Client State (Redux)          Server State (API Calls)
     ↓                               ↓
  UI State                     Backend Data
  Selections                   Lists/Details
  Temp Data                    Async Operations
```

**Разделение ответственности:**
- **Redux**: Client-side состояние, UI state, кэши, WebSocket данные
- **API Client**: Прямые обращения к backend через BFF proxy
- **Middleware**: WebSocket, side effects, синхронизация

### Технологический стек

| Технология | Версия | Назначение |
|------------|--------|------------|
| Redux Toolkit | Latest | State management |
| @reduxjs/toolkit | Latest | Async thunks, slices |
| react-redux | Latest | React bindings |
| BFF Proxy | Custom | Backend integration |
| WebSocket | Native | Real-time updates |

---

## Redux Store

### Конфигурация

**Файл:** `/frontend/src/store/index.ts`

```typescript
export const store = configureStore({
  reducer: {
    chat: chatReducer,                    // WebSocket чаты
    reviews: reviewsReducer,               // Отзывы и рейтинги
    b2cStores: b2cStoresReducer,          // B2C витрины
    import: importReducer,                 // Импорт товаров
    products: productReducer,              // Управление товарами
    payment: paymentReducer,               // Платежи
    cart: cartReducer,                     // Корзина покупок
    localCart: localCartReducer,           // Локальная корзина
    categories: categoriesReducer,         // Категории
    universalCompare: universalCompareReducer, // Сравнение товаров
    favorites: favoritesReducer,           // Избранное
    savedSearches: savedSearchesReducer,   // Сохраненные поиски
    categoryProposals: categoryProposalsReducer, // Предложения категорий
    delivery: deliverySlice,               // Доставка
    checkout: checkoutReducer,             // Оформление заказа
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: [/* WebSocket, File actions */],
        ignoredPaths: [/* ws, files, Sets */],
      },
    }).concat(websocketMiddleware),
});
```

### Middleware Stack

1. **Immutability Check** (dev only)
2. **Serializable Check** (dev only, custom config)
3. **Thunk** (async operations)
4. **WebSocket Middleware** (custom)

### TypeScript Types

```typescript
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// Custom hooks
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

---

## Слайсы Redux

### 1. Chat Slice

**Назначение:** WebSocket чаты, real-time сообщения
**Файл:** `store/slices/chatSlice.ts`

#### State Schema

```typescript
interface ChatState {
  // Данные
  chats: MarketplaceChat[];
  currentChat: MarketplaceChat | null;
  messages: Record<number, MarketplaceMessage[]>; // chatId -> messages
  unreadCount: number;

  // WebSocket
  ws: WebSocket | null;
  typingUsers: Record<number, number[]>; // chatId -> userIds
  onlineUsers: number[];
  userLastSeen: Record<number, string>;
  currentUserId: number | null;

  // Connection State
  connectionStatus: 'connecting' | 'connected' | 'disconnected' | 'error';
  connectionError?: string;
  reconnectAttempts: number;
  lastConnectedAt?: string;

  // Пагинация
  chatsPage: number;
  messagesPage: Record<number, number>;
  hasMoreChats: boolean;
  hasMoreMessages: Record<number, boolean>;

  // Загрузка файлов
  uploadingFiles: Record<string, UploadingFile>;
  attachments: Record<number, ChatAttachment[]>;

  // Offline Queue
  messageQueue: SendMessagePayload[];

  // Message Status
  messageStatuses: Record<number, MessageStatus>; // messageId -> status
  recentMessageIds: number[]; // Deduplication (last 1000)
}
```

#### Actions

**Async Thunks:**
- `loadChats(page)` - загрузка списка чатов
- `loadMessages(chatId, page)` - загрузка сообщений
- `sendMessage(payload)` - отправка сообщения
- `markMessagesAsRead(chatId, messageIds)` - пометить прочитанными
- `uploadFiles(messageId, files)` - загрузка файлов
- `archiveChat(chatId)` - архивация чата

**Sync Reducers:**
- `setCurrentChat` - выбор текущего чата
- `handleNewMessage` - новое сообщение через WebSocket
- `handleMessageRead` - сообщение прочитано
- `handleUserOnline/Offline` - статус пользователя
- `setUserTyping` - индикатор печати
- `updateUploadProgress` - прогресс загрузки
- `clearAllData` - очистка при логауте

#### Selectors

```typescript
selectChats(state)
selectCurrentChat(state)
selectMessages(state, chatId)
selectUnreadCount(state)
selectOnlineUsers(state)
selectConnectionStatus(state)
selectMessageStatuses(state)
```

#### Паттерны

**WebSocket Integration:**
```typescript
// Инициализация (через middleware)
dispatch({ type: 'chat/initWebSocket', payload: { getCurrentUserId } });

// Обработка событий (middleware -> actions)
ws.onmessage -> handleNewMessage(message)
              -> handleUserOnline(userId)
              -> handleMessageRead(data)
```

**Message Deduplication:**
```typescript
// Хранит последние 1000 message IDs
recentMessageIds.push(message.id);
if (recentMessageIds.length > 1000) {
  recentMessageIds = recentMessageIds.slice(-1000);
}
```

**File Upload (non-serializable):**
```typescript
// Redux хранит только метаданные
uploadingFiles[fileId] = { id, name, size, type, progress, status };

// Файлы хранятся отдельно
fileUploadManager.addFile(fileId, file);
```

---

### 2. Cart Slice

**Назначение:** Корзина покупок (B2C)
**Файл:** `store/slices/cartSlice.ts`

#### State Schema

```typescript
interface CartState {
  cart: ShoppingCart | null;       // Активная корзина
  allCarts: ShoppingCart[];        // Все корзины пользователя
  loading: boolean;
  error: string | null;
}
```

#### Actions

**Async Thunks:**
- `fetchCart(storefrontId)` - загрузка корзины
- `fetchUserCarts(userId)` - все корзины пользователя
- `addToCart({ storefrontId, item })` - добавить товар
- `updateCartItem({ storefrontId, itemId, data })` - обновить
- `removeFromCart({ storefrontId, itemId })` - удалить
- `clearCart(storefrontId)` - очистить корзину

**Sync Reducers:**
- `resetCart` - сбросить корзину
- `clearCartOnLogout` - очистка при выходе
- `clearError` - сбросить ошибку

#### LocalStorage Sync

```typescript
// Ключи по пользователю
const CART_STORAGE_KEY_PREFIX = 'vondi_cart';

getCartStorageKey(userId): string {
  return userId
    ? `${CART_STORAGE_KEY_PREFIX}_user_${userId}`
    : `${CART_STORAGE_KEY_PREFIX}_anon`;
}

// Сохранение
saveCartToStorage(cart, userId);

// Загрузка
loadCartFromStorage(userId);
```

#### Selectors

```typescript
selectCart(state) → cart
selectAllCarts(state) → allCarts
selectCartItemsCount(state) → total items
selectAllCartsItemsCount(state) → all carts total
selectCartTotal(state) → total price
```

---

### 3. Favorites Slice

**Назначение:** Избранные товары
**Файл:** `store/slices/favoritesSlice.ts`

#### State Schema

```typescript
interface FavoritesState {
  items: C2CListing[];           // Полные объекты
  itemIds: Set<number>;          // Быстрая проверка
  loading: boolean;
  error: string | null;
  count: number;
}
```

#### Optimistic Updates

```typescript
// Добавление в избранное
addToFavorites.pending: (state, action) => {
  const id = action.meta.arg.id;
  state.itemIds.add(id);  // Optimistic
  state.count += 1;
}

addToFavorites.fulfilled: (state, action) => {
  // Confirm
}

addToFavorites.rejected: (state, action) => {
  const id = action.meta.arg.id;
  state.itemIds.delete(id);  // Rollback
  state.count = Math.max(0, state.count - 1);
}
```

#### Actions

- `fetchFavorites()` - загрузка списка
- `fetchFavoritesCount()` - только счетчик
- `addToFavorites({ id, type })` - добавить
- `removeFromFavorites({ id, type })` - удалить
- `checkIfInFavorites(id)` - проверка
- `toggleFavoriteOptimistic(id)` - toggle без API

---

### 4. Checkout Slice

**Назначение:** Multi-storefront checkout flow
**Файл:** `store/slices/checkoutSlice.ts`

#### State Schema

```typescript
interface CheckoutState {
  // Wizard Steps
  currentStep: CheckoutStep; // 'contact' | 'address' | 'delivery' | 'review'

  // Contact
  contactInfo: ContactInfo | null;

  // Address
  selectedAddressId: number | null;
  addresses: Address[];
  addressesLoading: boolean;

  // Delivery (multi-storefront)
  deliveryByStorefront: Record<number, SelectedDeliveryMethod>;
  availableMethods: Record<number, DeliveryMethod[]>;
  methodsLoading: Record<number, boolean>;
  methodsError: Record<number, string | null>;

  // Pickup Points
  pickupPoints: PickupPoint[];
  pickupPointsLoading: boolean;

  // Order Creation
  isSubmitting: boolean;
  orderCreated: CreateOrderResponse | null;
}
```

#### Multi-Storefront Pattern

```typescript
// Для каждой витрины отдельный метод доставки
deliveryByStorefront: {
  [storefrontId: number]: {
    methodId: string;
    provider: string;
    priceCents: number;
    pickupPointId?: string;
    estimatedDays: { min: number; max: number };
  }
}
```

#### Actions

**Navigation:**
- `setCurrentStep(step)` - установить шаг
- `goToNextStep()` - следующий шаг
- `goToPreviousStep()` - предыдущий шаг

**Data:**
- `fetchCheckoutProfile()` - загрузить профиль + адреса
- `fetchAddresses()` - только адреса
- `fetchDeliveryMethods({ storefrontId, ... })` - методы доставки
- `fetchPickupPoints(request)` - точки самовывоза
- `submitOrder(orderData)` - создать заказ

**Setters:**
- `setContactInfo(info)`
- `setSelectedAddress(id)`
- `setDeliveryMethod({ storefrontId, method })`
- `setPickupPoint({ storefrontId, pickupPointId })`

#### Selectors

```typescript
selectCheckoutStep(state)
selectContactInfo(state)
selectSelectedAddress(state)
selectAddresses(state)
selectDeliveryForStorefront(state, storefrontId)
selectAvailableMethodsForStorefront(state, storefrontId)
selectTotalShippingCost(state) // Sum across all storefronts
selectIsCheckoutReady(state) // Validation
```

---

### 5. Delivery Slice

**Назначение:** Delivery providers & calculations cache
**Файл:** `store/slices/deliverySlice.ts`

#### State Schema

```typescript
interface DeliveryState {
  // Providers
  providers: DeliveryProvider[];
  providersLoading: boolean;
  providersError: string | null;

  // Selected quotes by storefront
  selectedQuotes: Record<string, DeliveryQuote>;

  // Calculation cache with TTL
  calculations: Record<string, CachedCalculation>;
  calculationsLoading: Record<string, boolean>;
  calculationsError: Record<string, string>;

  // Tracking
  tracking: Record<string, TrackingInfo>;
  trackingLoading: Record<string, boolean>;
  trackingError: Record<string, string>;
}

interface CachedCalculation {
  data: CalculationResponse;
  timestamp: number;
  params: CalculationRequest;
}
```

#### Cache Strategy

```typescript
const CACHE_TTL = 5 * 60 * 1000; // 5 минут

function isCacheValid(cached: CachedCalculation, newParams: CalculationRequest): boolean {
  // Check TTL
  if (Date.now() - cached.timestamp > CACHE_TTL) return false;

  // Compare params
  return JSON.stringify(cached.params) === JSON.stringify(newParams);
}

// Cache key
function getCacheKey(request: CalculationRequest): string {
  return JSON.stringify({
    from: request.from_location,
    to: request.to_location,
    items: request.items,
    provider_id: request.provider_id,
  });
}
```

#### Actions

- `fetchProviders()` - загрузить провайдеров
- `calculateRate({ request, bypassCache? })` - расчет доставки
- `trackShipment(trackingToken)` - отслеживание

---

### 6. Categories Slice

**Назначение:** Категории товаров с i18n
**Файл:** `store/slices/categoriesSlice.ts`

#### State Schema

```typescript
interface CategoriesState {
  categories: Category[];
  popularCategories: Category[];
  isLoadingCategories: boolean;
  isLoadingPopular: boolean;
  error: string | null;
  lastLocale: string | null; // Кэш по локали
}

interface Category {
  id: string;
  slug: string;
  name: string; // Локализованное имя
  description?: string;
  parent_id?: string;
  iconName?: string; // 'BsHouseDoor'
  color?: string;
  count?: number;
  translations?: {
    en: string;
    ru: string;
    sr: string;
  };
}
```

#### Локализация

```typescript
fetchPopularCategories({ locale, forceRefresh? })

// Проверка кэша по локали
if (!forceRefresh && state.lastLocale === locale && state.popularCategories.length > 0) {
  return state.popularCategories;
}

// Извлечение локализованного имени
const translations = JSON.parse(cat.translations);
const localizedName = translations[locale] || translations.sr || cat.name;
```

---

### 7. Product Slice

**Назначение:** Управление B2C товарами
**Файл:** `store/slices/productSlice.ts`

#### State Schema

```typescript
interface ProductState {
  products: B2CProduct[];
  selectedIds: number[];
  loading: boolean;
  error: string | null;

  // Filters
  filters: {
    search: string;
    categoryId: string | null;
    minPrice: number | null;
    maxPrice: number | null;
    stockStatus: 'all' | 'in_stock' | 'low_stock' | 'out_of_stock';
    isActive: boolean | null;
  };

  // Pagination
  pagination: {
    page: number;
    limit: number;
    total: number;
    hasMore: boolean;
  };

  // Bulk Operations
  bulkOperation: {
    isProcessing: boolean;
    progress: number;
    total: number;
    errors: BulkOperationError[];
    successCount: number;
    currentOperation: 'idle' | 'delete' | 'update' | 'status' | 'export';
  };

  // UI
  ui: {
    isSelectMode: boolean;
    viewMode: 'grid' | 'list' | 'table';
    sortBy: 'name' | 'price' | 'created_at' | 'stock_quantity';
    sortOrder: 'asc' | 'desc';
  };
}
```

#### Bulk Operations

```typescript
bulkDeleteProducts({ storefrontSlug, productIds })
bulkUpdateStatus({ storefrontSlug, productIds, isActive })
exportProducts({ storefrontSlug, productIds, format })
```

---

### 8. Import Slice

**Назначение:** Импорт товаров (XML, CSV, ZIP)
**Файл:** `store/slices/importSlice.ts`

#### State Schema

```typescript
interface ImportState {
  jobs: ImportJob[];
  currentJob: ImportJob | null;
  isUploading: boolean;
  uploadProgress: UploadProgress | null;
  validationErrors: ValidationError[];
  formats: ImportFormat[] | null;

  // Preview
  previewData: any | null;
  isPreviewLoading: boolean;
  previewError: string | null;

  // UI
  selectedFiles: File[];
  importUrl: string;
  selectedFileType: 'xml' | 'csv' | 'zip' | '';
  updateMode: 'create_only' | 'update_only' | 'upsert';
  categoryMappingMode: 'auto' | 'manual' | 'skip';

  // Modals
  isImportModalOpen: boolean;
  isJobDetailsModalOpen: boolean;
  isErrorsModalOpen: boolean;

  // Enhanced Import (AI-assisted)
  analysisFile: File | null;
  categoryAnalysis: CategoryAnalysis | null;
  attributeAnalysis: AttributeAnalysis | null;
  variantDetection: VariantDetection | null;
  isAnalyzing: boolean;
  analysisProgress: number;

  // User modifications
  approvedMappings: CategoryMapping[];
  customMappings: Record<string, string>;
  selectedAttributes: string[];
  approvedVariantGroups: VariantGroup[];
}
```

#### Actions

- `fetchImportFormats()` - форматы импорта
- `fetchImportJobs({ storefrontId, status?, limit? })` - история
- `importFromUrl({ storefrontId, request })` - импорт по URL
- `importFromFile({ storefrontId, file, options })` - файл
- `analyzeImportFile()` - AI анализ
- `analyzeCategories()` - категории AI
- `detectVariants()` - варианты AI

---

### 9. Review Slice

**Назначение:** Отзывы и рейтинги
**Файл:** `store/slices/reviewsSlice.ts`

**Actions:**
- `fetchReviews(listingId)`
- `submitReview({ listingId, rating, comment })`
- `updateReview({ reviewId, data })`
- `deleteReview(reviewId)`

---

### 10. Payment Slice

**Назначение:** Платежные методы
**Файл:** `store/slices/paymentSlice.ts`

**Actions:**
- `fetchPaymentMethods()`
- `selectPaymentMethod(methodId)`
- `processPayment(data)`

---

### 11-15. Дополнительные Slices

- **localCartSlice** - Локальная корзина (анонимные пользователи)
- **universalCompareSlice** - Сравнение товаров
- **savedSearchesSlice** - Сохраненные поиски
- **categoryProposalsSlice** - Предложения категорий
- **b2cStoreSlice** - Управление B2C витринами

---

## API Client

**Файл:** `services/api-client.ts`

### Архитектура BFF Proxy

```
Browser → /api/v2/* (Next.js BFF) → /api/v1/* (Backend)
         └─ httpOnly cookies    └─ Authorization: Bearer <JWT>
```

### Class ApiClient

```typescript
class ApiClient {
  private defaultTimeout = 30000;
  private defaultRetries = 3;

  // Базовый URL (всегда BFF proxy)
  private getBaseUrl(): string {
    if (typeof window === 'undefined') {
      // SSR: http://localhost:3001/api/v2
      return `${process.env.NEXT_PUBLIC_FRONTEND_URL}/api/v2`;
    }
    // Browser: /api/v2
    return '/api/v2';
  }

  // Основной метод
  async request<T>(endpoint: string, options: ApiClientOptions): Promise<ApiResponse<T>> {
    // Remove /api/v1/ prefix if present
    if (endpoint.startsWith('/api/v1/')) {
      endpoint = endpoint.replace('/api/v1/', '/');
    }

    const url = `${this.getBaseUrl()}${endpoint}`;

    // Добавляем заголовки
    headers.set('Content-Type', 'application/json');
    headers.set('Accept-Language', locale);

    // Credentials (cookies including JWT)
    credentials: 'include'

    // Retry logic + timeout
    const response = await this.fetchWithRetries(url, options, timeout, retries);

    return { data, status, headers, ok };
  }

  // Удобные методы
  async get<T>(endpoint, options) { ... }
  async post<T>(endpoint, data, options) { ... }
  async put<T>(endpoint, data, options) { ... }
  async patch<T>(endpoint, data, options) { ... }
  async delete<T>(endpoint, options) { ... }
  async upload<T>(endpoint, formData, options) { ... }
}

export const apiClient = new ApiClient();
```

### Retry Strategy

```typescript
async fetchWithRetries(url, options, timeout, retries): Promise<Response> {
  for (let i = 0; i <= retries; i++) {
    try {
      const response = await fetchWithTimeout(url, options, timeout);

      // Не повторяем client errors (4xx)
      if (response.ok || (response.status >= 400 && response.status < 500)) {
        return response;
      }

      // Для server errors (5xx) повторяем
      if (i < retries) {
        await delay(1000 * (i + 1)); // Exponential backoff
        continue;
      }
    } catch (error) {
      if (i < retries) {
        await delay(1000 * (i + 1));
        continue;
      }
      throw error;
    }
  }
}
```

### Next.js Cache Control

```typescript
interface ApiClientOptions {
  timeout?: number;
  retries?: number;
  nextCache?: RequestCache | number; // 'force-cache' | 'no-store' | 60
  nextTags?: string[];
}

// Пример
await apiClient.get('/marketplace/categories', {
  nextCache: 3600, // Revalidate через 1 час
  nextTags: ['categories'],
});
```

---

## Middleware

### WebSocket Middleware

**Файл:** `store/middleware/websocketMiddleware.ts`

#### Жизненный цикл

```typescript
1. Dispatch initWebSocket({ getCurrentUserId })
2. Middleware перехватывает action
3. Создается WebSocket connection
4. Устанавливаются обработчики событий
5. Запускается heartbeat (ping/pong)
6. Reconnection logic при обрыве
```

#### Reconnection Strategy

```typescript
const MAX_RECONNECT_ATTEMPTS = 10;
const INITIAL_RECONNECT_DELAY = 1000;
const MAX_RECONNECT_DELAY = 30000;

function getReconnectDelay(attempts: number): number {
  const delay = Math.min(
    INITIAL_RECONNECT_DELAY * Math.pow(2, attempts),
    MAX_RECONNECT_DELAY
  );
  const jitter = Math.random() * 1000; // Anti-thundering herd
  return delay + jitter;
}
```

#### Event Handlers

```typescript
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);

  switch (data.type) {
    case 'new_message':
      dispatch(handleNewMessage(data.message));
      break;

    case 'message_read':
      dispatch(handleMessageRead({
        chat_id: data.chat_id,
        message_ids: [data.message_id],
        reader_id: data.read_by,
      }));
      break;

    case 'user_typing':
      dispatch(setUserTyping({
        chatId: data.chat_id,
        userId: data.user_id,
        isTyping: data.is_typing,
      }));
      break;

    case 'user_online':
      dispatch(handleUserOnline({ user_id: data.user_id }));
      break;

    case 'user_offline':
      dispatch(handleUserOffline({
        user_id: data.user_id,
        last_seen: data.last_seen
      }));
      break;

    case 'online_users_list':
      dispatch(setOnlineUsersList({ user_ids: data.online_users }));
      break;
  }
};

ws.onclose = () => {
  dispatch(setConnectionStatus('disconnected'));
  scheduleReconnect();
};

ws.onerror = (error) => {
  dispatch(setConnectionError(error.message));
};
```

#### Heartbeat

```typescript
// Ping каждые 30 секунд
heartbeatInterval = setInterval(() => {
  if (ws?.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({ type: 'ping' }));
  }
}, 30000);
```

---

## Паттерны использования

### 1. Async Data Fetching

```typescript
// Thunk definition
export const fetchCart = createAsyncThunk(
  'cart/fetchCart',
  async (storefrontId: number) => {
    return await cartService.getCart(storefrontId);
  }
);

// Slice
extraReducers: (builder) => {
  builder
    .addCase(fetchCart.pending, (state) => {
      state.loading = true;
      state.error = null;
    })
    .addCase(fetchCart.fulfilled, (state, action) => {
      state.loading = false;
      state.cart = action.payload;
    })
    .addCase(fetchCart.rejected, (state, action) => {
      state.loading = false;
      state.error = action.error.message;
    });
}

// Component
const dispatch = useAppDispatch();
const { cart, loading, error } = useAppSelector(state => state.cart);

useEffect(() => {
  dispatch(fetchCart(storefrontId));
}, [storefrontId]);
```

### 2. Optimistic Updates

```typescript
// Favorites
addToFavorites.pending: (state, action) => {
  const id = action.meta.arg.id;
  state.itemIds.add(id);  // Optimistic
  state.count += 1;
}

addToFavorites.fulfilled: (state) => {
  // Success - do nothing (already updated)
}

addToFavorites.rejected: (state, action) => {
  const id = action.meta.arg.id;
  state.itemIds.delete(id);  // Rollback
  state.count = Math.max(0, state.count - 1);
  state.error = action.error.message;
}
```

### 3. Normalization

```typescript
// Хранение по ID
messages: Record<number, MarketplaceMessage[]> // chatId -> messages
uploadingFiles: Record<string, UploadingFile>  // fileId -> file
deliveryByStorefront: Record<number, SelectedDeliveryMethod> // storefrontId -> method

// Быстрый доступ
itemIds: Set<number>  // favorites
```

### 4. Cache Invalidation

```typescript
// TTL-based
const CACHE_TTL = 5 * 60 * 1000;

function isCacheValid(cached: CachedCalculation): boolean {
  return Date.now() - cached.timestamp < CACHE_TTL;
}

// Locale-based
if (!forceRefresh && state.lastLocale === locale && state.popularCategories.length > 0) {
  return state.popularCategories; // Return cached
}
```

### 5. Pagination

```typescript
// Infinite scroll
loadMessages.fulfilled: (state, action) => {
  const { chatId, messages, page } = action.payload;

  if (page === 1) {
    // First page - replace
    state.messages[chatId] = messages;
  } else {
    // Next pages - prepend old messages
    state.messages[chatId] = [
      ...messages,
      ...(state.messages[chatId] || [])
    ];
  }

  state.hasMoreMessages[chatId] = messages.length === action.payload.limit;
}
```

### 6. Multi-Step Wizard

```typescript
// Checkout flow
const steps: CheckoutStep[] = ['contact', 'address', 'delivery', 'review'];

goToNextStep: (state) => {
  const currentIndex = steps.indexOf(state.currentStep);
  if (currentIndex < steps.length - 1) {
    state.currentStep = steps[currentIndex + 1];
  }
}

// Validation
selectIsCheckoutReady: (state) => {
  return !!(
    state.contactInfo?.firstName &&
    state.selectedAddressId &&
    Object.keys(state.deliveryByStorefront).length > 0
  );
}
```

---

## Синхронизация состояний

### localStorage Sync

**Pattern: Auto-save on changes**

```typescript
// Cart Slice
extraReducers: (builder) => {
  builder
    .addCase(fetchCart.fulfilled, (state, action) => {
      state.cart = action.payload;
      saveCartToStorage(action.payload); // Sync to localStorage
    })
    .addCase(addToCart.fulfilled, (state, action) => {
      state.cart = action.payload;
      saveCartToStorage(action.payload); // Sync to localStorage
    });
}

// Load on init
const useCartSync = () => {
  const dispatch = useAppDispatch();

  useEffect(() => {
    const savedCart = loadCartFromStorage(userId);
    if (savedCart) {
      // Restore from localStorage
    } else {
      // Fetch from server
      dispatch(fetchCart(storefrontId));
    }
  }, [userId]);
};
```

### WebSocket Sync

**Pattern: Server events → Redux actions**

```typescript
// WebSocket event
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);

  if (data.type === 'new_message') {
    // Update Redux immediately
    dispatch(handleNewMessage(data.message));

    // No need to fetch - already have the data
  }
};

// Deduplication
if (state.recentMessageIds.includes(message.id)) {
  return; // Skip duplicate
}
state.recentMessageIds.push(message.id);
```

### Server-Client Sync

**Pattern: Server as source of truth**

```typescript
// Always fetch on mount
useEffect(() => {
  dispatch(fetchCart(storefrontId));
}, [storefrontId]);

// Optimistic update + server confirmation
const addToFavorite = async (id: number) => {
  // 1. Optimistic update
  dispatch(toggleFavoriteOptimistic(id));

  // 2. Send to server
  const result = await dispatch(addToFavorites({ id }));

  // 3. If failed, rollback is automatic (see rejected case)
};
```

### Multi-Tab Sync

**Pattern: BroadcastChannel API**

```typescript
// Not implemented yet - future enhancement
const channel = new BroadcastChannel('vondi_cart');

channel.onmessage = (event) => {
  if (event.data.type === 'cart_updated') {
    dispatch(fetchCart(event.data.storefrontId));
  }
};

// Send updates
channel.postMessage({ type: 'cart_updated', storefrontId });
```

---

## Итоги

### Статистика

- **Redux Slices:** 16 слайсов
- **Async Thunks:** ~80+ thunks
- **Middleware:** 1 custom (WebSocket)
- **API Client:** Unified BFF proxy
- **LocalStorage Sync:** Cart, Preferences
- **WebSocket Events:** 10+ event types

### Ключевые преимущества

1. **Typesafe:** Full TypeScript coverage
2. **Normalized:** Efficient data structure
3. **Cached:** TTL-based caching
4. **Optimistic:** Instant UI updates
5. **Real-time:** WebSocket integration
6. **Resilient:** Retry logic, reconnection
7. **BFF Proxy:** Centralized auth, no CORS

### Связанные документы

- Frontend Architecture: `.passport/frontend/svetu-web.md`
- Backend API: `.passport/services/monolith.md`
- Auth Flow: `.passport/flows/authentication.md`
- WebSocket: `.passport/infrastructure/network.md`

---

**Конец документа**
