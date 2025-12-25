# Паспорт домена: Delivery (Доставка)

> **Обновлено:** 2025-12-21
> **Версия:** 1.0.0
> **Статус:** Production-ready
> **Bounded Context:** Delivery Management

---

## 1. Обзор домена

### 1.1 Описание

**Delivery** — доменная область, отвечающая за управление доставкой товаров от продавца к покупателю. Включает расчёт стоимости, создание отправлений, отслеживание статусов, интеграцию с курьерскими службами и управление адресами доставки.

### 1.2 Границы домена (Bounded Context)

**Входит в домен:**
- Создание и управление отправлениями (Shipment)
- Расчёт стоимости доставки (Rate Calculation)
- Отслеживание статусов (Tracking)
- Интеграция с провайдерами доставки (Post Express, BEX, AKS, D Express)
- Управление адресами доставки (Delivery Address)
- Справочники (населённые пункты, улицы, пункты выдачи)
- Зоны доставки (DeliveryZone)
- Правила расчёта тарифов (PricingRule)

**НЕ входит в домен:**
- Заказы (Orders) — домен Listings/Marketplace
- Оплата доставки (Payment) — домен Payment
- Инвентаризация товаров — домен Warehouse
- Аутентификация пользователей — домен Auth

### 1.3 Ubiquitous Language (Единый язык домена)

| Термин | Определение |
|--------|-------------|
| **Shipment** | Отправление — физическая посылка с товарами, отправленная через провайдера |
| **Provider** | Провайдер доставки (Post Express, BEX, AKS, D Express, City Express) |
| **Tracking Number** | Трек-номер — уникальный идентификатор отправления в системе провайдера |
| **COD** | Cash on Delivery — наложенный платёж (оплата при получении) |
| **Parcel Locker** | Паккетомат — автоматический пункт выдачи посылок |
| **Pickup Point** | Пункт выдачи — точка самовывоза (офис провайдера или склад продавца) |
| **Rate** | Тариф — стоимость доставки |
| **Zone** | Зона доставки — географическая область с общими правилами тарификации |
| **Settlement** | Населённый пункт (город, посёлок) |
| **Proof of Delivery** | Подтверждение доставки (подпись получателя, фото) |
| **Volumetric Weight** | Объёмный вес — расчётный вес на основе габаритов |

---

## 2. Core Entities (Основные сущности)

### 2.1 Shipment (Отправление)

**Агрегат:** Shipment
**Идентификатор:** UUID
**Ответственность:** Управление жизненным циклом отправления

```go
type Shipment struct {
    ID                 int        // Первичный ключ (внутренний)
    UUID               uuid.UUID  // Внешний идентификатор
    UserUUID           *uuid.UUID // Пользователь (FK -> Auth Service)
    OrderUUID          *uuid.UUID // Заказ (FK -> Listings Service)
    ProviderID         int        // FK -> delivery_providers
    OrderID            *int       // Legacy order ID
    ExternalID         *string    // ID провайдера (Post Express)
    TrackingNumber     *string    // Трек-номер
    Status             string     // Статус отправления

    // JSONB поля
    SenderInfo         JSONB      // Отправитель {name, phone, address}
    RecipientInfo      JSONB      // Получатель {name, phone, address}
    PackageInfo        JSONB      // Посылка {weight, dimensions, value}
    CostBreakdown      JSONB      // Детализация стоимости
    ProviderResponse   JSONB      // Ответ API провайдера
    Labels             JSONB      // Этикетки для печати

    // Стоимость
    DeliveryCost       *float64   // Стоимость доставки
    InsuranceCost      *float64   // Страховка
    CODAmount          *float64   // Наложенный платёж

    // Временные метки
    PickupDate         *time.Time // Дата забора
    EstimatedDelivery  *time.Time // Расчётная доставка
    ActualDeliveryDate *time.Time // Фактическая доставка
    CreatedAt          time.Time
    UpdatedAt          time.Time

    // Связи (eager loading)
    Provider *Provider         // Провайдер доставки
    Events   []TrackingEvent   // События трекинга
}
```

