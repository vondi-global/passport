# Database Passport: vondi_db (Monolith Main Database)

> **Статус:** Production-ready | **Версия:** 52+ миграции | **Последнее обновление:** 2025-12-21

---

## 📋 Основная информация

### Подключение

```bash
# Connection String
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db?sslmode=disable"

# Environment Variables
DB_HOST=localhost
DB_PORT=5433
DB_USER=postgres
DB_PASSWORD=mX3g1XGhMRUZEX3l
DB_NAME=vondi_db
DB_SSLMODE=disable
```

### Характеристики

- **СУБД:** PostgreSQL 15+
- **Порт:** 5433 (НЕ стандартный 5432!)
- **Таблиц:** 149 (основная схема public)
- **Расширения:** PostGIS, pg_trgm, fuzzystrmatch, uuid-ossp
- **Миграции:** 52 файла (последняя: 20251220000004)
- **Локация:** Локальная БД монолита

---

## 🗂️ Доменная структура (по модулям)

### 1️⃣ Users & Authentication (14 таблиц)

**Основные таблицы:**
- `users` (Auth Service) - основная таблица пользователей (в отдельной БД auth_dev_db)
- `roles` - роли пользователей (admin, seller, buyer, warehouse_owner)
- `permissions` - разрешения системы
- `role_permissions` - связь ролей и разрешений
- `role_audit_log` - аудит изменения ролей

**Пользовательские настройки:**
- `user_privacy_settings` - настройки приватности
- `user_notification_preferences` - предпочтения уведомлений
- `user_notification_contacts` - контакты для уведомлений
- `user_telegram_connections` - связь с Telegram ботом
- `user_contacts` - контактная информация

**Балансы и подписки:**
- `user_balances` - балансы пользователей (кошелек)
- `balance_transactions` - история транзакций
- `user_subscriptions` - активные подписки
- `subscription_history` - история подписок

**Схема основных связей:**
```
users (Auth Service DB)
  ├─→ roles (1:N)
  ├─→ user_privacy_settings (1:1)
  ├─→ user_notification_preferences (1:1)
  ├─→ user_balances (1:1)
  ├─→ user_telegram_connections (1:1)
  └─→ user_subscriptions (1:N)

roles
  ├─→ role_permissions (N:N через permissions)
  └─→ role_audit_log (аудит изменений)
```

---

### 2️⃣ Marketplace Listings (C2C) (10 таблиц)

**ВАЖНО:** Основная таблица listings находится в микросервисе Listings (listings_dev_db).
В монолите хранятся только вспомогательные данные.

**Таблицы в монолите:**
- `listing_images` - изображения товаров (DEPRECATED - теперь в Listings)
- `listing_locations` - геолокация объявлений
- `listing_stats` - статистика просмотров/действий
- `listing_tags` - теги/метки объявлений
- `listing_views` - история просмотров
- `price_history` - история изменения цен
- `item_performance_metrics` - метрики эффективности
- `map_items_cache` - кэш объектов на карте

**C2C Listings (старая архитектура):**
- `c2c_listings` - объявления C2C (DEPRECATED)
- `c2c_images` - изображения C2C (DEPRECATED)
- `c2c_categories` - категории C2C (DEPRECATED)
- `c2c_listing_variants` - варианты товаров C2C
- `c2c_favorites` - избранные объявления
- `c2c_chats` - чаты по объявлениям
- `c2c_messages` - сообщения в чатах

**Схема:**
```
Listings (Listings Service)
  ├─→ listing_locations (геолокация)
  ├─→ listing_stats (статистика)
  ├─→ listing_tags (теги)
  ├─→ listing_views (просмотры)
  ├─→ price_history (история цен)
  └─→ c2c_favorites (избранное)
```

---

### 3️⃣ Storefronts (B2C) (20+ таблиц)

**Основные сущности:**
- `b2c_stores` - витрины магазинов
- `storefront_staff` - сотрудники витрин
- `storefront_hours` - часы работы
- `storefront_delivery_options` - варианты доставки
- `storefront_payment_methods` - способы оплаты

