# Паспорт: Delivery Service

> Обновлено: 2025-12-21
> Версия: 0.1.6
> Статус: Production-ready

## Идентификация

| Параметр | Значение |
|----------|----------|
| Название | Delivery Service |
| Назначение | Микросервис доставки (shipment, tracking, cost calculation) |
| Язык | Go 1.25.0 |
| Архитектура | DDD (Domain-Driven Design) |
| Репозиторий | github.com/vondi-global/delivery |
| Директория | /p/github.com/vondi-global/delivery |
| Порт HTTP | 15010 (optional gateway) |
| Порт gRPC | 50052 |
| Порт Metrics | 9091 (Prometheus) |
| База данных | delivery_db (PostgreSQL + PostGIS, port 35432) |

## Конфигурация (Environment Variables)

### Service
```bash
VONDIDELIVERY_SERVICE_NAME=delivery-service
VONDIDELIVERY_ENV=development  # development | production
VONDIDELIVERY_LOG_LEVEL=info   # debug | info | warn | error
```

### Database (PostgreSQL + PostGIS)
```bash
VONDIDELIVERY_DB_HOST=localhost
VONDIDELIVERY_DB_PORT=35432
VONDIDELIVERY_DB_NAME=delivery_db
VONDIDELIVERY_DB_USER=delivery_user
VONDIDELIVERY_DB_PASSWORD=delivery_password
VONDIDELIVERY_DB_SSL_MODE=disable
VONDIDELIVERY_DB_MAX_CONNECTIONS=100
VONDIDELIVERY_DB_MAX_IDLE_CONNECTIONS=10
VONDIDELIVERY_DB_CONNECTION_MAX_LIFETIME=1h
VONDIDELIVERY_DB_MIGRATION_PATH=migrations
```

### Server Ports
```bash
VONDIDELIVERY_GRPC_PORT=50052
VONDIDELIVERY_HTTP_PORT=15010
VONDIDELIVERY_METRICS_PORT=9091
VONDIDELIVERY_SERVER_READ_TIMEOUT=30s
VONDIDELIVERY_SERVER_WRITE_TIMEOUT=30s
VONDIDELIVERY_SERVER_IDLE_TIMEOUT=120s
VONDIDELIVERY_SERVER_SHUTDOWN_TIMEOUT=30s
```

### Delivery Providers
```bash
# Default provider (post_express | bex_express | local)
VONDIDELIVERY_GATEWAYS_DEFAULT=post_express

# Post Express Serbia
VONDIDELIVERY_POST_RS_ENABLED=true
VONDIDELIVERY_POST_RS_API_KEY=your_api_key
VONDIDELIVERY_POST_RS_BASE_URL=https://api.postexpress.rs
VONDIDELIVERY_POST_RS_TIMEOUT=30s

# Dex Express (placeholder)
VONDIDELIVERY_DEX_ENABLED=false
VONDIDELIVERY_DEX_API_KEY=your_dex_api_key
VONDIDELIVERY_DEX_BASE_URL=https://api.dex.com
VONDIDELIVERY_DEX_TIMEOUT=30s
```

### OpenTelemetry (Distributed Tracing)
```bash
VONDIDELIVERY_TELEMETRY_ENABLED=false
VONDIDELIVERY_TELEMETRY_SERVICE_NAME=delivery-service
VONDIDELIVERY_TELEMETRY_SERVICE_VERSION=0.1.6
VONDIDELIVERY_TELEMETRY_OTLP_ENDPOINT=localhost:4317  # Jaeger
VONDIDELIVERY_TELEMETRY_SAMPLING_RATE=1.0  # 1.0 = 100%
```

### Auth Service Integration
```bash
VONDIDELIVERY_AUTH_SERVICE_GRPC_URL=localhost:20053
VONDIDELIVERY_AUTH_SERVICE_TIMEOUT=10s
```

### Listings Service Integration (optional)
```bash
VONDIDELIVERY_LISTINGS_SERVICE_ENABLED=false
VONDIDELIVERY_LISTINGS_SERVICE_ADDRESS=localhost:50051
VONDIDELIVERY_LISTINGS_SERVICE_TIMEOUT=30s
```

**⚠️ ВАЖНО:** Категории товаров управляются через **Listings Microservice (listings_db:35434)**, а не монолит. Таблица `delivery_category_defaults` была удалена 2025-12-21.

## Доменная модель