**Статусы Shipment:**
- `pending` — создан, ожидает обработки
- `confirmed` — подтверждён провайдером
- `processing` — обрабатывается
- `picked_up` — забран у отправителя
- `in_transit` — в пути
- `out_for_delivery` — на доставке (курьер у получателя)
- `delivered` — доставлен
- `canceled` — отменён
- `failed` — доставка не удалась
- `delivery_attempted` — попытка доставки неудачна
- `returning` — возвращается отправителю
- `returned` — возвращён
- `lost` — утерян
- `damaged` — повреждён

**Бизнес-правила:**
1. Shipment может быть создан только для существующего заказа (OrderUUID)
2. TrackingNumber генерируется провайдером и уникален
3. Status переходит по строгому конечному автомату (FSM)
4. CODAmount не может быть отрицательным
5. При отмене (canceled) возможен возврат только если ещё не `picked_up`
6. Статус `delivered` является финальным и необратимым

**Инварианты:**
- UUID всегда уникален
- ProviderID всегда ссылается на активного провайдера
- DeliveryCost >= 0 (не может быть отрицательной)
- EstimatedDelivery >= PickupDate (логика времени)

---

### 2.2 Provider (Провайдер доставки)

**Entity:** Provider
**Идентификатор:** ID (int)
**Ответственность:** Описание возможностей курьерской службы

```go
type Provider struct {
    ID                   int       // Первичный ключ
    Code                 string    // post_express | bex_express | local
    Name                 string    // Post Express Serbia
    LogoURL              *string   // URL логотипа
    IsActive             bool      // Доступен для использования
    SupportsCOD          bool      // Поддерживает наложенный платёж
    SupportsInsurance    bool      // Поддерживает страховку
    SupportsTracking     bool      // Поддерживает трекинг
    APIConfig            JSONB     // API credentials
    Capabilities         JSONB     // Дополнительные возможности
    CreatedAt            time.Time
    UpdatedAt            time.Time
}
```

**Провайдеры (поддерживаемые):**
- `post_express` — Post Express Serbia (primary, production)
- `bex_express` — BEX Express (planned)
- `aks_express` — AKS Express (planned)
- `d_express` — D Express (planned)
- `city_express` — City Express (planned)
- `local` — Самовывоз / доставка продавцом

**Capabilities (примеры):**
```json
{
  "max_weight_kg": 30,
  "max_dimensions_cm": {"length": 60, "width": 60, "height": 60},
  "service_types": ["same_day", "next_day", "standard", "express"],
  "supports_parcel_lockers": true,
  "supports_pickup_points": true,
  "countries": ["RS", "ME"],
  "cod_fee": 45,
  "insurance_max": 15000
}
```

---

### 2.3 DeliveryAddress (Адрес доставки)

**Entity:** DeliveryAddress
**Идентификатор:** ID (int)
**Ответственность:** Хранение адреса доставки заказа

```go
type DeliveryAddress struct {
    ID              int       // Первичный ключ
    OrderID         int       // FK -> orders (Listings Service)
    UserID          int       // FK -> users (Auth Service)
    SourceAddressID *int      // FK -> user_addresses (Auth Service)

    // Получатель
    RecipientName   string
    RecipientPhone  string
    RecipientEmail  *string

    // Адрес (sr_latin для Post Express)
    Street          string
    City            string
    District        *string
    PostalCode      string    // 5 цифр (11000)
    CountryCode     string    // RS | ME
    Building        *string
    Apartment       *string
    Entrance        *string
    Floor           *string
    Intercom        *string
    Notes           *string   // Комментарий курьеру

    // Геолокация
    Latitude        *float64
    Longitude       *float64

    // Post Express специфичные поля
    PostExpressSettlementID *int  // FK -> post_express_locations
    PostExpressStreetID     *int  // FK -> post_express_streets

    // Выбранный метод доставки
    DeliveryProvider *string  // post_express | bex | local
    DeliveryMethod   *string  // courier | pickup_point | locker
    PickupPointID    *string  // ID пункта выдачи

    CreatedAt       time.Time
}
```