**Заказы (Storefront Orders):**
- `b2c_orders` - заказы в витринах
- `b2c_order_items` - позиции заказов
- `cart_items` - корзина покупателя
- `shopping_carts` - корзины покупок
- `shopping_cart_items` - товары в корзине

**Товары и варианты:**
- `b2c_products` - продукты витрин
- `b2c_product_variants` - варианты продуктов (размер, цвет)
- `product_variant_attributes` - атрибуты вариантов (цвет, размер)
- `product_variant_attribute_values` - значения атрибутов
- `variant_attribute_mappings` - маппинг атрибутов на варианты
- `category_variant_attributes` - связь категорий с атрибутами вариантов

**Инвентаризация:**
- `b2c_inventory` - остатки товаров
- `inventory_reservations` - резервы товаров (в корзине)
- `inventory_movements` - движение товаров
- `product_locations` - расположение товаров на складе
- `product_batches` - партии товаров

**Склады (WMS интеграция):**
- `warehouses` - склады
- `warehouse_staff` - персонал складов
- `warehouse_zones` - зоны склада
- `storage_locations` - складские локации
- `picking_tasks` - задачи комплектации
- `picking_task_items` - позиции задач комплектации
- `packing_tasks` - задачи упаковки
- `workers` - работники склада
- `wms_inventory_movements` - движения WMS
- `wms_reservations` - резервы WMS

**Схема:**
```
b2c_stores (витрины)
  ├─→ storefront_staff (персонал)
  ├─→ storefront_hours (часы работы)
  ├─→ storefront_delivery_options (доставка)
  ├─→ storefront_payment_methods (оплата)
  ├─→ b2c_products (товары)
  │    ├─→ b2c_product_variants (варианты)
  │    │    └─→ variant_attribute_mappings
  │    └─→ b2c_inventory (остатки)
  ├─→ shopping_carts (корзины)
  │    └─→ shopping_cart_items (товары в корзине)
  └─→ b2c_orders (заказы)
       └─→ b2c_order_items (позиции заказа)

warehouses (склады)
  ├─→ warehouse_staff
  ├─→ warehouse_zones
  ├─→ storage_locations
  ├─→ picking_tasks → picking_task_items
  ├─→ packing_tasks
  └─→ wms_inventory_movements
```

---

### 4️⃣ Payments (10 таблиц)

**Транзакции:**
- `payment_transactions` - все платежные транзакции
- `payment_gateways` - платежные шлюзы (Stripe, Raiffeisen)
- `payment_methods` - способы оплаты
- `marketplace_payment_settings` - настройки оплаты маркетплейса

**Эскроу:**
- `escrow_payments` - эскроу платежи (средства на удержании)
- `merchant_payouts` - выплаты продавцам

**Подписки:**
- `subscription_plans` - планы подписок
- `subscription_payments` - платежи за подписки
- `subscription_usage` - использование подписок

**Переводы:**
- `transfer_requests` - запросы на перевод средств
- `transfer_request_history` - история переводов

**Схема:**
```
payment_transactions
  ├─→ payment_gateways (N:1 - шлюз)
  ├─→ users (N:1 - плательщик)
  ├─→ b2c_orders (1:1 - заказ)
  └─→ escrow_payments (1:1 - эскроу)

marketplace_payment_settings
  └─→ storefronts (1:N)

merchant_payouts
  ├─→ users (продавец)
  └─→ payment_transactions (источник)

transfer_requests
  ├─→ users (отправитель)
  ├─→ users (получатель)
  └─→ transfer_request_history (история)
```

---

### 5️⃣ Delivery & Logistics (24+ таблиц)

**⚠️ ВАЖНО:** `delivery_category_defaults` удалена из монолита.

**Source of Truth для delivery settings:** Delivery Microservice (`delivery_db`)

**Провайдеры доставки:**
- `delivery_providers` - провайдеры (Post Express, BEX, Courier)
- `delivery_pricing_rules` - правила ценообразования
- `delivery_zones` - зоны доставки
- `delivery_notifications` - уведомления о доставке

**Отправления:**
- `delivery_shipments` - все отправления
- `deliveries` - доставки (общая таблица)
- `packages` - посылки
- `delivery_tracking_events` - события отслеживания