### Shipment (отправление)
```go
type Shipment struct {
    ID                 int        // Первичный ключ
    UUID               uuid.UUID  // Внешний идентификатор
    UserUUID           *uuid.UUID // Пользователь
    OrderUUID          *uuid.UUID // Заказ
    ProviderID         int        // FK -> delivery_providers
    OrderID            *int       // Legacy order ID
    ExternalID         *string    // ID провайдера (Post Express)
    TrackingNumber     *string    // Трекинг-номер
    Status             string     // pending | confirmed | in_transit | delivered | cancelled

    // JSONB поля
    SenderInfo         JSONB      // Отправитель (name, phone, address)
    RecipientInfo      JSONB      // Получатель (name, phone, address)
    PackageInfo        JSONB      // Посылка (weight, dimensions, value)
    CostBreakdown      JSONB      // Детали стоимости (base, insurance, COD)
    ProviderResponse   JSONB      // Ответ API провайдера
    Labels             JSONB      // Этикетки для печати

    // Стоимость
    DeliveryCost       *float64   // Стоимость доставки
    InsuranceCost      *float64   // Страховка
    CODAmount          *float64   // Наложенный платеж (Cash on Delivery)

    // Временные метки
    PickupDate         *time.Time // Дата забора
    EstimatedDelivery  *time.Time // Расчетная доставка
    ActualDeliveryDate *time.Time // Фактическая доставка
    CreatedAt          time.Time
    UpdatedAt          time.Time

    // Связанные данные (join)
    Provider *Provider         // Провайдер доставки
    Events   []TrackingEvent   // События трекинга
}
```

**Статусы Shipment:**
- `pending` - создан, ожидает обработки
- `confirmed` - подтвержден провайдером
- `processing` - обрабатывается
- `picked_up` - забран у отправителя
- `in_transit` - в пути
- `out_for_delivery` - на доставке
- `delivered` - доставлен
- `cancelled` - отменен
- `failed` - доставка не удалась
- `returning` - возвращается отправителю
- `returned` - возвращен

### Provider (провайдер доставки)
```go
type Provider struct {
    ID                   int       // Первичный ключ
    Code                 string    // post_express | bex_express | local
    Name                 string    // Post Express Serbia
    LogoURL              *string   // URL логотипа
    IsActive             bool      // Доступен для использования
    SupportsCOD          bool      // Поддерживает наложенный платеж
    SupportsInsurance    bool      // Поддерживает страховку
    SupportsTracking     bool      // Поддерживает трекинг
    APIConfig            JSONB     // API credentials
    Capabilities         JSONB     // Дополнительные возможности
    CreatedAt            time.Time
    UpdatedAt            time.Time
}
```

**Провайдеры:**
- `post_express` - Post Express Serbia (primary)
- `bex_express` - BEX Express
- `aks_express` - AKS Express
- `d_express` - D Express
- `city_express` - City Express
- `local` - Самовывоз / доставка продавцом

### DeliveryZone (зона доставки)
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

### PricingRule (правило расчета стоимости)
```go
type PricingRule struct {
    ID                        int       // Первичный ключ
    ProviderID                int       // FK -> delivery_providers
    RuleType                  string    // weight | volume | zone | distance
    WeightRanges              JSONB     // [{min: 0, max: 5, price: 300}]
    VolumeRanges              JSONB     // Объемные диапазоны
    ZoneMultipliers           JSONB     // Множители по зонам
    FragileSurcharge          float64   // Надбавка за хрупкость
    OversizedSurcharge        float64   // Надбавка за негабарит
    SpecialHandlingSurcharge  float64   // Надбавка за спец. обработку
    MinPrice                  *float64  // Минимальная стоимость
    MaxPrice                  *float64  // Максимальная стоимость
    CustomFormula             *string   // SQL-like формула
    Priority                  int       // Приоритет применения
    IsActive                  bool      // Активно правило
    CreatedAt                 time.Time
    UpdatedAt                 time.Time
}
```

### CostBreakdown (детализация стоимости)
```go
type CostBreakdown struct {
    BasePrice         float64  // Базовая стоимость
    WeightSurcharge   float64  // Надбавка за вес
    OversizeSurcharge float64  // Надбавка за размер
    FragileSurcharge  float64  // Надбавка за хрупкость
    InsuranceFee      float64  // Страховка
    CODFee            float64  // Комиссия за наложенный платеж
    Discount          float64  // Скидка
    // Total = Base + Weight + Oversize + Fragile + Insurance + COD - Discount
}
```

### TrackingEvent (событие трекинга)
```go
type TrackingEvent struct {
    ID           int       // Первичный ключ
    ShipmentID   int       // FK -> delivery_shipments
    ProviderID   int       // FK -> delivery_providers
    EventTime    time.Time // Время события (от провайдера)
    Status       string    // Статус события
    Location     *string   // Город/адрес события
    Description  *string   // Описание события
    RawData      JSONB     // Сырые данные от провайдера
    CreatedAt    time.Time // Время записи в БД
}
```