**Бизнес-правила:**
1. Один OrderID → один DeliveryAddress (1:1)
2. PostalCode обязателен для курьерской доставки
3. PickupPointID обязателен для метода `pickup_point` или `locker`
4. PostExpressSettlementID обязателен для Post Express доставки
5. Адрес копируется из Auth Service (SourceAddressID) при создании заказа

**Методы:**
- `GetFullAddress() string` — формирует полный адрес строкой
- `HasGeolocation() bool` — проверяет наличие координат
- `IsPickupPoint() bool` — проверяет тип доставки (pickup/locker)
- `IsCourierDelivery() bool` — проверяет курьерскую доставку

---

### 2.4 TrackingEvent (Событие трекинга)

**Entity:** TrackingEvent
**Идентификатор:** ID (int)
**Ответственность:** Фиксация событий жизненного цикла отправления

```go
type TrackingEvent struct {
    ID           int       // Первичный ключ
    ShipmentID   int       // FK -> delivery_shipments
    ProviderID   int       // FK -> delivery_providers
    EventTime    time.Time // Время события (от провайдера)
    Status       string    // Статус события (confirmed, in_transit, ...)
    Location     *string   // Город/адрес события
    Description  *string   // Описание события
    RawData      JSONB     // Сырые данные от провайдера
    CreatedAt    time.Time // Время записи в БД
}
```

**Источники событий:**
1. Синхронизация с API провайдера (polling)
2. Webhook callbacks (будущая версия)
3. Ручное обновление (админка)

---

### 2.5 DeliveryZone (Зона доставки)

**Entity:** DeliveryZone
**Идентификатор:** ID (int)
**Ответственность:** Определение географических зон для тарификации

```go
type DeliveryZone struct {
    ID           int                     // Первичный ключ
    Name         string                  // Belgrade City Center
    Type         string                  // city | region | country
    Countries    []string                // [RS, ME]
    Regions      []string                // [Vojvodina, Central Serbia]
    Cities       []string                // [Beograd, Novi Sad]
    PostalCodes  []string                // [11000, 21000]
    Boundary     *geography.Polygon      // PostGIS полигон (SRID 4326)
    CenterPoint  *geography.Point        // PostGIS точка
    RadiusKm     *float64                // Радиус зоны (если круговая)
    CreatedAt    time.Time
}
```

**Типы зон:**
- `local` — локальная доставка (в пределах города)
- `regional` — региональная (в пределах региона)
- `national` — национальная (вся страна)
- `international` — международная (planned)

**PostGIS функции:**
- `ST_Contains(Boundary, Point)` — проверка попадания адреса в зону
- `ST_Distance(CenterPoint, Point)` — расчёт расстояния
- `ST_DWithin(Point1, Point2, RadiusKm)` — проверка радиуса

---

### 2.6 PricingRule (Правило расчёта стоимости)

**Entity:** PricingRule
**Идентификатор:** ID (int)
**Ответственность:** Определение правил тарификации

```go
type PricingRule struct {
    ID                        int       // Первичный ключ
    ProviderID                int       // FK -> delivery_providers
    RuleType                  string    // weight | volume | zone | distance
    WeightRanges              JSONB     // [{min: 0, max: 5, price: 300}]
    VolumeRanges              JSONB     // Объёмные диапазоны
    ZoneMultipliers           JSONB     // Множители по зонам
    FragileSurcharge          float64   // Надбавка за хрупкость
    OversizedSurcharge        float64   // Надбавка за негабарит
    SpecialHandlingSurcharge  float64   // Надбавка за спец. обработку
    MinPrice                  *float64  // Минимальная стоимость
    MaxPrice                  *float64  // Максимальная стоимость
    CustomFormula             *string   // SQL-like формула
    Priority                  int       // Приоритет применения (1 = highest)
    IsActive                  bool      // Активно правило
    CreatedAt                 time.Time
    UpdatedAt                 time.Time
}
```