**Post Express интеграция:**
- `post_express_settings` - настройки Post Express API
- `post_express_locations` - населенные пункты
- `post_express_offices` - отделения почты
- `post_express_rates` - тарифы доставки
- `post_express_shipments` - отправления Post Express
- `post_express_tracking_events` - события отслеживания

**BEX интеграция:**
- `bex_configuration` - конфигурация BEX
- `bex_shipments` - отправления BEX
- `bex_tracking_events` - события BEX

**Курьеры:**
- `couriers` - курьеры
- `courier_zones` - зоны курьеров
- `courier_location_history` - история перемещений
- `tracking_websocket_connections` - WebSocket подключения для трекинга

**Схема:**
```
delivery_shipments
  ├─→ delivery_providers (провайдер)
  ├─→ b2c_orders (заказ)
  ├─→ delivery_tracking_events (события)
  └─→ post_express_shipments (Post Express)
       ├─→ post_express_locations (локация)
       ├─→ post_express_offices (отделение)
       └─→ post_express_tracking_events (события)

couriers
  ├─→ courier_zones (зоны)
  └─→ courier_location_history (история)
```

---

### 6️⃣ Categories & Attributes

**⚠️ ВАЖНО:** Категории и атрибуты перенесены в Listings Microservice (listings_db).

**Source of Truth:** Listings Microservice (`listings_db`)
- `categories` (UUID, JSONB i18n) - основная таблица категорий
- `attributes` (JSONB i18n) - универсальные атрибуты
- `category_attributes` - связь категорий с атрибутами
- `attribute_values` - значения атрибутов листингов
- `variant_attribute_values` - значения атрибутов вариантов

**Удалённые таблицы (DEPRECATED - очищено 2025-12-21):**
- `unified_attributes` ✗ УДАЛЕНА → используй Listings MS
- `unified_category_attributes` ✗ УДАЛЕНА → используй Listings MS
- `unified_attribute_values` ✗ УДАЛЕНА → используй Listings MS
- `variant_attribute_mappings` ✗ УДАЛЕНА → используй Listings MS
- `ai_category_decisions` ✗ УДАЛЕНА
- `category_detection_cache` ✗ УДАЛЕНА
- `category_detection_feedback` ✗ УДАЛЕНА
- `category_ai_mappings` ✗ УДАЛЕНА
- `category_proposals` ✗ УДАЛЕНА

**Оставшиеся локальные таблицы:**
- `category_keywords` - ключевые слова категорий
- `category_keyword_weights` - веса ключевых слов

**Схема (NEW - микросервисная):**
```
Listings Microservice (listings_db:35434)
  categories (UUID, JSONB)
    ├─→ category_attributes
    │    └─→ attributes (JSONB i18n)
    │         ├─→ attribute_values (listing attrs)
    │         └─→ variant_attribute_values (variant attrs)
    └─→ [категоризация и атрибуты - source of truth]

Monolith (vondi_db:5433)
  └─→ category_keywords (legacy search keywords)
```

**Документация:**
- Listings DB Passport: `.passport/databases/listings_db.md`
- Categories Domain: `.passport/domains/categories.md`
- Listings Service: `.passport/services/listings.md`

---

### 7️⃣ Search & Indexing (15 таблиц)

**Конфигурация поиска:**
- `search_config` - основная конфигурация
- `search_weights` - веса полей поиска
- `search_weights_history` - история изменений весов
- `search_synonyms` - синонимы поиска
- `search_synonyms_config` - конфигурация синонимов

**Поисковые запросы:**
- `search_queries` - история запросов
- `search_statistics` - статистика поиска
- `search_behavior_metrics` - метрики поведения
- `saved_searches` - сохраненные поиски
- `saved_search_notifications` - уведомления о новых результатах

**Оптимизация:**
- `search_optimization_sessions` - сессии оптимизации весов

**Кэш:**
- `query_cache` - кэш запросов
- `map_items_cache` - кэш для карты
- `geocoding_cache` - кэш геокодирования

**Очередь индексации:**
- `indexing_queue` - очередь для индексации в OpenSearch
- `opensearch_indexing_dlq` - Dead Letter Queue для ошибок