### DeliveryAddress (адрес доставки заказа)
```go
type DeliveryAddress struct {
    ID              int       // Первичный ключ
    OrderID         int       // FK -> orders
    UserID          int       // FK -> users
    SourceAddressID *int      // FK -> user_addresses (Auth Service)

    // Получатель
    RecipientName   string
    RecipientPhone  string
    RecipientEmail  *string

    // Адрес
    Street          string
    City            string
    District        *string
    PostalCode      string
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
    DeliveryMethod   *string  // courier | pickup_point
    PickupPointID    *string  // ID пункта выдачи

    CreatedAt       time.Time
}
```

## gRPC API (12 методов)

### 1. CreateShipment - Создание отправления
```protobuf
rpc CreateShipment(CreateShipmentRequest) returns (CreateShipmentResponse);

message CreateShipmentRequest {
  DeliveryProvider provider = 1;      // post_express | bex | local
  Address from_address = 2;           // Отправитель
  Address to_address = 3;             // Получатель
  Package package = 4;                // Параметры посылки (вес, размеры)
  string user_id = 5;                 // UUID пользователя
}

message CreateShipmentResponse {
  Shipment shipment = 1;              // Созданное отправление
}
```

**Бизнес-логика:**
1. Валидация адресов (обязательные поля)
2. Расчет стоимости через `CalculateRate`
3. Вызов API провайдера (Post Express)
4. Сохранение в БД `delivery_shipments`
5. Генерация tracking number
6. Возврат shipment с UUID и external_id

### 2. GetShipment - Получение отправления
```protobuf
rpc GetShipment(GetShipmentRequest) returns (GetShipmentResponse);

message GetShipmentRequest {
  string id = 1;  // UUID отправления
}

message GetShipmentResponse {
  Shipment shipment = 1;
}
```

### 3. TrackShipment - Трекинг отправления
```protobuf
rpc TrackShipment(TrackShipmentRequest) returns (TrackShipmentResponse);

message TrackShipmentRequest {
  string tracking_number = 1;
}

message TrackShipmentResponse {
  Shipment shipment = 1;
  repeated TrackingEvent events = 2;  // История событий
}
```

**Бизнес-логика:**
1. Поиск shipment по tracking_number
2. Запрос актуального статуса от провайдера
3. Синхронизация событий в `delivery_tracking_events`
4. Обновление статуса shipment
5. Возврат shipment + events

### 4. CancelShipment - Отмена отправления
```protobuf
rpc CancelShipment(CancelShipmentRequest) returns (CancelShipmentResponse);

message CancelShipmentRequest {
  string id = 1;      // UUID отправления
  string reason = 2;  // Причина отмены
}

message CancelShipmentResponse {
  Shipment shipment = 1;
}
```

### 5. CalculateRate - Расчет стоимости доставки
```protobuf
rpc CalculateRate(CalculateRateRequest) returns (CalculateRateResponse);

message CalculateRateRequest {
  DeliveryProvider provider = 1;
  Address from_address = 2;
  Address to_address = 3;
  Package package = 4;  // Вес, размеры, стоимость
}

message CalculateRateResponse {
  string cost = 1;                             // Итоговая стоимость
  string currency = 2;                         // RSD
  google.protobuf.Timestamp estimated_delivery = 3;
}
```

**Алгоритм расчета:**
1. Определение зоны доставки (через PostGIS)
2. Выбор pricing rules по весу/объему/зоне
3. Применение надбавок (fragile, oversize, COD)
4. Применение скидок (free delivery threshold)
5. Возврат CostBreakdown

### 6. GetSettlements - Список населенных пунктов
```protobuf
rpc GetSettlements(GetSettlementsRequest) returns (GetSettlementsResponse);

message GetSettlementsRequest {
  DeliveryProvider provider = 1;  // Фильтр по провайдеру
  string country = 2;             // "RS" | "ME"
  string search_query = 3;        // Поиск по имени
}

message GetSettlementsResponse {
  repeated Settlement settlements = 1;
}

message Settlement {
  int32 id = 1;
  string name = 2;      // Beograd
  string zip_code = 3;  // 11000
  string country = 4;   // RS
}
```

**Источник данных:**
- `post_express_locations` для Post Express
- Справочник городов для других провайдеров

### 7. GetStreets - Список улиц населенного пункта
```protobuf
rpc GetStreets(GetStreetsRequest) returns (GetStreetsResponse);

message GetStreetsRequest {
  DeliveryProvider provider = 1;
  string settlement_name = 2;  // ОБЯЗАТЕЛЬНО
  string search_query = 3;     // Поиск по имени улицы
}

message GetStreetsResponse {
  repeated Street streets = 1;
}

message Street {
  int32 id = 1;
  string name = 2;             // Kneza Miloša
  string settlement_name = 3;  // Beograd
}
```