**Rule Types:**
- `weight_based` — расчёт по весу (WeightRanges)
- `volume_based` — расчёт по объёму (VolumeRanges)
- `zone_based` — расчёт по зонам (ZoneMultipliers)
- `combined` — комбинированный (вес + объём + зона)

**Пример WeightRanges:**
```json
[
  {"from": 0, "to": 1, "base_price": 250},
  {"from": 1, "to": 5, "base_price": 350},
  {"from": 5, "to": 10, "base_price": 500}
]
```

---

## 3. Value Objects

### 3.1 CostBreakdown (Детализация стоимости)

```go
type CostBreakdown struct {
    BasePrice           float64 // Базовая стоимость
    WeightSurcharge     float64 // Надбавка за вес
    OversizeSurcharge   float64 // Надбавка за размер
    FragileSurcharge    float64 // Надбавка за хрупкость
    InsuranceFee        float64 // Страховка
    CODFee              float64 // Комиссия за наложенный платёж
    FuelSurcharge       float64 // Топливная надбавка
    RemoteAreaSurcharge float64 // Надбавка за отдалённый район
    Discount            float64 // Скидка (free delivery threshold)
    Tax                 float64 // Налог (VAT)
    Total               float64 // ИТОГО
}
```

**Формула расчёта:**
```
Total = BasePrice + WeightSurcharge + OversizeSurcharge + FragileSurcharge
      + InsuranceFee + CODFee + FuelSurcharge + RemoteAreaSurcharge
      + Tax - Discount
```

**Бизнес-правила:**
- Total >= MinPrice (если задано в PricingRule)
- Total <= MaxPrice (если задано в PricingRule)
- Discount применяется последним (после всех надбавок)

---

### 3.2 Address (Адрес)

```go
type Address struct {
    Name        string // Имя получателя
    Phone       string // Телефон
    Email       string // Email (опционально)
    Street      string // Улица, дом
    City        string // Город
    PostalCode  string // Почтовый индекс
    Country     string // Код страны (RS, ME)
    CompanyName string // Название компании (для B2B)
    Note        string // Примечание
}
```

**Валидация:**
- Phone: формат +381... (Serbian format)
- PostalCode: 5 цифр (11000, 21000)
- Country: ISO 3166-1 alpha-2 (RS, ME)

---

### 3.3 Dimensions (Габариты)

```go
type Dimensions struct {
    LengthCm float64 // Длина (см)
    WidthCm  float64 // Ширина (см)
    HeightCm float64 // Высота (см)
}

// Методы
func (d *Dimensions) CalculateVolume() float64 {
    return (d.LengthCm * d.WidthCm * d.HeightCm) / 1000000 // см³ -> м³
}

func (d *Dimensions) CalculateVolumetricWeight(divisor float64) float64 {
    // Post Express: divisor = 5000
    return (d.LengthCm * d.WidthCm * d.HeightCm) / divisor
}
```

**Ограничения Post Express:**
- Max length: 60 cm
- Max width: 60 cm
- Max height: 60 cm
- Max sum (L+W+H): 180 cm

---

### 3.4 DeliveryAttributes (Атрибуты доставки)

```go
type DeliveryAttributes struct {
    WeightKg                float64     // Вес в кг
    Dimensions              *Dimensions // Габариты
    VolumeM3                float64     // Объём в м³
    IsFragile               bool        // Хрупкий товар
    RequiresSpecialHandling bool        // Требует спец. обработки
    Stackable               bool        // Можно штабелировать
    MaxStackWeightKg        float64     // Макс. вес на стопке
    PackagingType           string      // box | envelope | pallet
    HazmatClass             *string     // Класс опасности (если есть)
}
```

---

## 4. Провайдеры доставки

### 4.1 Provider Interface