**Схема:**
```
search_config
  ├─→ search_weights
  │    └─→ search_weights_history
  └─→ search_synonyms_config
       └─→ search_synonyms

search_queries
  ├─→ search_statistics
  └─→ search_behavior_metrics

saved_searches
  ├─→ saved_search_notifications
  └─→ users (владелец)

indexing_queue
  └─→ opensearch_indexing_dlq (ошибки)
```

---

### 8️⃣ Reviews & Ratings (7 таблиц)

**Основные таблицы:**
- `reviews` - отзывы (универсальные: на listing, storefront, order)
- `review_responses` - ответы продавцов на отзывы
- `review_votes` - голоса за отзывы (helpful/not_helpful)
- `review_confirmations` - подтверждения отзывов
- `review_disputes` - споры по отзывам

**Кэш:**
- `rating_cache` - кэш рейтингов (для быстрого доступа)

**Схема:**
```
reviews
  ├─→ users (автор отзыва)
  ├─→ entity (listing/storefront/order - polymorphic)
  ├─→ review_responses (1:1 - ответ продавца)
  ├─→ review_votes (N:N - голоса пользователей)
  ├─→ review_confirmations (подтверждение покупки)
  └─→ review_disputes (споры)
       └─→ admin (модератор)

rating_cache
  └─→ entity (кэш для быстрого доступа)
```

---

### 9️⃣ Chats & Messages (6 таблиц)

**Основные чаты:**
- `chats` - чаты (универсальные)
- `messages` - сообщения
- `chat_attachments` - вложения в чаты

**C2C чаты (DEPRECATED - удалены 2025-12-21):**
- `c2c_chats` ✗ УДАЛЕНА → используй Listings MS
- `c2c_messages` ✗ УДАЛЕНА → используй Listings MS
- `c2c_favorites` ✗ УДАЛЕНА → используй Listings MS

**Viber интеграция:**
- `viber_users` - пользователи Viber
- `viber_sessions` - сессии Viber
- `viber_messages` - сообщения Viber
- `viber_tracking_sessions` - сессии отслеживания

**Схема:**
```
chats
  ├─→ users (участники - N:N)
  ├─→ messages (N:1)
  │    └─→ chat_attachments (1:N)
  └─→ listing/order (контекст чата)

viber_users
  ├─→ viber_sessions
  ├─→ viber_messages
  └─→ viber_tracking_sessions
```

---

### 🔟 Notifications (6 таблиц)

**Основные таблицы:**
- `notifications` - все уведомления
- `notification_settings` - настройки уведомлений
- `notification_templates` - шаблоны уведомлений
- `user_notification_preferences` - предпочтения пользователей
- `user_notification_contacts` - контакты для уведомлений
- `user_telegram_connections` - связь с Telegram

**Схема:**
```
notifications
  ├─→ users (получатель)
  ├─→ notification_templates (шаблон)
  └─→ entity (контекст: order, listing, etc)

notification_settings
  └─→ notification_templates

user_notification_preferences
  ├─→ users (1:1)
  └─→ user_notification_contacts (1:N)
       └─→ user_telegram_connections (Telegram)
```

---

### 1️⃣1️⃣ Geo & Locations (6 таблиц)

**Административное деление:**
- `unified_geo` - унифицированная таблица геолокаций
- `municipalities` - муниципалитеты
- `cities` - города
- `districts` - районы

**PostGIS:**
- `spatial_ref_sys` - справочник систем координат (PostGIS)

**Кэш:**
- `geocoding_cache` - кэш геокодирования

**Схема:**
```
unified_geo (иерархическая структура)
  ├─→ country (Сербия)
  │    ├─→ region (Воеводина)
  │    │    ├─→ municipality (Нови Сад)
  │    │    │    └─→ city (Нови Сад)
  │    │    │         └─→ district (Центр)
  │    │    └─→ municipality (Суботица)
  │    └─→ region (Центральная Сербия)
  └─→ coordinates (PostGIS POINT)

geocoding_cache
  └─→ address → coordinates (кэш)
```

---

### 1️⃣2️⃣ Translations (10 таблиц)

**Основные таблицы:**
- `translations` - переводы интерфейса
- `translation_tasks` - задачи на перевод
- `translation_providers` - провайдеры переводов (DeepL, Google)
- `translation_quality_metrics` - метрики качества
- `translation_sync_conflicts` - конфликты синхронизации
- `translation_audit_log` - аудит переводов