### 8. GetParcelLockers - Список постаматов/пунктов выдачи
```protobuf
rpc GetParcelLockers(GetParcelLockersRequest) returns (GetParcelLockersResponse);

message GetParcelLockersRequest {
  DeliveryProvider provider = 1;
  string city = 2;           // Фильтр по городу
  string search_query = 3;   // Поиск по имени/коду
}

message GetParcelLockersResponse {
  repeated ParcelLocker parcel_lockers = 1;
}

message ParcelLocker {
  int32 id = 1;
  string code = 2;       // BG001
  string name = 3;
  string address = 4;
  string city = 5;
  string zip_code = 6;
  double latitude = 7;
  double longitude = 8;
  bool available = 9;
}
```

**Источник данных:**
- `post_express_offices` для Post Express
- Справочники других провайдеров

### 9. GetAvailableMethods - Доступные методы доставки для checkout
```protobuf
rpc GetAvailableMethods(GetAvailableMethodsRequest) returns (GetAvailableMethodsResponse);

message GetAvailableMethodsRequest {
  int64 storefront_id = 1;           // ОБЯЗАТЕЛЬНО
  int64 address_id = 2;              // Адрес из Auth Service (optional)
  Address delivery_address = 3;      // Или адрес напрямую
  repeated OrderItem items = 4;      // Товары заказа (вес, размеры)
  double order_total = 5;            // Сумма заказа (для бесплатной доставки)
  bool include_inactive = 6;         // Для админки
}

message GetAvailableMethodsResponse {
  repeated DeliveryMethod methods = 1;
  string cheapest_method_id = 2;  // ID самого дешевого
  string fastest_method_id = 3;   // ID самого быстрого
  google.protobuf.Timestamp calculated_at = 4;
}

message DeliveryMethod {
  string provider_id = 1;        // post_express | bex | local
  string method_type = 2;        // courier | pickup_point | standard
  string display_name = 3;       // "Post Express - Доставка курьером"
  double price = 4;              // Стоимость
  string currency = 5;           // RSD
  int32 estimated_days_min = 6;  // 1
  int32 estimated_days_max = 7;  // 3
  bool is_available = 8;         // Доступен ли метод
  string unavailable_reason = 9; // Причина недоступности
  map<string, string> metadata = 10;

  CostBreakdown cost_breakdown = 11;  // Детализация стоимости

  // Возможности провайдера
  bool supports_cod = 12;
  bool supports_insurance = 13;
  bool supports_tracking = 14;
}
```

**Бизнес-логика:**
1. Получение настроек storefront (откуда отправка)
2. Расчет веса/объема всех items
3. Фильтрация провайдеров (active, supports_zone)
4. Расчет стоимости для каждого метода
5. Проверка free delivery threshold
6. Сортировка по цене/скорости
7. Определение cheapest/fastest

### 10. GetPickupPoints - Пункты выдачи с фильтрацией
```protobuf
rpc GetPickupPoints(GetPickupPointsRequest) returns (GetPickupPointsResponse);

message GetPickupPointsRequest {
  string provider_id = 1;       // post_express | bex | local
  string city = 2;              // Beograd
  string postal_code = 3;       // 11000
  Location user_location = 4;   // Для сортировки по расстоянию
  int64 storefront_id = 5;      // Для local provider
  int32 limit = 6;              // Max 20
  double max_distance_km = 7;   // Радиус поиска
  string search_query = 8;      // Поиск по имени/адресу
}

message GetPickupPointsResponse {
  repeated PickupPoint points = 1;
  int32 total_count = 2;
}

message PickupPoint {
  string id = 1;
  string provider_id = 2;
  string type = 3;              // locker | office | storefront
  string name = 4;
  string address = 5;
  string city = 6;
  string postal_code = 7;
  Location coordinates = 8;
  string working_hours = 9;
  string phone = 10;
  bool is_active = 11;
  double distance_km = 12;      // От user_location (если указан)
  map<string, string> metadata = 13;
}
```

**Бизнес-логика:**
1. Фильтрация по provider_id, city, postal_code
2. Расчет расстояния через PostGIS ST_Distance
3. Фильтрация по max_distance_km
4. Поиск по имени/адресу (ILIKE)
5. Сортировка по расстоянию
6. Лимит результатов