```go
type DeliveryProvider interface {
    CreateShipment(ctx context.Context, req *CreateShipmentRequest) (*Shipment, error)
    GetShipment(ctx context.Context, id string) (*Shipment, error)
    TrackShipment(ctx context.Context, trackingNumber string) (*TrackingResponse, error)
    CancelShipment(ctx context.Context, id string, reason string) error
    CalculateRate(ctx context.Context, req *CalculateRateRequest) (*RateResponse, error)
    GetSettlements(ctx context.Context, country string) ([]Settlement, error)
    GetStreets(ctx context.Context, settlementName string) ([]Street, error)
    GetPickupPoints(ctx context.Context, city string) ([]PickupPoint, error)
}
```

### 4.2 Post Express Serbia

**API Base URL:** `https://wsp.posta.rs/api`
**Partner ID:** 10109 (vondi.rs)
**Documentation:** `/p/github.com/vondi-global/.passport/integrations/external/post-express.md`

**Основные методы:**
- **TX 11** — `CalculatePostage` (расчёт стоимости)
- **TX 73** — `CreateManifest` (создание отправления)
- **TX 63** — `GetShipmentStatus` (трекинг)
- **TX 3** — `GetSettlements` (справочник городов)
- **TX 10** — `GetParcelLockers` (список паккетоматов)

**Возможности:**
- ✅ Курьерская доставка (door-to-door)
- ✅ Parcel Lockers (100+ точек)
- ✅ COD (наложенный платёж с авто-переводом)
- ✅ Страхование до 15000 RSD
- ✅ Real-time tracking
- ✅ SMS уведомления

**Тарифы (примеры):**
| Вес (кг) | Базовая цена (RSD) | COD комиссия | Сроки |
|----------|-------------------|--------------|-------|
| 0-1 | 250 | 45 | 1-3 дня |
| 1-5 | 350 | 45 | 1-3 дня |
| 5-10 | 500 | 45 | 1-3 дня |
| 10-20 | 750 | 45 | 2-4 дня |

**Особенности API:**
- Вес в **граммах** (не кг!)
- Стоимость в **para** (1 RSD = 100 para)
- Manifest-based architecture (1 Manifest → N Orders → M Shipments)
- COD настройки: банк счёт, код платежа, модель платежа

### 4.3 BEX Express (Planned)

**Status:** Not implemented yet
**Priority:** v0.2.0

### 4.4 Local Provider (Pickup/Seller Delivery)

**Code:** `local`
**Возможности:**
- Самовывоз из пункта продавца
- Доставка силами продавца
- Бесплатная доставка (опционально)

---

## 5. Use Cases (Варианты использования)

### 5.1 CreateShipment (Создание отправления)

**Актёр:** Order Service
**Цель:** Создать отправление в системе провайдера

**Preconditions:**
- Заказ подтверждён и оплачен
- Адрес доставки валиден
- Выбран провайдер доставки

**Flow:**
1. Получить данные заказа (items, weights, dimensions)
2. Определить провайдера (Post Express, BEX, Local)
3. Рассчитать стоимость доставки (`CalculateRate`)
4. Создать Shipment в БД (status = `pending`)
5. Вызвать API провайдера (POST /shipments)
6. Получить tracking_number и external_id
7. Обновить Shipment (status = `confirmed`, tracking_number)
8. Сгенерировать этикетки (labels)
9. Вернуть ShipmentResponse

**Postconditions:**
- Shipment создан с tracking_number
- Событие `shipment.created` отправлено в event bus
- Уведомление отправлено покупателю

### 5.2 TrackShipment (Отслеживание)

**Актёр:** Customer / Order Service
**Цель:** Получить актуальный статус отправления

**Flow:**
1. Получить tracking_number
2. Найти Shipment в БД
3. Запросить статус у провайдера (API)
4. Синхронизировать события в `delivery_tracking_events`
5. Обновить статус Shipment (если изменился)
6. Вернуть TrackingResponse с events

**Postconditions:**
- События трекинга записаны в БД
- Если статус изменился → событие `shipment.status_changed`

### 5.3 CalculateRate (Расчёт стоимости)

**Актёр:** Checkout Flow
**Цель:** Рассчитать стоимость доставки для checkout