**Специализированные:**
- `transliteration_rules` - правила транслитерации (SR Cyrillic ↔ Latin)
- `unit_translations` - переводы единиц измерения
- `attribute_option_translations` - переводы опций атрибутов

**Схема:**
```
translations
  ├─→ key (ключ перевода)
  ├─→ locale (en/ru/sr)
  ├─→ value (переведенный текст)
  └─→ translation_providers (провайдер)

translation_tasks
  ├─→ translations (результат)
  ├─→ translation_quality_metrics (качество)
  └─→ translation_audit_log (история)

transliteration_rules
  └─→ SR: Cyrillic ↔ Latin
```

---

### 1️⃣3️⃣ Automotive (VIN & Cars) (9 таблиц)

**Автомобили:**
- `car_makes` - производители (BMW, Mercedes, Toyota)
- `car_models` - модели (X5, E-Class, Camry)
- `car_generations` - поколения моделей

**VIN декодирование:**
- `vin_decode_cache` - кэш декодирования VIN
- `vin_check_history` - история проверок VIN

**История авто:**
- `vin_accident_history` - история ДТП
- `vin_ownership_history` - история владельцев
- `vin_recalls` - отзывные кампании

**Аналитика:**
- `car_market_analysis` - анализ авторынка
- `user_car_view_history` - история просмотров авто

**Схема:**
```
car_makes (производители)
  └─→ car_models (модели)
       └─→ car_generations (поколения)

vin_decode_cache
  ├─→ car_makes
  ├─→ car_models
  ├─→ car_generations
  └─→ vin_check_history

VIN → история
  ├─→ vin_accident_history (ДТП)
  ├─→ vin_ownership_history (владельцы)
  └─→ vin_recalls (отзывы)
```

---

### 1️⃣4️⃣ Franchise (WMS Partner Program) (3 таблицы)

**Заявки на франшизу:**
- `franchise_applications` - заявки на открытие склада-партнера
- `franchise_info_requests` - запросы информации о программе
- `franchise_documents` - документы франшизы

**Схема:**
```
franchise_info_requests
  ├─→ email, phone (контакты)
  └─→ status (pending/contacted/converted)

franchise_applications
  ├─→ user_id (владелец склада)
  ├─→ company_info (юридические данные)
  ├─→ warehouse_details (площадь, оборудование)
  ├─→ status (pending/approved/rejected)
  └─→ franchise_documents (договоры, лицензии)
```

---

### 1️⃣5️⃣ User Behavior & Analytics (6 таблиц)

**Поведение пользователей:**
- `user_behavior_events` - события поведения
- `user_view_history` - история просмотров (универсальная)
- `user_car_view_history` - просмотры авто
- `view_statistics` - статистика просмотров

**Метрики:**
- `item_performance_metrics` - метрики производительности товаров

**Схема:**
```
user_behavior_events
  ├─→ users (кто)
  ├─→ entity (что: listing/product/storefront)
  ├─→ action (view/click/add_to_cart/purchase)
  └─→ timestamp

user_view_history
  ├─→ users
  └─→ listings/products

view_statistics
  └─→ aggregated metrics (просмотры, CTR, конверсии)
```

---

### 1️⃣6️⃣ Import & Data Management (4 таблицы)

**Импорт данных:**
- `import_jobs` - задачи импорта (CSV/Excel → PostgreSQL)
- `import_history` - история импорта
- `import_errors` - ошибки импорта

**Категории:**
- `imported_categories` - импортированные категории (временная таблица)

**Схема:**
```
import_jobs
  ├─→ user_id (кто запустил)
  ├─→ file_url (источник)
  ├─→ status (pending/processing/completed/failed)
  ├─→ import_history (результат)
  └─→ import_errors (ошибки)
```

---

### 1️⃣7️⃣ Testing & QA (3 таблицы)

**Тестирование:**
- `test_runs` - запуски тестов
- `test_results` - результаты тестов
- `test_logs` - логи выполнения