### 11. CreateDeliveryAddress - Создание адреса доставки заказа
```protobuf
rpc CreateDeliveryAddress(CreateDeliveryAddressRequest) returns (CreateDeliveryAddressResponse);

message CreateDeliveryAddressRequest {
  int32 order_id = 1;                        // FK -> orders
  int32 user_id = 2;                         // FK -> users
  optional int32 source_address_id = 3;      // Адрес из Auth Service

  string recipient_name = 4;
  string recipient_phone = 5;
  optional string recipient_email = 6;

  string street = 7;
  string city = 8;
  optional string district = 9;
  string postal_code = 10;
  string country_code = 11;  // RS | ME

  optional string building = 12;
  optional string apartment = 13;
  optional string entrance = 14;
  optional string floor = 15;
  optional string intercom = 16;
  optional string notes = 17;

  optional double latitude = 18;
  optional double longitude = 19;

  optional int32 post_express_settlement_id = 20;
  optional int32 post_express_street_id = 21;

  optional string delivery_provider = 22;  // post_express
  optional string delivery_method = 23;    // courier
  optional string pickup_point_id = 24;
}

message CreateDeliveryAddressResponse {
  DeliveryAddress address = 1;
}
```

**Бизнес-логика:**
1. Валидация обязательных полей
2. Если source_address_id - копирование данных из Auth Service
3. Сохранение в `delivery_addresses`
4. Возврат созданного адреса

### 12. GetDeliveryAddress / GetDeliveryAddressByOrder
```protobuf
rpc GetDeliveryAddress(GetDeliveryAddressRequest) returns (GetDeliveryAddressResponse);
rpc GetDeliveryAddressByOrder(GetDeliveryAddressByOrderRequest) returns (GetDeliveryAddressByOrderResponse);

message GetDeliveryAddressRequest {
  int32 id = 1;
}

message GetDeliveryAddressByOrderRequest {
  int32 order_id = 1;
}
```

## Провайдеры доставки

### Provider Factory Pattern
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

func NewProvider(providerCode string, config *Config) (DeliveryProvider, error) {
    switch providerCode {
    case "post_express":
        return postexpress.NewProvider(config)
    case "bex_express":
        return bex.NewProvider(config)
    case "local":
        return local.NewProvider(config)
    default:
        return nil, ErrUnsupportedProvider
    }
}
```

### Post Express Serbia Integration

**API Base URL:** `https://wsp.postexpress.rs/api`

**Основные методы:**
1. **CreateShipment** - `POST /shipments`
   - Создание отправления в системе Post Express
   - Возврат tracking number + barcode

2. **GetTracking** - `GET /shipments/{tracking_number}/tracking`
   - Получение истории событий доставки
   - Статусы: registered, picked_up, in_transit, delivered

3. **GetLocations** - `GET /locations`
   - Справочник городов/населенных пунктов
   - Используется для GetSettlements

4. **GetOffices** - `GET /offices`
   - Список пунктов выдачи Post Express
   - Фильтрация по городу

5. **CalculateRate** - `POST /rates/calculate`
   - Расчет стоимости доставки
   - Входные данные: from/to location, weight, dimensions

**Аутентификация:**
```http
Authorization: Bearer {API_TOKEN}
Content-Type: application/json
```

**Хранение настроек:**
Таблица `post_express_settings`:
```sql
SELECT api_username, api_password, api_endpoint, sender_address
FROM post_express_settings
WHERE enabled = TRUE
LIMIT 1;
```

**Синхронизация справочников:**
- Locations → `post_express_locations` (daily cron)
- Offices → `post_express_offices` (daily cron)
- Rates → `post_express_rates` (weekly cron)

## Расчет стоимости доставки

### Алгоритм CalculateRate

```go
func CalculateDeliveryCost(ctx context.Context, req *CalculateRateRequest) (*CostBreakdown, error) {
    // 1. Определение зоны доставки
    zone := determineDeliveryZone(req.ToAddress)

    // 2. Получение pricing rules
    rules := getPricingRules(req.ProviderID, zone.ID, req.Package.Weight)

    // 3. Базовая стоимость
    basePrice := rules.BasePrice

    // 4. Надбавка за вес
    weightSurcharge := calculateWeightSurcharge(req.Package.Weight, rules.WeightRanges)

    // 5. Надбавка за объем (габариты)
    oversizeSurcharge := 0.0
    volumeM3 := req.Package.Dimensions.CalculateVolume()
    if volumeM3 > rules.MaxVolumeM3 {
        oversizeSurcharge = rules.OversizedSurcharge
    }

    // 6. Надбавка за хрупкость
    fragileSurcharge := 0.0
    if req.Package.IsFragile {
        fragileSurcharge = rules.FragileSurcharge
    }

    // 7. Страховка
    insuranceFee := 0.0
    if req.InsuranceAmount > 0 {
        insuranceFee = calculateInsurance(req.InsuranceAmount)
    }

    // 8. Комиссия COD (наложенный платеж)
    codFee := 0.0
    if req.CODAmount > 0 {
        codFee = rules.CODFeeFixed  // Обычно 45 RSD для Post Express
    }

    // 9. Скидки (free delivery threshold)
    discount := 0.0
    if req.OrderTotal >= FREE_DELIVERY_THRESHOLD {
        discount = basePrice + weightSurcharge
    }

    // 10. Итоговая стоимость
    total := basePrice + weightSurcharge + oversizeSurcharge + fragileSurcharge +
             insuranceFee + codFee - discount

    return &CostBreakdown{
        BasePrice:         basePrice,
        WeightSurcharge:   weightSurcharge,
        OversizeSurcharge: oversizeSurcharge,
        FragileSurcharge:  fragileSurcharge,
        InsuranceFee:      insuranceFee,
        CODFee:            codFee,
        Discount:          discount,
        Total:             total,
    }, nil
}
```