**Flow:**
1. Получить from_address (storefront), to_address (buyer)
2. Рассчитать общий вес и объём товаров
3. Определить зону доставки (PostGIS)
4. Выбрать применимые PricingRules (provider, zone, weight)
5. Рассчитать базовую стоимость (base_price)
6. Применить надбавки (weight, oversize, fragile, COD)
7. Применить скидки (free delivery threshold)
8. Вернуть CostBreakdown

### 5.4 GetAvailableMethods (Методы доставки для checkout)

**Актёр:** Checkout Flow
**Цель:** Получить список доступных методов доставки с ценами

**Flow:**
1. Получить storefront_id и delivery_address
2. Получить настройки storefront (откуда отправка)
3. Рассчитать общий вес/объём items
4. Фильтрация провайдеров (active, supports_zone)
5. Для каждого провайдера:
   - Рассчитать стоимость (`CalculateRate`)
   - Проверить free delivery threshold
   - Добавить в список DeliveryMethod
6. Сортировать по цене/скорости
7. Определить cheapest/fastest
8. Вернуть GetAvailableMethodsResponse

### 5.5 CreateDeliveryAddress (Создание адреса доставки)

**Актёр:** Checkout Flow
**Цель:** Сохранить адрес доставки для заказа

**Flow:**
1. Получить CreateDeliveryAddressRequest
2. Валидация полей (postal_code, phone, city)
3. Если source_address_id указан:
   - Получить адрес из Auth Service
   - Скопировать данные
4. Сохранить в `delivery_addresses`
5. Вернуть созданный DeliveryAddress

---

## 6. gRPC API (13 методов)

### 6.1 CreateShipment
```protobuf
rpc CreateShipment(CreateShipmentRequest) returns (CreateShipmentResponse);
```
Создаёт отправление в системе провайдера.

### 6.2 GetShipment
```protobuf
rpc GetShipment(GetShipmentRequest) returns (GetShipmentResponse);
```
Получает информацию об отправлении по UUID.

### 6.3 TrackShipment
```protobuf
rpc TrackShipment(TrackShipmentRequest) returns (TrackShipmentResponse);
```
Отслеживает статус отправления по tracking_number.

### 6.4 CancelShipment
```protobuf
rpc CancelShipment(CancelShipmentRequest) returns (CancelShipmentResponse);
```
Отменяет отправление (если ещё не `picked_up`).

### 6.5 CalculateRate
```protobuf
rpc CalculateRate(CalculateRateRequest) returns (CalculateRateResponse);
```
Рассчитывает стоимость доставки.

### 6.6 GetSettlements
```protobuf
rpc GetSettlements(GetSettlementsRequest) returns (GetSettlementsResponse);
```
Возвращает список населённых пунктов провайдера.

### 6.7 GetStreets
```protobuf
rpc GetStreets(GetStreetsRequest) returns (GetStreetsResponse);
```
Возвращает список улиц для населённого пункта.

### 6.8 GetParcelLockers
```protobuf
rpc GetParcelLockers(GetParcelLockersRequest) returns (GetParcelLockersResponse);
```
Возвращает список паккетоматов/пунктов выдачи.

### 6.9 GetAvailableMethods
```protobuf
rpc GetAvailableMethods(GetAvailableMethodsRequest) returns (GetAvailableMethodsResponse);
```
Возвращает доступные методы доставки для checkout.

### 6.10 GetPickupPoints
```protobuf
rpc GetPickupPoints(GetPickupPointsRequest) returns (GetPickupPointsResponse);
```
Возвращает пункты выдачи с фильтрацией (город, расстояние).

### 6.11 CreateDeliveryAddress
```protobuf
rpc CreateDeliveryAddress(CreateDeliveryAddressRequest) returns (CreateDeliveryAddressResponse);
```
Создаёт адрес доставки для заказа.

### 6.12 GetDeliveryAddress
```protobuf
rpc GetDeliveryAddress(GetDeliveryAddressRequest) returns (GetDeliveryAddressResponse);
```
Получает адрес доставки по ID.