**Схема:**
```
test_runs
  ├─→ test_suite (набор тестов)
  ├─→ status (running/passed/failed)
  └─→ test_results (1:N)
       └─→ test_logs (детальные логи)
```

---

### 1️⃣8️⃣ System & Audit (4 таблицы)

**Миграции:**
- `schema_migrations` - история миграций (golang-migrate)
- `schema_fixtures` - фикстуры данных
- `migration_audit_log` - аудит миграций

**Компоненты:**
- `component_templates` - шаблоны компонентов UI

**Схема:**
```
schema_migrations
  ├─→ version (номер миграции)
  ├─→ dirty (статус)
  └─→ migration_audit_log (история)

schema_fixtures
  └─→ fixture_name (имя фикстуры)
```

---

## 🔗 ERD - Основные связи между доменами

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           USERS & AUTH                                  │
│  users (Auth Service) → roles → permissions                            │
│       ↓                    ↓                                            │
│  user_balances    user_subscriptions                                   │
└─────────────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┴─────────────────────┐
        ↓                                           ↓
┌──────────────────────────┐              ┌──────────────────────────┐
│   MARKETPLACE (C2C)      │              │    STOREFRONTS (B2C)     │
│                          │              │                          │
│  listings (Listings DB)  │              │  b2c_stores              │
│    ↓                     │              │    ↓                     │
│  listing_stats           │              │  b2c_products            │
│  listing_views           │              │    ↓                     │
│  c2c_favorites           │              │  b2c_product_variants    │
│  c2c_chats               │              │    ↓                     │
└──────────────────────────┘              │  b2c_inventory           │
        ↓                                 │    ↓                     │
        └─────────────┐                   │  shopping_carts          │
                      ↓                   │    ↓                     │
              ┌───────────────┐           │  b2c_orders              │
              │   CATEGORIES  │           └──────────────────────────┘
              │               │                     ↓
              │  unified_cat. │           ┌──────────────────────────┐
              │       ↓       │           │      PAYMENTS            │
              │  unified_attr.│           │                          │
              │       ↓       │           │  payment_transactions    │
              │  category_    │           │    ↓                     │
              │  variant_attr │           │  escrow_payments         │
              └───────────────┘           │  merchant_payouts        │
                                          │  transfer_requests       │
                                          └──────────────────────────┘
                                                    ↓
                                          ┌──────────────────────────┐
                                          │     DELIVERY             │
                                          │                          │
                                          │  delivery_shipments      │
                                          │    ↓                     │
                                          │  post_express_shipments  │
                                          │  bex_shipments           │
                                          │  courier_deliveries      │
                                          └──────────────────────────┘
                                                    ↓
                                          ┌──────────────────────────┐
                                          │    WMS (WAREHOUSES)      │
                                          │                          │
                                          │  warehouses              │
                                          │    ↓                     │
                                          │  picking_tasks           │
                                          │  packing_tasks           │
                                          │  wms_inventory           │
                                          └──────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                      SUPPORTING SYSTEMS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  SEARCH:        search_weights, search_queries, indexing_queue         │
│  REVIEWS:       reviews → review_responses → review_disputes           │
│  CHATS:         chats → messages → chat_attachments                    │
│  NOTIFICATIONS: notifications → notification_templates                  │
│  GEO:           unified_geo → municipalities → cities                  │
│  TRANSLATIONS:  translations → translation_providers                    │
│  AUTOMOTIVE:    car_makes → car_models → vin_decode_cache             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Ключевые индексы

### Performance-критичные индексы:

**Users & Auth:**
```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_roles_name ON roles(name);
CREATE INDEX idx_user_balances_user_id ON user_balances(user_id);
```

**Listings:**
```sql
CREATE INDEX idx_listing_stats_listing_id ON listing_stats(listing_id);
CREATE INDEX idx_listing_views_listing_id ON listing_views(listing_id);
CREATE INDEX idx_listing_locations_geom ON listing_locations USING GIST(geom);
```

**Storefronts:**
```sql
CREATE INDEX idx_b2c_products_store_id ON b2c_products(store_id);
CREATE INDEX idx_b2c_inventory_variant_id ON b2c_inventory(variant_id);
CREATE INDEX idx_b2c_orders_store_id ON b2c_orders(store_id);
CREATE INDEX idx_b2c_orders_status ON b2c_orders(status);
```