### Таблица тарифов Post Express (пример)

| Вес (кг) | Базовая цена (RSD) | Страховка (до) | COD комиссия | Сроки доставки |
|----------|-------------------|----------------|--------------|----------------|
| 0 - 1    | 250               | 15000 RSD      | 45 RSD       | 1-3 дня        |
| 1 - 5    | 350               | 15000 RSD      | 45 RSD       | 1-3 дня        |
| 5 - 10   | 500               | 15000 RSD      | 45 RSD       | 1-3 дня        |
| 10 - 20  | 750               | 15000 RSD      | 45 RSD       | 2-4 дня        |
| 20 - 30  | 1000              | 15000 RSD      | 45 RSD       | 2-4 дня        |

**Ограничения размеров:**
- Max length: 60 cm
- Max width: 60 cm
- Max height: 60 cm
- Max sum (L+W+H): 180 cm

**Ограничения веса:**
- Max weight: 30 kg (стандарт)
- Специальные отправления: до 50 kg

## Миграции

Всего миграций: **7** (состояние на 2025-12-21)

### 0001_create_shipments_table (initial schema)
```sql
-- Создание таблицы shipments
CREATE TABLE shipments (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID UNIQUE DEFAULT gen_random_uuid(),
    tracking_number VARCHAR(255),
    status VARCHAR(50) DEFAULT 'pending',
    ...
);
```

### 0002_delivery_tables (main tables)
```sql
-- Создание всех основных таблиц доставки:
-- deliveries
-- delivery_providers
-- delivery_zones (с PostGIS geometry)
-- delivery_pricing_rules
-- delivery_category_defaults
-- delivery_notifications
-- delivery_shipments
-- delivery_tracking_events
-- post_express_* (6 таблиц)

-- Индексы:
CREATE INDEX idx_deliveries_status ON deliveries(status);
CREATE INDEX idx_delivery_zones_boundary ON delivery_zones USING gist(boundary);
```

### 0003_add_uuid_fields
```sql
-- Добавление UUID полей для интеграции с другими сервисами
ALTER TABLE deliveries ADD COLUMN uuid UUID DEFAULT gen_random_uuid();
ALTER TABLE deliveries ADD COLUMN user_uuid UUID;
ALTER TABLE deliveries ADD COLUMN order_uuid UUID;
```

### 0004_create_storefronts_tables
```sql
-- Таблицы для интеграции со storefronts
-- (возможно устарели, проверить актуальность)
```

### 0005_create_delivery_addresses
```sql
-- Таблица адресов доставки заказов
CREATE TABLE delivery_addresses (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    source_address_id INTEGER,  -- FK -> user_addresses (Auth Service)

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

CREATE INDEX idx_delivery_addresses_order_id ON delivery_addresses(order_id);
CREATE INDEX idx_delivery_addresses_user_id ON delivery_addresses(user_id);
```

### 0006_drop_b2c_rudiments
```sql
-- Удаление устаревших таблиц B2C
DROP TABLE IF EXISTS b2c_delivery_options;
DROP TABLE IF EXISTS b2c_shipping_methods;
```

### 0007_category_uuid
```sql
-- Добавление UUID в delivery_category_defaults
ALTER TABLE delivery_category_defaults ADD COLUMN category_uuid UUID;
CREATE INDEX idx_delivery_category_defaults_uuid ON delivery_category_defaults(category_uuid);
```

## Запуск и развертывание

### Локальная разработка

```bash
# Клонирование репозитория
cd /p/github.com/vondi-global/delivery

# Копирование .env
cp .env.example .env

# Запуск PostgreSQL + PostGIS
docker compose up -d

# Применение миграций
make migrate-up

# Установка зависимостей
go mod download

# Генерация protobuf
make proto

# Сборка
make build

# Запуск
make run
# или
./bin/delivery-service
```

**Проверка health:**
```bash
# gRPC health check
grpcurl -plaintext localhost:50052 grpc.health.v1.Health/Check

# Metrics
curl http://localhost:9091/metrics
```

### Docker

```bash
# Сборка образа
docker build -t delivery-service:0.1.6 .

# Запуск контейнера
docker run -d \
  --name delivery-service \
  -p 50052:50052 \
  -p 9091:9091 \
  --env-file .env \
  delivery-service:0.1.6
```