### 6.13 GetDeliveryAddressByOrder
```protobuf
rpc GetDeliveryAddressByOrder(GetDeliveryAddressByOrderRequest) returns (GetDeliveryAddressByOrderResponse);
```
Получает адрес доставки по order_id.

---

## 7. Domain Events

### 7.1 shipment.created
Отправление создано.
```json
{
  "event_type": "shipment.created",
  "shipment_uuid": "...",
  "user_uuid": "...",
  "order_uuid": "...",
  "tracking_number": "PE123456789",
  "provider": "post_express",
  "timestamp": "2025-12-21T10:00:00Z"
}
```

### 7.2 shipment.status_changed
Статус отправления изменился.
```json
{
  "event_type": "shipment.status_changed",
  "shipment_uuid": "...",
  "old_status": "in_transit",
  "new_status": "delivered",
  "location": "Beograd",
  "timestamp": "2025-12-21T14:00:00Z"
}
```

### 7.3 shipment.delivered
Отправление доставлено (финальный статус).
```json
{
  "event_type": "shipment.delivered",
  "shipment_uuid": "...",
  "user_uuid": "...",
  "delivered_at": "2025-12-21T14:00:00Z",
  "delivered_to": "Petar Petrovic",
  "proof_of_delivery": {
    "signature_url": "..."
  }
}
```

### 7.4 shipment.canceled
Отправление отменено.

### 7.5 delivery_address.created
Адрес доставки создан для заказа.

---

## 8. Интеграции

### 8.1 Auth Service
**Направление:** Delivery → Auth
**Цель:** Получение данных пользователя и адресов

**Методы:**
- `GetUser(user_uuid)` — получить информацию о пользователе
- `GetAddress(address_id)` — получить адрес пользователя
- `ValidateJWT(token)` — валидация JWT токена

### 8.2 Listings Service
**Направление:** Delivery ← Listings
**Цель:** Создание отправлений для заказов

**Методы:**
- `GetOrder(order_uuid)` — получить данные заказа
- `GetOrderItems(order_uuid)` — получить товары заказа (вес, габариты)
- `UpdateOrderShipment(order_uuid, shipment_uuid)` — обновить shipping_id заказа

### 8.3 Notification Service (Planned)
**Направление:** Delivery → Notification
**Цель:** Отправка уведомлений о статусах

**События:**
- `shipment.created` → Email/SMS покупателю
- `shipment.status_changed` → Push-уведомление
- `shipment.delivered` → Email с подтверждением

---

## 9. Микросервис

### 9.1 Архитектура

**Репозиторий:** `/p/github.com/vondi-global/delivery`
**Язык:** Go 1.25.0
**Архитектура:** DDD (Domain-Driven Design)

**Порты:**
- gRPC: 50052 (internal)
- HTTP: 15010 (optional gateway)
- Metrics: 9091 (Prometheus)

**База данных:** delivery_db (PostgreSQL 17 + PostGIS, port 35432)

### 9.2 Структура проекта

```
delivery/
├── cmd/server/main.go
├── internal/
│   ├── domain/          # Entities, Value Objects
│   ├── service/         # Use Cases, Business Logic
│   ├── repository/      # Data Access (PostgreSQL)
│   ├── gateway/         # Provider adapters
│   │   └── provider/
│   │       ├── postexpress_adapter.go
│   │       ├── bex_adapter.go
│   │       └── factory.go
│   ├── grpc/            # gRPC handlers
│   ├── http/            # HTTP handlers (webhooks)
│   └── config/          # Configuration
├── proto/               # Protobuf definitions
├── migrations/          # DB migrations
└── docker-compose.yml
```

### 9.3 Конфигурация (Environment Variables)