**Payments:**
```sql
CREATE INDEX idx_payment_transactions_user_id ON payment_transactions(user_id);
CREATE INDEX idx_payment_transactions_status ON payment_transactions(status);
CREATE INDEX idx_payment_transactions_order_reference ON payment_transactions(order_reference);
```

**Delivery:**
```sql
CREATE INDEX idx_delivery_shipments_tracking_number ON delivery_shipments(tracking_number);
CREATE INDEX idx_post_express_shipments_marketplace_order_id ON post_express_shipments(marketplace_order_id);
```

**Search:**
```sql
CREATE INDEX idx_search_queries_normalized_query ON search_queries(normalized_query);
CREATE INDEX idx_search_statistics_query ON search_statistics USING HASH(query);
CREATE INDEX idx_indexing_queue_status ON indexing_queue(status);
```

**Categories:**
```sql
CREATE INDEX idx_unified_category_attributes_category_id ON unified_category_attributes(category_id);
CREATE INDEX idx_unified_attributes_name ON unified_attributes(name);
```

---

## 📊 Миграции - Хронология развития

### 000001-000015: Initial Schema (Legacy Svetu)
- Базовые таблицы (users, listings, orders)
- Расширения PostgreSQL (PostGIS, pg_trgm)
- Legacy категории (до унификации)

### 000016: Fix get_user_subscription Function
- Исправление функции подписок

### 000017: Transfer Requests (2025-11-XX)
- Система переводов средств между пользователями

### 000018: Warehouse Ownership (2025-11-XX)
- Связь складов с владельцами

### 000019: Franchise Applications (2025-12-XX)
- Система заявок на франшизу WMS

### 000020: Franchise Info Requests (2025-12-10)
- Запросы информации о программе франшизы

### 000021: Franchise User ID (2025-12-10)
- Связь заявок франшизы с пользователями

### 20251217000001-3: Category Expansion L2 (2025-12-17)
- Развертывание подкатегорий L2 (Part 4, 5, 6)
- ~17KB каждая миграция

### 20251217000004: Add Variant to Order Items (2025-12-17)
- Добавление поддержки вариантов в позиции заказов

### 20251218000001: Drop Legacy Categories (2025-12-18)
- Удаление старых таблиц c2c_categories
- Миграция на unified_categories

### 20251220000001: Drop Legacy Attributes (2025-12-20)
- Удаление legacy attribute таблиц
- Миграция на unified_attributes

### 20251220000002: Drop Legacy AI Tables (2025-12-20)
- Удаление таблиц AI категоризации:
  - ai_category_decisions
  - category_detection_cache
  - category_detection_feedback
  - category_ai_mappings

### 20251220000003: Drop Legacy Attribute Tables (2025-12-20)
- Удаление старых таблиц атрибутов
- Полный переход на unified систему

### 20251220000004: Marketplace Payment Settings (2025-12-20)
- Настройки оплаты для маркетплейса
- Поддержка нескольких способов оплаты

---

## 🚀 Команды для работы с БД

### Подключение

```bash
# Интерактивная консоль
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db?sslmode=disable"

# Выполнить SQL файл
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db?sslmode=disable" -f script.sql

# Выполнить одну команду
psql "postgres://postgres:mX3g1XGhMRUZEX3l@localhost:5433/vondi_db?sslmode=disable" -c "SELECT COUNT(*) FROM b2c_orders;"
```

### Дампы БД

```bash
# Полный дамп (схема + данные)
PGPASSWORD=mX3g1XGhMRUZEX3l pg_dump \
  -h localhost -p 5433 -U postgres -d vondi_db \
  --no-owner --no-acl \
  --column-inserts --inserts \
  -f vondi_db_full_dump_$(date +%Y%m%d_%H%M%S).sql

# Только схема (без данных)
PGPASSWORD=mX3g1XGhMRUZEX3l pg_dump \
  -h localhost -p 5433 -U postgres -d vondi_db \
  --schema-only --no-owner --no-acl \
  -f vondi_db_schema_only_$(date +%Y%m%d_%H%M%S).sql

# Только данные (без схемы)
PGPASSWORD=mX3g1XGhMRUZEX3l pg_dump \
  -h localhost -p 5433 -U postgres -d vondi_db \
  --data-only --column-inserts --inserts \
  -f vondi_db_data_only_$(date +%Y%m%d_%H%M%S).sql
```