### Makefile команды

```bash
make help             # Список всех команд
make build            # Сборка бинарника
make run              # Запуск сервиса
make test             # Запуск тестов
make lint             # Линтер (golangci-lint)
make format           # Форматирование кода
make proto            # Генерация protobuf
make migrate-up       # Применить миграции
make migrate-down     # Откатить миграции
make docker-up        # Запустить PostgreSQL
make docker-down      # Остановить PostgreSQL
make clean            # Очистка бинарников
```

## Интеграция с другими сервисами

### Auth Service (gRPC)
**Цель:** Валидация JWT, получение user_id/email

```go
import authService "github.com/vondi-global/auth/pkg/http/service"

// Middleware для проверки JWT
jwtParserMW := authMiddleware.JWTParser(authServiceInstance)
protected := app.Use(authmiddleware.RequireAuth())
```

### Listings Service (gRPC)
**Цель:** Получение данных о заказах/товарах для расчета доставки

```go
conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())
listingsClient := listingspb.NewListingsServiceClient(conn)

// Получение информации о заказе
order, _ := listingsClient.GetOrder(ctx, &listingspb.GetOrderRequest{
    OrderId: orderID,
})
```

### Monolith (REST API)
**Цель:** Legacy интеграция, получение справочников

```bash
# Получение категорий для delivery_category_defaults
curl http://localhost:3000/api/v1/categories
```

## Мониторинг и метрики

### Prometheus Metrics (порт 9091)

```bash
# Счетчики запросов
delivery_requests_total{method="CreateShipment",status="success"}
delivery_requests_total{method="CalculateRate",status="error"}

# Гистограмма времени выполнения
delivery_request_duration_seconds{method="CreateShipment"}

# Gauges
delivery_active_shipments{status="in_transit"}
delivery_providers_available{provider="post_express"}

# Интеграция с Post Express
post_express_api_calls_total{endpoint="/shipments",status="200"}
post_express_api_errors_total{endpoint="/tracking"}
```

### Логирование

Библиотека: `github.com/vondi-global/lib` (structured logging на базе zerolog)

```go
log.Info().
    Str("shipment_uuid", shipment.UUID.String()).
    Str("tracking_number", shipment.TrackingNumber).
    Str("status", shipment.Status).
    Msg("Shipment created successfully")

log.Error().
    Err(err).
    Str("provider", "post_express").
    Msg("Failed to call Post Express API")
```

**Уровни логов:**
- `debug` - детальная отладка (SQL запросы, API requests)
- `info` - основные события (создание shipment, смена статуса)
- `warn` - предупреждения (timeout, retry)
- `error` - ошибки (failed API calls, DB errors)

### Distributed Tracing (OpenTelemetry)

```bash
# Включение трейсинга
VONDIDELIVERY_TELEMETRY_ENABLED=true
VONDIDELIVERY_TELEMETRY_OTLP_ENDPOINT=localhost:4317

# Jaeger UI
http://localhost:16686
```

**Spans:**
- `delivery.CreateShipment` - создание отправления
- `delivery.CalculateRate` - расчет стоимости
- `post_express.CreateShipment` - вызов Post Express API
- `postgres.InsertShipment` - запись в БД

## Тестирование

### Unit тесты
```bash
go test ./internal/domain -v
go test ./internal/service -v
go test ./internal/repository/postgres -v -db-url="postgres://..."
```

### Integration тесты
```bash
# Требуется запущенная БД
make test-integration
```

### gRPC клиент (example)
```bash
cd examples/grpc_client
go run main.go --method=create-shipment
```

## Roadmap

### Завершено (v0.1.6)
- ✅ Core domain models (Shipment, Provider, DeliveryZone)
- ✅ 12 gRPC методов (CreateShipment, CalculateRate, GetAvailableMethods и т.д.)
- ✅ PostgreSQL + PostGIS (геолокация, зоны)
- ✅ Post Express integration (API client)
- ✅ Provider Factory Pattern
- ✅ DeliveryAddress management
- ✅ Prometheus metrics
- ✅ OpenTelemetry tracing

### Планируется (v0.2.0)
- ⏳ BEX Express provider implementation
- ⏳ AKS Express provider implementation
- ⏳ Webhook для уведомлений о статусах
- ⏳ Batch processing (массовое создание shipments)
- ⏳ Auto-sync справочников Post Express (cron)
- ⏳ Label printing (PDF generation)
- ⏳ COD settlements tracking

### Будущее (v1.0.0)
- 🔮 International shipping (ME, HR, BA)
- 🔮 Real-time tracking via WebSockets
- 🔮 Delivery optimization (route planning)
- 🔮 Returns management
- 🔮 Mobile app integration

## Events (Redis Streams)