```bash
# Service
VONDIDELIVERY_SERVICE_NAME=delivery-service
VONDIDELIVERY_ENV=development
VONDIDELIVERY_LOG_LEVEL=info

# Database
VONDIDELIVERY_DB_HOST=localhost
VONDIDELIVERY_DB_PORT=35432
VONDIDELIVERY_DB_NAME=delivery_db
VONDIDELIVERY_DB_USER=delivery_user
VONDIDELIVERY_DB_PASSWORD=delivery_password

# Server Ports
VONDIDELIVERY_GRPC_PORT=50052
VONDIDELIVERY_HTTP_PORT=15010
VONDIDELIVERY_METRICS_PORT=9091

# Post Express
VONDIDELIVERY_POST_RS_ENABLED=true
VONDIDELIVERY_POST_RS_API_KEY=your_api_key
VONDIDELIVERY_POST_RS_BASE_URL=https://wsp.posta.rs/api

# Auth Service Integration
VONDIDELIVERY_AUTH_SERVICE_GRPC_URL=localhost:20053
```

---

## 10. Database Schema

```sql
-- Провайдеры доставки
CREATE TABLE delivery_providers (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    logo_url VARCHAR(500),
    is_active BOOLEAN DEFAULT TRUE,
    supports_cod BOOLEAN DEFAULT FALSE,
    supports_insurance BOOLEAN DEFAULT FALSE,
    supports_tracking BOOLEAN DEFAULT FALSE,
    api_config JSONB,
    capabilities JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Отправления
CREATE TABLE delivery_shipments (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID UNIQUE DEFAULT gen_random_uuid(),
    user_uuid UUID,
    order_uuid UUID,
    provider_id INTEGER REFERENCES delivery_providers(id),
    external_id VARCHAR(255),
    tracking_number VARCHAR(255) UNIQUE,
    status VARCHAR(50) DEFAULT 'pending',
    sender_info JSONB,
    recipient_info JSONB,
    package_info JSONB,
    delivery_cost DECIMAL(10,2),
    insurance_cost DECIMAL(10,2),
    cod_amount DECIMAL(10,2),
    cost_breakdown JSONB,
    pickup_date TIMESTAMP,
    estimated_delivery TIMESTAMP,
    actual_delivery_date TIMESTAMP,
    provider_response JSONB,
    labels JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- События трекинга
CREATE TABLE delivery_tracking_events (
    id BIGSERIAL PRIMARY KEY,
    shipment_id BIGINT REFERENCES delivery_shipments(id),
    provider_id INTEGER REFERENCES delivery_providers(id),
    event_time TIMESTAMP NOT NULL,
    status VARCHAR(50) NOT NULL,
    location VARCHAR(255),
    description TEXT,
    raw_data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Адреса доставки
CREATE TABLE delivery_addresses (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    source_address_id INTEGER,
    recipient_name VARCHAR(255) NOT NULL,
    recipient_phone VARCHAR(50) NOT NULL,
    recipient_email VARCHAR(255),
    street VARCHAR(500) NOT NULL,
    city VARCHAR(255) NOT NULL,
    district VARCHAR(255),
    postal_code VARCHAR(20) NOT NULL,
    country_code VARCHAR(2) NOT NULL,
    building VARCHAR(50),
    apartment VARCHAR(50),
    entrance VARCHAR(50),
    floor VARCHAR(50),
    intercom VARCHAR(50),
    notes TEXT,
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    post_express_settlement_id INTEGER,
    post_express_street_id INTEGER,
    delivery_provider VARCHAR(50),
    delivery_method VARCHAR(50),
    pickup_point_id VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 11. Roadmap

### Завершено (v0.1.6)
- ✅ Core domain models
- ✅ 13 gRPC методов
- ✅ Post Express integration
- ✅ PostgreSQL + PostGIS
- ✅ Provider Factory Pattern
- ✅ Prometheus metrics

### Планируется (v0.2.0)
- ⏳ BEX Express provider
- ⏳ AKS Express provider
- ⏳ Webhook callbacks
- ⏳ Batch shipment creation
- ⏳ Label printing (PDF)
- ⏳ COD settlements tracking

### Будущее (v1.0.0)
- 🔮 International shipping (ME, HR, BA)
- 🔮 Real-time tracking via WebSockets
- 🔮 Returns management
- 🔮 Mobile app integration

---

**Дата создания:** 2025-12-21
**Автор:** Claude Code
**Версия паспорта:** 1.0.0