### Миграции

```bash
# Директория с миграциями
cd /p/github.com/vondi-global/vondi/backend

# Применить все миграции (только схема)
./migrator up

# Применить миграции + фикстуры (тестовые данные)
./migrator -with-fixtures up

# Применить только фикстуры (без миграций)
./migrator -only-fixtures up

# Откатить последнюю миграцию
./migrator down

# Проверить версию БД
./migrator version
```

### Полезные SQL запросы

```sql
-- Список всех таблиц с количеством строк
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    (SELECT COUNT(*) FROM ONLY pg_catalog.quote_ident(tablename)) as rows
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Топ 10 самых больших таблиц
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size('public.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size('public.'||tablename) DESC
LIMIT 10;

-- Проверка версии PostgreSQL
SELECT version();

-- Список установленных расширений
SELECT * FROM pg_extension;

-- Активные подключения
SELECT
    datname,
    usename,
    application_name,
    client_addr,
    state,
    query
FROM pg_stat_activity
WHERE datname = 'vondi_db';

-- Проверка индексов таблицы
SELECT
    indexname,
    indexdef
FROM pg_indexes
WHERE tablename = 'b2c_orders';
```

---

## ⚠️ Важные замечания

### 1. Микросервисная архитектура

**КРИТИЧНО:** Монолит vondi_db НЕ хранит все данные!

Отдельные БД микросервисов:
- **Auth Service:** `auth_dev_db` (порт 25432) - пользователи, роли
- **Listings Service:** `listings_dev_db` (порт 35434) - объявления, категории
- **Delivery Service:** `delivery_db` (порт 35432) - доставки, треки

### 2. Legacy таблицы (очищено 2025-12-20)

✗ **УДАЛЁННЫЕ таблицы (НЕ использовать!):**
- `c2c_categories` → используй `unified_categories` (Listings Service)
- `ai_category_decisions` → удалена, AI категоризация отключена
- `category_detection_cache` → удалена
- Legacy attribute таблицы → используй `unified_attributes`

### 3. Naming Conventions

**Таблицы:**
- Marketplace C2C: префикс `c2c_*` (DEPRECATED)
- Storefronts B2C: префикс `b2c_*` или `storefront_*`
- Унифицированные: префикс `unified_*`
- WMS: префикс `wms_*` или `warehouse_*`

**Индексы:**
- Primary Key: `{table}_pkey`
- Foreign Key: `fk_{table}_{column}`
- Regular Index: `idx_{table}_{column}`
- Unique: `uq_{table}_{column}`

### 4. Timezone

**ВСЕ timestamp колонки:** `timestamp with time zone` (UTC)

### 5. JSONB columns

Активно используются для гибких данных:
- `gateway_response` (payment_transactions)
- `working_hours` (post_express_offices)
- `filters` (saved_searches)
- `metadata` (различные таблицы)

---

## 📚 Связанные документы

### Паспорта других БД:
- `/p/github.com/vondi-global/.passport/databases/listings_db.md` - Listings Microservice
- `/p/github.com/vondi-global/.passport/databases/delivery_db.md` - Delivery Microservice
- `/p/github.com/vondi-global/.passport/databases/auth_db.md` - Auth Service (TBD)

### Документация проекта:
- `/p/github.com/vondi-global/SYSTEM_PASSPORT.md` - System Architecture
- `/p/github.com/vondi-global/vondi/backend/README.md` - Backend Documentation
- `/p/github.com/vondi-global/vondi/docs/CLAUDE_DATABASE_GUIDELINES.md` - Database Guidelines

### Миграции:
- `/p/github.com/vondi-global/vondi/backend/migrations/` - SQL миграции
- `/p/github.com/vondi-global/vondi/backend/migrator` - CLI утилита миграций

---

**Последнее обновление:** 2025-12-21
**Версия документа:** 2.0
**Автор:** Claude (Elite Full-Stack Architect)