> **Статус интеграции:** ✅ ПОЛНОСТЬЮ ИНТЕГРИРОВАНО (2025-12-21)

### Конфигурация Events

```bash
# События включены
VONDIDELIVERY_EVENTS_ENABLED=true
VONDIDELIVERY_EVENTS_CONSUMER_NAME=delivery-instance-1

# Redis для событий
VONDIDELIVERY_REDIS_HOST=localhost
VONDIDELIVERY_REDIS_PORT=36380
VONDIDELIVERY_REDIS_PASSWORD=
VONDIDELIVERY_REDIS_DB=0
```

### Публикуемые события (delivery.events → PVZ Service)

Publisher: `internal/events/publisher.go`

```yaml
delivery.parcel.routed_to_pvz:
  # PVZ Service создаёт Handover с ExpectedAt
  parcel_id, pvz_id, recipient_id, estimated_arrival

delivery.courier.approaching:
  # PVZ Service уведомляет оператора о приближении курьера
  courier_id, pvz_id, parcel_ids, parcel_count, eta_minutes
```

### Подписка (pvz.events ← PVZ Service)

Consumer: `internal/events/consumer.go` (запускается в main.go)

```yaml
pvz.parcel.arrived:
  # Delivery Service обновляет статус посылки
  parcel_id, pvz_id, pvz_code, recipient_id, pickup_code, storage_days, accepted_by

pvz.parcel.picked_up:
  # Delivery Service финализирует доставку
  parcel_id, pvz_id, recipient_id, picked_up_by, pickup_code, storage_fee, days_stored

pvz.parcel.returned:
  # Delivery Service создаёт обратную отправку
  parcel_id, pvz_id, recipient_id, return_reason, returned_by, days_stored

pvz.handover.accepted:
  # Delivery Service подтверждает успешную передачу
  handover_id, pvz_id, parcel_ids, courier_id, accepted_by, parcel_count

pvz.handover.rejected:
  # Delivery Service переназначает посылку
  handover_id, pvz_id, parcel_ids, courier_id, rejected_by, reason

pvz.capacity.warning:
  # Delivery Service снижает приоритет ПВЗ
  pvz_id, pvz_code, current_capacity, max_capacity, usage_percent, level

pvz.capacity.critical:
  # Delivery Service исключает ПВЗ из роутинга
  pvz_id, pvz_code, current_capacity, max_capacity, usage_percent, available_slots
```

### Интеграция в main.go

```go
// Delivery Service (cmd/server/main.go)
if cfg.Events.Enabled {
    redisClient, err = pkgredis.NewClient(cfg.Redis)
    if err == nil {
        eventHandler := events.NewDefaultPVZEventHandler(serviceLogger)
        eventConsumer = events.NewConsumer(redisClient, eventHandler, serviceLogger, cfg.Events.ConsumerName)
        eventConsumer.Start(ctx)
    }
}
```

### Схема взаимодействия

```
Delivery Service                       PVZ Service
      │                                      │
      │──── delivery.events ────────────────►│
      │     (parcel.routed_to_pvz,           │
      │      courier.approaching)            │
      │                                      │
      │◄──── pvz.events ─────────────────────│
      │      (parcel.arrived, picked_up,     │
      │       capacity.warning/critical)     │
```

---

## Зависимости

### Runtime Dependencies
```go
github.com/gofiber/fiber/v2 v2.52.9         // HTTP framework (optional gateway)
github.com/lib/pq v1.10.9                   // PostgreSQL driver
github.com/jmoiron/sqlx v1.4.0              // SQL extensions
github.com/golang-migrate/migrate/v4 v4.19.0 // DB migrations
github.com/google/uuid v1.6.0               // UUID generation
github.com/kelseyhightower/envconfig v1.4.0 // Config management
github.com/rs/zerolog v1.34.0               // Structured logging
github.com/prometheus/client_golang v1.23.2 // Metrics
google.golang.org/grpc v1.76.0              // gRPC framework
google.golang.org/protobuf v1.36.10         // Protobuf
github.com/vondi-global/auth v1.22.1        // Auth client library
```

### Development Dependencies
```go
github.com/golangci/golangci-lint/v2  // Linter
mvdan.cc/gofumpt                      // Code formatter
github.com/bufbuild/buf               // Protobuf tooling
```

## Контакты и поддержка

- **Репозиторий:** https://github.com/vondi-global/delivery
- **Документация:** `/p/github.com/vondi-global/delivery/docs/`
- **CHANGELOG:** `/p/github.com/vondi-global/delivery/CHANGELOG.md`
- **Миграции:** `/p/github.com/vondi-global/delivery/migrations/`

---

**Дата создания паспорта:** 2025-12-21
**Автор:** Claude Code
**Версия паспорта:** 1.0.0
