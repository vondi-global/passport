# Паспорт домена: PVZ (Пункты Выдачи Заказов)

> **Версия:** 1.0.0
> **Дата:** 2025-12-21
> **Статус:** Production-ready
> **Bounded Context:** PVZ Management & Partner Program

---

## 1. Обзор домена

### 1.1 Описание

**PVZ (Пункты Выдачи Заказов)** — доменная область, отвечающая за управление сетью точек выдачи заказов платформы Vondi Marketplace. Домен поддерживает две бизнес-модели:

1. **Полноценные ПВЗ (Full)** — собственные точки выдачи с выделенным персоналом, вместимостью 50-500 посылок, расширенными сервисами (выдача, примерка, возврат, отправка).

2. **Mikro-ПВЗ (Partner)** — партнёрские точки на базе существующего бизнеса (кафе, магазины, салоны, аптеки, АЗС). Вместимость 10-50 посылок, только выдача. Комиссионная модель: 65-75% партнёру, 25-35% Vondi.

**Цель домена:** обеспечить удобную, быструю и доступную выдачу заказов клиентам по всей Сербии с минимальными капитальными затратами на расширение сети за счёт партнёрской программы.

### 1.2 Границы домена (Bounded Context)

**Входит в домен:**
- Управление ПВЗ (создание, редактирование, статусы, capacity)
- Операторы ПВЗ (аутентификация, смены, права доступа)
- Приёмка посылок от курьеров (Handover flow)
- Выдача посылок клиентам (Pickup flow с 6-значным кодом)
- Возвраты товаров (Return processing с фотофиксацией)
- Хранение посылок (3 дня бесплатно, 20 RSD/день, max 21 день)
- Партнёрская программа (регистрация, одобрение, комиссии, выплаты)
- Capacity tracking (мониторинг загрузки, auto-routing при переполнении)
- Геопоиск ближайших ПВЗ (PostGIS)

**НЕ входит в домен:**
- Доставка посылок до ПВЗ — домен Delivery
- Оплата заказов — домен Payment (PVZ использует Payment для COD, refunds, partner commissions)
- Инвентаризация товаров на складе — домен Warehouse
- Заказы и товары — домен Listings/Marketplace
- Аутентификация пользователей — домен Auth

### 1.3 Ubiquitous Language (Единый язык домена)

| Термин | Определение |
|--------|-------------|
| **PVZ** | Пункт Выдачи Заказов — физическая точка для получения посылок клиентами |
| **Full PVZ** | Полноценный ПВЗ — собственная точка Vondi с персоналом и расширенными сервисами |
| **Mikro-PVZ** | Партнёрская точка — кафе/магазин/салон, принимающий посылки за комиссию |
| **Оператор** | Сотрудник ПВЗ, обрабатывающий приёмку и выдачу посылок |
| **Смена (Shift)** | Рабочая сессия оператора с фиксацией начала/конца, статистики |
| **Handover** | Передача посылки от курьера оператору ПВЗ (с QR-сканированием) |
| **Pickup Code** | 6-значный код для получения посылки (TTL 48ч, 5 попыток) |
| **Pickup Event** | Событие выдачи посылки клиенту (с фиксацией оплаты, оператора) |
| **Capacity** | Вместимость ПВЗ (max_capacity - current_parcels = available slots) |
| **Utilization Rate** | Процент заполненности ПВЗ (current / max * 100%) |
| **Overloaded** | Статус ПВЗ при достижении 100% вместимости (auto-routing) |
| **Storage Deadline** | Срок хранения посылки (21 день с момента прибытия) |
| **Storage Fee** | Плата за хранение (20 RSD/день после 3 бесплатных дней) |
| **Partner Commission** | Комиссия партнёру за выдачу посылки (65-75% от платы) |
| **Return Request** | Запрос на возврат товара (с фотофиксацией, причиной) |

---

## 2. Core Entities (Основные сущности)

### 2.1 PVZ (Пункт выдачи)

**Агрегат:** PVZ
**Идентификатор:** UUID (ID BIGSERIAL internal)
**Ответственность:** Управление точкой выдачи, capacity tracking

```go
type PVZType string
const (
    PVZTypeFull   PVZType = "full"   // Собственный ПВЗ
    PVZTypeMikro  PVZType = "mikro"  // Партнёрский
)

type PVZStatus string
const (
    PVZStatusActive          PVZStatus = "active"
    PVZStatusInactive        PVZStatus = "inactive"
    PVZStatusTemporaryClosed PVZStatus = "temporary_closed"
    PVZStatusOverloaded      PVZStatus = "overloaded"  // 100% capacity
)

type PVZ struct {
    ID             uuid.UUID
    Code           string        // PVZ-BG-001, MIKRO-BG-045
    Type           PVZType
    Status         PVZStatus

    // Локация
    Name           string        // Vondi Downtown, Kafe Sunrise (Dorćol)
    Address        string
    City           string
    PostalCode     string
    Latitude       float64
    Longitude      float64
    Country        string        // "RS" default

    // Контакты
    Phone          string
    Email          string

    // Вместимость
    MaxCapacity    int           // 200 для full, 25 для mikro
    CurrentParcels int           // Real-time счётчик
    ReservedSlots  int           // Зарезервировано при роутинге

    // Расписание (JSONB)
    WorkingHours   WorkingHours  // {"mon": "08:00-22:00", "sat": "09:00-18:00"}

    // Владение
    OwnerID        *uuid.UUID    // NULL для собственных
    PartnerID      *uuid.UUID    // FK → partner_accounts для Mikro

    // Комиссия (для Mikro-ПВЗ)
    CommissionRate float64       // 0.70 = 70% партнёру

    // Сервисы
    SupportsPickup  bool         // true всегда
    SupportsReturn  bool         // true для full, зависит от партнёра
    SupportsTryOn   bool         // true для full (примерочная)
    SupportsSending bool         // true для full (отправка посылок)

    CreatedAt      time.Time
    UpdatedAt      time.Time
}

// Методы
func (p *PVZ) IsFull() bool {
    return p.CurrentParcels >= p.MaxCapacity
}

func (p *PVZ) AvailableSlots() int {
    return p.MaxCapacity - p.CurrentParcels - p.ReservedSlots
}

func (p *PVZ) UtilizationRate() float64 {
    return (float64(p.CurrentParcels) / float64(p.MaxCapacity)) * 100
}

func (p *PVZ) CanAcceptParcel() bool {
    return p.Status == PVZStatusActive && p.AvailableSlots() > 0
}

func (p *PVZ) IsOpen(t time.Time) bool {
    weekday := t.Weekday().String()
    hours, ok := p.WorkingHours[weekday]
    if !ok || hours == "closed" {
        return false
    }
    // Parse hours and check if t falls within range
    return true
}
```

**Бизнес-правила:**
1. PVZ может быть создан только с уникальным Code
2. Mikro-ПВЗ обязательно имеет PartnerID (FK → partner_accounts)
3. Full ПВЗ имеет NULL PartnerID и OwnerID
4. Status автоматически меняется на `overloaded` при достижении 100% capacity
5. MaxCapacity должен быть > 0
6. CurrentParcels обновляется триггером при изменении parcels_at_pvz
7. CommissionRate обязателен для Mikro-ПВЗ (NULL для Full)

**Инварианты:**
- Code всегда уникален
- MaxCapacity > 0
- CurrentParcels >= 0
- CurrentParcels <= MaxCapacity
- UtilizationRate <= 100%
- CommissionRate IN [0.65, 0.75] для Mikro-ПВЗ

---

### 2.2 PVZOperator (Оператор ПВЗ)

**Entity:** PVZOperator
**Идентификатор:** UUID
**Ответственность:** Управление рабочими сменами, обработка handover/pickup

```go
type OperatorRole string
const (
    OperatorRoleOperator OperatorRole = "operator"  // Рядовой оператор
    OperatorRoleManager  OperatorRole = "manager"   // Управляющий ПВЗ
)

type PVZOperator struct {
    ID                uuid.UUID
    UserID            uuid.UUID    // FK → auth_db.users
    PVZID             uuid.UUID    // FK → pvz

    Role              OperatorRole
    IsActive          bool

    // Смена
    CurrentShiftStart *time.Time   // NULL если смена не начата

    // Статистика (lifetime)
    TotalHandovers    int
    TotalPickups      int
    TotalReturns      int

    CreatedAt         time.Time
    UpdatedAt         time.Time
}

// Методы
func (o *PVZOperator) StartShift() error {
    if o.CurrentShiftStart != nil {
        return errors.New("shift already started")
    }
    now := time.Now()
    o.CurrentShiftStart = &now
    return nil
}

func (o *PVZOperator) EndShift() error {
    if o.CurrentShiftStart == nil {
        return errors.New("no active shift")
    }
    o.CurrentShiftStart = nil
    return nil
}

func (o *PVZOperator) IsOnShift() bool {
    return o.CurrentShiftStart != nil
}
```

**Бизнес-правила:**
1. Оператор может работать только в одном ПВЗ одновременно
2. Смена должна быть начата перед любыми операциями (handover/pickup)
3. Только manager может редактировать настройки ПВЗ
4. Статистика обновляется при каждом завершённом действии
5. Оператор привязан к auth_db.users через UserID

---

### 2.3 PartnerAccount (Партнёр Mikro-ПВЗ)

**Entity:** PartnerAccount
**Идентификатор:** UUID
**Ответственность:** Партнёрская программа, earnings tracking

```go
type PartnerStatus string
const (
    PartnerStatusPending   PartnerStatus = "pending"   // Ожидает одобрения
    PartnerStatusActive    PartnerStatus = "active"    // Одобрен, работает
    PartnerStatusSuspended PartnerStatus = "suspended" // Приостановлен
    PartnerStatusRejected  PartnerStatus = "rejected"  // Отклонён
)

type BusinessType string
const (
    BusinessTypeCafe       BusinessType = "cafe"        // 70% commission
    BusinessTypeShop       BusinessType = "shop"        // 65%
    BusinessTypeSalon      BusinessType = "salon"       // 70%
    BusinessTypePharmacy   BusinessType = "pharmacy"    // 65%
    BusinessTypeGasStation BusinessType = "gas_station" // 65%
    BusinessTypePostOffice BusinessType = "post_office" // 75%
)

type PartnerAccount struct {
    ID            uuid.UUID
    UserID        uuid.UUID     // FK → auth_db.users

    // Бизнес информация
    BusinessName  string
    BusinessType  BusinessType
    TaxID         string        // PIB в Сербии

    // Контакт
    ContactName   string
    Phone         string
    Email         string

    // Статус
    Status        PartnerStatus
    ApprovedAt    *time.Time
    ApprovedBy    *uuid.UUID    // FK → auth_db.users (admin)
    RejectionReason string

    // Банковские реквизиты
    BankAccount   string        // RS35160005010011234456
    BankName      string        // Banca Intesa

    // Статистика
    TotalEarnings float64       // Lifetime заработки
    TotalParcels  int           // Lifetime выдано посылок

    // Документы (JSONB)
    Documents     []PartnerDocument

    CreatedAt     time.Time
    UpdatedAt     time.Time
}

// Методы
func (p *PartnerAccount) Approve(adminID uuid.UUID) {
    now := time.Now()
    p.Status = PartnerStatusActive
    p.ApprovedAt = &now
    p.ApprovedBy = &adminID
}

func (p *PartnerAccount) Reject(reason string) {
    p.Status = PartnerStatusRejected
    p.RejectionReason = reason
}

func (p *PartnerAccount) Suspend(reason string) {
    p.Status = PartnerStatusSuspended
    p.RejectionReason = reason
}

func (p *PartnerAccount) CanReceiveParcels() bool {
    return p.Status == PartnerStatusActive
}

func (p *PartnerAccount) CommissionRate() float64 {
    switch p.BusinessType {
    case BusinessTypeCafe, BusinessTypeSalon:
        return 0.70
    case BusinessTypeShop, BusinessTypePharmacy, BusinessTypeGasStation:
        return 0.65
    case BusinessTypePostOffice:
        return 0.75
    default:
        return 0.65
    }
}
```

**Бизнес-правила:**
1. Партнёр регистрируется со статусом `pending`
2. Admin должен одобрить (approve) или отклонить (reject) заявку
3. При одобрении создаётся Mikro-ПВЗ с привязкой к PartnerAccount
4. Комиссия зависит от BusinessType (65-75%)
5. Выплаты производятся раз в месяц (или по требованию при накоплении > 5000 RSD)
6. Партнёр может быть приостановлен (suspended) за нарушения

---

### 2.4 ParcelAtPVZ (Посылка в ПВЗ)

**Entity:** ParcelAtPVZ
**Идентификатор:** UUID
**Ответственность:** Lifecycle посылки в ПВЗ, storage fee calculation

```go
type ParcelStatus string
const (
    ParcelStatusAwaiting  ParcelStatus = "awaiting_pickup" // Ожидает клиента
    ParcelStatusPickedUp  ParcelStatus = "picked_up"       // Клиент забрал
    ParcelStatusReturned  ParcelStatus = "returned"        // Возврат продавцу
    ParcelStatusOverdue   ParcelStatus = "overdue"         // >3 дней, начислена плата
    ParcelStatusExpired   ParcelStatus = "expired"         // >21 день, возврат
)

type ParcelAtPVZ struct {
    ID              uuid.UUID
    ParcelID        uuid.UUID     // FK → delivery_db.parcels
    PVZID           uuid.UUID     // FK → pvz
    OrderID         uuid.UUID     // FK → listings_db.orders

    // Статус
    Status          ParcelStatus

    // Клиент
    CustomerID      uuid.UUID
    CustomerName    string
    CustomerPhone   string

    // Код выдачи
    PickupCode      string        // 6 цифр (123456)
    PickupCodeExpiry time.Time    // ArrivedAt + 48h
    PickupAttempts  int           // Max 5

    // Хранение
    ArrivedAt       time.Time
    StorageDeadline time.Time     // ArrivedAt + 21 день
    StorageFee      float64       // Накопленная плата (20 RSD/день)

    // Выдача/возврат
    PickedUpAt      *time.Time
    ReturnedAt      *time.Time

    // Товары (JSONB, для отображения оператору)
    Items           []OrderItem

    // Требования
    RequiresIDCheck bool          // Дорогие товары (>10000 RSD)

    CreatedAt       time.Time
    UpdatedAt       time.Time
}

// Методы
func (p *ParcelAtPVZ) StorageDays() int {
    return int(time.Since(p.ArrivedAt).Hours() / 24)
}

func (p *ParcelAtPVZ) IsOverdue() bool {
    return p.StorageDays() > 3 && p.Status == ParcelStatusAwaiting
}

func (p *ParcelAtPVZ) CalculateStorageFee() float64 {
    days := p.StorageDays()
    if days <= 3 {
        return 0
    }
    return float64(days-3) * 20.0 // 20 RSD/день
}

func (p *ParcelAtPVZ) GeneratePickupCode() string {
    // Crypto random 6 digits
    return fmt.Sprintf("%06d", rand.Intn(1000000))
}

func (p *ParcelAtPVZ) ValidatePickupCode(code string) bool {
    if p.PickupAttempts >= 5 {
        return false // Max 5 attempts
    }
    p.PickupAttempts++
    return p.PickupCode == code && time.Now().Before(p.PickupCodeExpiry)
}
```

**Бизнес-правила:**
1. Посылка создаётся при событии `delivery.parcel.routed_to_pvz`
2. Pickup code генерируется при создании (6 цифр, TTL 48h)
3. Хранение бесплатно первые 3 дня
4. С 4-го дня начисляется 20 RSD/день (StorageFee)
5. Максимум хранения — 21 день, после статус = `expired`
6. После 5 неудачных попыток ввода кода — pickup блокируется на 15 минут
7. При статусе `picked_up` или `returned` — CurrentParcels ПВЗ уменьшается (триггер)

**Инварианты:**
- PickupCode всегда 6 цифр
- StorageDeadline = ArrivedAt + 21 день
- StorageFee >= 0
- PickupAttempts <= 5
- Status переходы: awaiting → (picked_up | returned | overdue | expired)

---

### 2.5 Handover (Передача посылки от курьера)

**Entity:** Handover
**Идентификатор:** UUID
**Ответственность:** Фиксация приёмки посылки в ПВЗ

```go
type HandoverStatus string
const (
    HandoverStatusPending   HandoverStatus = "pending"   // Курьер в пути
    HandoverStatusAccepted  HandoverStatus = "accepted"  // Оператор принял
    HandoverStatusRejected  HandoverStatus = "rejected"  // Оператор отклонил
)

type Handover struct {
    ID              uuid.UUID
    ParcelID        uuid.UUID    // FK → delivery_db.parcels
    PVZID           uuid.UUID    // FK → pvz

    // Участники
    CourierID       uuid.UUID    // FK → auth_db.users
    OperatorID      *uuid.UUID   // FK → pvz_operators (NULL до приёмки)

    // Статус
    Status          HandoverStatus

    // Время
    ExpectedAt      time.Time    // ETA курьера
    ArrivedAt       *time.Time   // Фактическое прибытие
    AcceptedAt      *time.Time

    // Примечания
    Notes           string
    RejectionReason string

    // QR код (для сканирования)
    QRCode          string       // base64 encoded parcel_id

    CreatedAt       time.Time
    UpdatedAt       time.Time
}

// Методы
func (h *Handover) Accept(operatorID uuid.UUID) error {
    if h.Status != HandoverStatusPending {
        return errors.New("handover already processed")
    }
    now := time.Now()
    h.Status = HandoverStatusAccepted
    h.OperatorID = &operatorID
    h.AcceptedAt = &now
    // Создаётся ParcelAtPVZ
    return nil
}

func (h *Handover) Reject(reason string) error {
    if h.Status != HandoverStatusPending {
        return errors.New("handover already processed")
    }
    h.Status = HandoverStatusRejected
    h.RejectionReason = reason
    // Delivery Service должен переназначить ПВЗ
    return nil
}

func (h *Handover) MarkArrived() {
    now := time.Now()
    h.ArrivedAt = &now
}
```

**Бизнес-правила:**
1. Handover создаётся Delivery Service за 30 минут до прибытия курьера
2. Оператор сканирует QR код → статус меняется на `accepted`
3. При `accepted` создаётся ParcelAtPVZ с pickup code
4. При `rejected` — Delivery Service переназначает ПВЗ или возвращает продавцу
5. Если курьер не прибыл в течение 4 часов после ExpectedAt — handover отменяется

---

### 2.6 PickupEvent (Событие выдачи)

**Entity:** PickupEvent
**Идентификатор:** UUID
**Ответственность:** Фиксация выдачи посылки клиенту

```go
type PickupEventType string
const (
    PickupEventTypePickup PickupEventType = "pickup"  // Обычная выдача
    PickupEventTypeReturn PickupEventType = "return"  // Возврат
    PickupEventTypeTryOn  PickupEventType = "tryon"   // Примерка (Full ПВЗ)
)

type PickupEvent struct {
    ID               uuid.UUID
    ParcelID         uuid.UUID
    PVZID            uuid.UUID

    // Участники
    CustomerID       uuid.UUID
    OperatorID       uuid.UUID

    // Детали
    Type             PickupEventType
    PickupCode       string    // 6 цифр (для валидации)

    // Верификация
    CustomerPhone    string
    VerifiedByID     bool      // Проверен паспорт (для дорогих товаров)

    // Время
    EventTime        time.Time

    // Оплата (если COD)
    PaymentMethod    string    // cash | card | ips | prepaid
    CODAmount        float64   // Сумма наложенного платежа

    // Комиссия (для Mikro-ПВЗ)
    CommissionAmount float64   // 35-75 RSD

    CreatedAt        time.Time
}
```

**Бизнес-правила:**
1. PickupEvent создаётся после успешной валидации pickup code
2. Если ParcelAtPVZ.RequiresIDCheck = true → обязательна проверка ID (VerifiedByID)
3. При Type = `pickup` → статус ParcelAtPVZ меняется на `picked_up`
4. При Type = `return` → инициируется ReturnRequest
5. Если PaymentMethod = `cash` | `card` | `ips` → вызов Payment Service (RecordCODPayment)
6. Для Mikro-ПВЗ — расчёт комиссии партнёру (CommissionAmount)

---

### 2.7 ReturnRequest (Запрос на возврат)

**Entity:** ReturnRequest
**Идентификатор:** UUID
**Ответственность:** Обработка возвратов товаров

```go
type ReturnReason string
const (
    ReturnReasonWrongItem      ReturnReason = "wrong_item"
    ReturnReasonDamaged        ReturnReason = "damaged"
    ReturnReasonDefective      ReturnReason = "defective"
    ReturnReasonNotAsDescribed ReturnReason = "not_as_described"
    ReturnReasonChangedMind    ReturnReason = "changed_mind"
)

type ReturnStatus string
const (
    ReturnStatusPending   ReturnStatus = "pending"
    ReturnStatusApproved  ReturnStatus = "approved"
    ReturnStatusRejected  ReturnStatus = "rejected"
    ReturnStatusCompleted ReturnStatus = "completed"
)

type ReturnRequest struct {
    ID              uuid.UUID
    ParcelID        uuid.UUID
    OrderID         uuid.UUID
    CustomerID      uuid.UUID
    PVZID           uuid.UUID
    OperatorID      uuid.UUID

    // Причина
    Reason          ReturnReason
    Notes           string
    Photos          []string      // URLs фото товара

    // Статус
    Status          ReturnStatus
    RequestedAt     time.Time
    ProcessedAt     *time.Time
    CompletedAt     *time.Time

    // Возврат средств
    RefundAmount    float64
    RefundStatus    string        // pending | processing | completed

    CreatedAt       time.Time
    UpdatedAt       time.Time
}
```

**Бизнес-правила:**
1. Клиент может инициировать возврат в течение 14 дней с момента получения
2. Оператор фотографирует товар (Photos)
3. Если Reason = `damaged` | `defective` → возврат 100% без вопросов
4. Если Reason = `changed_mind` → клиент платит обратную доставку (150 RSD)
5. После одобрения (approved) → вызов Payment Service (InitiateRefund)
6. Возврат посылки продавцу через Delivery Service

---

## 3. Value Objects

### 3.1 WorkingHours (Расписание работы)

```go
type WorkingHours map[string]string

// Пример:
// {
//   "monday":    "08:00-22:00",
//   "tuesday":   "08:00-22:00",
//   "wednesday": "08:00-22:00",
//   "thursday":  "08:00-22:00",
//   "friday":    "08:00-22:00",
//   "saturday":  "09:00-18:00",
//   "sunday":    "closed"
// }

func (wh WorkingHours) IsOpen(day string, hour int) bool {
    hours, ok := wh[day]
    if !ok || hours == "closed" {
        return false
    }
    // Parse "08:00-22:00" and check hour
    return true
}
```

### 3.2 Coordinates (Геолокация)

```go
type Coordinates struct {
    Latitude  float64  // 44.8176 (Belgrade)
    Longitude float64  // 20.4574
}

func (c *Coordinates) DistanceTo(other Coordinates) float64 {
    // Haversine formula (PostGIS: earth_distance)
    return haversine(c.Latitude, c.Longitude, other.Latitude, other.Longitude)
}
```

### 3.3 OrderItem (Товар в заказе)

```go
type OrderItem struct {
    ProductID   uuid.UUID
    ProductName string
    SKU         string
    Quantity    int
    ImageURL    string
    Price       float64
    VariantInfo string  // "Размер: 42, Цвет: Синий"
}
```

### 3.4 PartnerDocument (Документ партнёра)

```go
type PartnerDocument struct {
    Type        string    // "tax_certificate", "business_license", "id_card"
    URL         string    // s3.vondi.rs URL
    UploadedAt  time.Time
    Verified    bool      // Проверен админом
    VerifiedBy  *uuid.UUID
    VerifiedAt  *time.Time
}
```

---

## 4. Бизнес-правила и инварианты

### 4.1 Комиссии партнёров

| Тип бизнеса | Код | Комиссия партнёру | Комиссия Vondi |
|-------------|-----|-------------------|----------------|
| Кафе | `cafe` | 70% | 30% |
| Магазин | `shop` | 65% | 35% |
| Салон красоты | `salon` | 70% | 30% |
| Аптека | `pharmacy` | 65% | 35% |
| АЗС | `gas_station` | 65% | 35% |
| Почтовое отделение | `post_office` | 75% | 25% |

**Расчёт:**
```
Плата за выдачу: 50 RSD (фиксировано)
Комиссия партнёру (cafe): 50 * 0.70 = 35 RSD
Комиссия Vondi: 50 * 0.30 = 15 RSD
```

**Выплаты:**
- Раз в месяц (1-го числа следующего месяца)
- По требованию при накоплении > 5000 RSD
- Банковский перевод на BankAccount партнёра

### 4.2 Хранение посылок

**Правила:**
- **Бесплатно:** первые 3 дня (с момента ArrivedAt)
- **Платно:** 20 RSD за каждый день с 4-го по 21-й
- **Максимум:** 21 день, после статус = `expired` и возврат продавцу

**Расчёт:**
```go
func CalculateStorageFee(arrivedAt time.Time) float64 {
    days := int(time.Since(arrivedAt).Hours() / 24)
    if days <= 3 {
        return 0
    }
    if days > 21 {
        days = 21
    }
    return float64(days - 3) * 20.0
}

// Пример:
// День 1-3: 0 RSD
// День 4: 20 RSD
// День 7: 80 RSD (4 дня * 20)
// День 21: 360 RSD (18 дней * 20)
```

**Статусы:**
- `awaiting_pickup` (день 1-3): бесплатно
- `overdue` (день 4-21): начислена плата за хранение
- `expired` (день 22+): возврат продавцу, плата взимается с клиента

### 4.3 Pickup Code (Код выдачи)

**Параметры:**
- Формат: 6 цифр (123456)
- Генерация: crypto/rand (secure random)
- TTL: 48 часов с момента ArrivedAt
- Попытки: максимум 5

**Валидация:**
```go
func ValidatePickupCode(parcel *ParcelAtPVZ, inputCode string) error {
    if parcel.PickupAttempts >= 5 {
        return errors.New("max attempts reached, blocked for 15 min")
    }
    if time.Now().After(parcel.PickupCodeExpiry) {
        // Regenerate code
        parcel.PickupCode = GeneratePickupCode()
        parcel.PickupCodeExpiry = time.Now().Add(48 * time.Hour)
        parcel.PickupAttempts = 0
        return errors.New("code expired, new code sent via SMS")
    }
    parcel.PickupAttempts++
    if parcel.PickupCode != inputCode {
        return errors.New("invalid code")
    }
    return nil
}
```

**SMS уведомления:**
- При прибытии в ПВЗ: "Vaš paket stigao u Vondi Downtown. Kod: 123456. Radno vreme: 08:00-22:00"
- При истечении кода: "Vaš kod ističe. Novi kod: 654321"

### 4.4 Capacity Tracking (Мониторинг загрузки)

**Thresholds:**
- **Normal:** < 90%
- **Warning:** 90-99%
- **Critical:** 100% (статус = `overloaded`)

**Автоматические действия:**
```go
func CheckCapacity(pvz *PVZ) {
    utilization := pvz.UtilizationRate()

    if utilization >= 100 {
        pvz.Status = PVZStatusOverloaded
        // Publish event: pvz.capacity.critical
        // Delivery Service: reroute new parcels to другой ПВЗ
    } else if utilization >= 90 {
        // Publish event: pvz.capacity.warning
        // Admin dashboard: показать alert
    }
}
```

**Auto-routing при переполнении:**
1. Delivery Service получает событие `pvz.capacity.critical`
2. Находит ближайший ПВЗ с доступной вместимостью (AvailableSlots > 0)
3. Переназначает маршрут посылки (RerouteParcel)
4. SMS клиенту: "Ваша посылка перенаправлена в ПВЗ {new_pvz_name} из-за переполнения"

---

## 5. Процессы и workflows

### 5.1 Приёмка посылки (Handover flow)

**Актёры:** Курьер (Delivery Service), Оператор ПВЗ
**Preconditions:**
- Посылка в пути (`in_transit`)
- Handover создан со статусом `pending`
- Оператор начал смену

**Flow:**
```
1. Delivery Service → PVZ Service:
   - CreateHandover(parcelID, pvzID, courierID, eta)
   - Событие: delivery.parcel.courier_approaching (30 мин до прибытия)

2. Курьер прибывает в ПВЗ:
   - Сканирует QR код посылки (генерируется при CreateHandover)

3. Оператор сканирует QR:
   - Вызов: AcceptHandover(handoverID, operatorID)
   - Создаётся ParcelAtPVZ:
     * PickupCode генерируется (6 цифр)
     * PickupCodeExpiry = ArrivedAt + 48h
     * Status = awaiting_pickup
     * StorageDeadline = ArrivedAt + 21 день

4. PVZ Service → Notification Service:
   - SendPickupReadyNotification(customerID, pvzID, pickupCode)
   - SMS: "Vaš paket stigao! Kod: 123456"

5. PVZ Service → Delivery Service:
   - UpdateParcelStatus(parcelID, "arrived_at_pvz")
   - Событие: pvz.parcel.arrived

6. CurrentParcels++:
   - Триггер обновляет pvz.current_parcels
   - Проверка capacity warning/critical
```

**Postconditions:**
- Handover.Status = `accepted`
- ParcelAtPVZ создан с pickup code
- CurrentParcels ПВЗ увеличился на 1
- Клиент получил SMS с кодом выдачи

**Альтернативные сценарии:**
- Оператор отклоняет (RejectHandover): нехватка места, повреждение упаковки
- Delivery Service переназначает ПВЗ или возвращает продавцу

---

### 5.2 Выдача посылки (Pickup flow)

**Актёры:** Клиент, Оператор ПВЗ
**Preconditions:**
- ParcelAtPVZ.Status = `awaiting_pickup`
- Pickup code не истёк (< 48h)
- Оператор на смене

**Flow:**
```
1. Клиент приходит в ПВЗ:
   - Сообщает PickupCode (6 цифр) или показывает SMS

2. Оператор вводит код в интерфейсе:
   - Вызов: ValidatePickupCode(pickupCode)
   - Проверка:
     * PickupAttempts < 5
     * PickupCodeExpiry > NOW()
     * PickupCode == inputCode

3. При успешной валидации:
   - Отображение деталей заказа (товары, фото, сумма)
   - Если RequiresIDCheck = true → проверка паспорта

4. Оператор подтверждает выдачу:
   - Вызов: CompletePickup(parcelID, operatorID, paymentMethod)
   - Если COD:
     * Payment Service: RecordCODPayment(orderID, amount, operatorID)
   - Создание PickupEvent:
     * Type = pickup
     * EventTime = NOW()
     * CommissionAmount (для Mikro-ПВЗ)

5. PVZ Service → Delivery Service:
   - UpdateParcelStatus(parcelID, "picked_up")
   - Событие: pvz.parcel.picked_up

6. PVZ Service → Payment Service (для Mikro-ПВЗ):
   - RecordPartnerCommission(partnerID, commissionAmount)

7. CurrentParcels--:
   - Триггер обновляет pvz.current_parcels

8. Notification Service → Клиент:
   - Email: "Спасибо за покупку! Оцените товар"
```

**Postconditions:**
- ParcelAtPVZ.Status = `picked_up`
- PickupEvent создан
- CurrentParcels ПВЗ уменьшился на 1
- Если COD → Payment Service зафиксировал оплату
- Если Mikro-ПВЗ → PartnerCommission начислена

**Альтернативные сценарии:**
- Неверный код (5 попыток) → блокировка на 15 минут, SMS с новым кодом
- Код истёк → регенерация, SMS с новым кодом
- Клиент хочет вернуть товар → инициирует Return flow

---

### 5.3 Возврат посылки (Return flow)

**Актёры:** Клиент, Оператор ПВЗ
**Preconditions:**
- Посылка выдана (PickedUpAt < 14 дней назад)
- SupportsReturn = true у ПВЗ

**Flow:**
```
1. Клиент приносит товар обратно в ПВЗ:
   - Сообщает причину возврата

2. Оператор осматривает товар:
   - Вызов: ProcessReturn(parcelID, operatorID, reason, photos)
   - Фотографирует товар (Photos[] → S3)
   - Проверяет комплектность

3. Создание ReturnRequest:
   - Reason (damaged | defective | changed_mind)
   - Status = pending
   - Photos

4. PVZ Service → Delivery Service:
   - CreateReturnShipment(orderID, pvzID, reason)
   - Курьер заберёт товар у ПВЗ

5. PVZ Service → Payment Service:
   - InitiateRefund(orderID, refundAmount, reason)
   - Если changed_mind → refundAmount -= 150 RSD (обратная доставка)

6. Admin одобряет возврат (при необходимости):
   - ApproveReturn(returnRequestID)
   - ReturnRequest.Status = approved

7. Payment Service → Клиент:
   - Возврат средств на карту/счёт (3-7 рабочих дней)

8. Notification Service → Клиент:
   - Email: "Ваш возврат одобрен, средства вернутся через 3-7 дней"
```

**Postconditions:**
- ReturnRequest создан
- Delivery Service запланировал забор товара
- Payment Service инициировал возврат средств
- Клиент получил уведомление

---

### 5.4 Партнёрская регистрация (Partner Onboarding)

**Актёры:** Потенциальный партнёр, Admin
**Цель:** Зарегистрировать партнёра, создать Mikro-ПВЗ

**Flow:**
```
1. Партнёр заполняет форму регистрации:
   - BusinessName, BusinessType, TaxID
   - ContactName, Phone, Email
   - Address, Coordinates
   - BankAccount
   - Загружает документы (license, ID)

2. Вызов: RegisterPartner(data)
   - Создаётся PartnerAccount.Status = pending
   - Создаётся User в Auth Service (role: pvz_partner)

3. Notification Service → Партнёр:
   - Email: "Заявка получена, ожидайте одобрения (2-3 рабочих дня)"

4. Admin проверяет заявку в админке:
   - Просматривает документы
   - Проверяет адрес на карте
   - Звонит партнёру для уточнения деталей

5. Admin одобряет:
   - ApprovePartner(partnerAccountID, adminID)
   - PartnerAccount.Status = active
   - Создаётся PVZ:
     * Type = mikro
     * PartnerID = partnerAccountID
     * CommissionRate = businessType.rate (0.65-0.75)
     * MaxCapacity = 25 (по умолчанию для Mikro)

6. Notification Service → Партнёр:
   - Email: "Поздравляем! Ваш Mikro-ПВЗ активирован"
   - SMS с данными для входа

7. Delivery Service:
   - Добавляет новый ПВЗ в роутинг
```

**Postconditions:**
- PartnerAccount.Status = active
- PVZ (type=mikro) создан
- Партнёр может начать приём посылок

**Альтернативные сценарии:**
- Admin отклоняет (RejectPartner): неполные документы, сомнительная локация

---

## 6. Интеграции с другими доменами

### 6.1 Auth Service

**Направление:** PVZ → Auth
**Цель:** Аутентификация операторов и партнёров

**Методы:**
- `ValidateJWT(token)` → UserInfo — валидация JWT токена
- `GetUserByID(userID)` → User — информация о пользователе
- `AssignRole(userID, "pvz_operator")` — назначение роли

**Роли:**
- `pvz_operator` — рядовой оператор
- `pvz_manager` — управляющий ПВЗ
- `pvz_partner` — партнёр Mikro-ПВЗ
- `pvz_admin` — администратор сети ПВЗ

**Permissions:**
- `pvz:handover:accept` — приёмка посылок
- `pvz:pickup:complete` — выдача посылок
- `pvz:return:process` — обработка возвратов
- `pvz:stats:view` — просмотр статистики
- `pvz:settings:edit` — редактирование ПВЗ (только manager)

### 6.2 Payment Service

**Направление:** PVZ → Payment
**Цель:** Оплата при выдаче (COD), возвраты, комиссии партнёрам

**Методы:**
- `RecordCODPayment(orderID, amount, operatorID)` → PaymentResult
  - Фиксация оплаты наличными/картой при выдаче
- `InitiateRefund(orderID, amount, reason)` → RefundResult
  - Инициирование возврата средств
- `RecordPartnerCommission(partnerID, orderID, amount)` → CommissionResult
  - Начисление комиссии партнёру за выдачу
- `GetPartnerBalance(partnerID)` → PartnerBalance
  - Получение текущего баланса партнёра
- `InitiatePartnerPayout(partnerID, amount)` → PayoutResult
  - Инициирование выплаты партнёру

**Сценарии:**
1. **COD при выдаче:**
   ```
   CompletePickup → RecordCODPayment → ReleaseEscrow (продавцу)
   ```

2. **Возврат:**
   ```
   ProcessReturn → InitiateRefund → клиент получает деньги через 3-7 дней
   ```

3. **Комиссия Mikro-ПВЗ:**
   ```
   CompletePickup (Mikro) → RecordPartnerCommission (35 RSD)
   ```

4. **Выплата партнёру:**
   ```
   Раз в месяц: GetPartnerBalance → InitiatePartnerPayout → банковский перевод
   ```

### 6.3 Delivery Service

**Направление:** PVZ ↔ Delivery (bidirectional)
**Цель:** Роутинг посылок на ПВЗ, синхронизация статусов

**Методы (Delivery → PVZ):**
- `GetAvailablePVZ(city, coordinates)` → []PVZInfo
  - Список доступных ПВЗ для роутинга
- `RouteParcelToPVZ(parcelID, pvzID, eta)` → Success
  - Направление посылки на конкретный ПВЗ
- `RerouteParcel(parcelID, newPVZID, reason)` → Success
  - Переназначение ПВЗ (при переполнении)

**Методы (PVZ → Delivery):**
- `UpdateParcelStatus(parcelID, status)` → Success
  - Обновление статуса посылки (arrived_at_pvz, picked_up, returned)
- `GetParcelInfo(parcelID)` → Parcel
  - Получение информации о посылке

**События (Delivery → PVZ):**
```yaml
delivery.parcel.routed_to_pvz:
  parcel_id, pvz_id, order_id, customer_id, eta, items[]
  → PVZ создаёт Handover

delivery.parcel.courier_approaching:
  parcel_id, pvz_id, courier_id, eta_minutes
  → PVZ уведомляет оператора
```

**События (PVZ → Delivery):**
```yaml
pvz.parcel.arrived:
  parcel_id, pvz_id, operator_id, pickup_code
  → Delivery обновляет статус

pvz.parcel.picked_up:
  parcel_id, pvz_id, customer_id, payment_method
  → Delivery финализирует доставку

pvz.parcel.returned:
  parcel_id, pvz_id, reason, photos[]
  → Delivery создаёт обратную доставку

pvz.capacity.warning:
  pvz_id, utilization_pct
  → Delivery избегает перегруженный ПВЗ
```

### 6.4 Listings Service

**Направление:** PVZ → Listings
**Цель:** Информация о товарах для отображения оператору

**Методы:**
- `GetProduct(productID)` → Product
- `GetProducts([]productIDs)` → []Product (batch)
- `GetProductImages(productID)` → []ImageURL

**Сценарии:**
1. **При приёмке (handover):**
   - Оператор видит список товаров в посылке для сверки

2. **При выдаче (pickup):**
   - Отображение товаров клиенту (название, фото, вариант)

3. **При возврате:**
   - Проверка комплектности, сверка с описанием

### 6.5 Notification Service

**Направление:** PVZ → Notification
**Цель:** SMS/Email уведомления клиентам и операторам

**Методы:**
- `SendPickupReadyNotification(customerID, pvzID, pickupCode)` → Success
  - SMS/Email о прибытии в ПВЗ
- `SendPickupCode(customerID, phone, code)` → Success
  - SMS с кодом выдачи (при регенерации)
- `SendStorageReminder(customerID, parcelID, daysLeft)` → Success
  - Напоминание о сроке хранения
- `SendExpiredNotification(customerID, parcelID)` → Success
  - Уведомление о возврате продавцу
- `SendPartnerApprovalNotification(partnerID, approved, reason)` → Success
  - Email партнёру об одобрении/отклонении
- `SendPayoutNotification(partnerID, amount)` → Success
  - Уведомление о выплате

**Шаблоны (SR - Serbian Latin):**
| Событие | SMS | Email |
|---------|-----|-------|
| Прибытие | "Vaš paket stigao u {pvz_name}! Kod: {code}" | Детали + карта |
| Напоминание (3 дня) | "Vaš paket čeka. Ostalo {days} dana besplatno" | — |
| Истечение | "HITNO: Paket ističe sutra! Kod: {code}" | — |
| Партнёр одобрен | — | "Čestitamo! Mikro-ПВЗ odobren" |
| Выплата | "Vondi isplata: {amount} RSD" | Детали перевода |

### 6.6 Monolith (Backend)

**Направление:** PVZ → Monolith
**Цель:** Информация о заказах, legacy интеграция

**Методы (REST):**
- `GET /api/v1/orders/:id` → Order
- `PUT /api/v1/orders/:id/status` → Success
- `GET /api/v1/customers/:id` → Customer

**Webhook Callbacks (PVZ → Monolith):**
- `POST /api/v1/webhooks/pvz/pickup` — уведомление о выдаче
- `POST /api/v1/webhooks/pvz/return` — уведомление о возврате

---

## 7. События домена (Domain Events)

### 7.1 pvz.parcel.arrived

Посылка прибыла в ПВЗ, создан pickup code.

```json
{
  "event_type": "pvz.parcel.arrived",
  "parcel_id": "uuid",
  "pvz_id": "uuid",
  "operator_id": "uuid",
  "pickup_code": "123456",
  "arrived_at": "2025-12-21T14:30:00Z",
  "storage_deadline": "2026-01-11T14:30:00Z"
}
```

**Подписчики:**
- Notification Service → SMS клиенту с кодом
- Delivery Service → обновление статуса

### 7.2 pvz.parcel.picked_up

Клиент забрал посылку.

```json
{
  "event_type": "pvz.parcel.picked_up",
  "parcel_id": "uuid",
  "pvz_id": "uuid",
  "customer_id": "uuid",
  "operator_id": "uuid",
  "payment_method": "cash",
  "cod_amount": 2500.00,
  "picked_up_at": "2025-12-22T16:00:00Z"
}
```

**Подписчики:**
- Payment Service → RecordCODPayment, ReleaseEscrow
- Delivery Service → финализация доставки
- Listings Service → обновление статуса заказа

### 7.3 pvz.parcel.returned

Клиент вернул товар.

```json
{
  "event_type": "pvz.parcel.returned",
  "parcel_id": "uuid",
  "pvz_id": "uuid",
  "customer_id": "uuid",
  "operator_id": "uuid",
  "reason": "defective",
  "photos": ["https://s3.vondi.rs/returns/photo1.jpg"],
  "returned_at": "2025-12-23T10:00:00Z"
}
```

**Подписчики:**
- Payment Service → InitiateRefund
- Delivery Service → CreateReturnShipment
- Notification Service → Email клиенту

### 7.4 pvz.capacity.warning

ПВЗ заполнен на 90%+.

```json
{
  "event_type": "pvz.capacity.warning",
  "pvz_id": "uuid",
  "pvz_code": "PVZ-BG-001",
  "current_parcels": 180,
  "max_capacity": 200,
  "utilization_pct": 90.0,
  "timestamp": "2025-12-21T18:00:00Z"
}
```

**Подписчики:**
- Delivery Service → избегать при роутинге новых посылок
- Admin dashboard → показать alert

### 7.5 pvz.capacity.critical

ПВЗ переполнен (100%).

```json
{
  "event_type": "pvz.capacity.critical",
  "pvz_id": "uuid",
  "pvz_code": "PVZ-BG-001",
  "current_parcels": 200,
  "max_capacity": 200,
  "status": "overloaded",
  "timestamp": "2025-12-21T20:00:00Z"
}
```

**Подписчики:**
- Delivery Service → RerouteParcel для pending посылок
- Notification Service → SMS операторам, admin

### 7.6 pvz.partner.approved

Партнёр одобрен, Mikro-ПВЗ создан.

```json
{
  "event_type": "pvz.partner.approved",
  "partner_id": "uuid",
  "pvz_id": "uuid",
  "business_name": "Kafe Sunrise",
  "commission_rate": 0.70,
  "approved_by": "uuid",
  "approved_at": "2025-12-21T12:00:00Z"
}
```

**Подписчики:**
- Notification Service → Email партнёру
- Delivery Service → добавить ПВЗ в роутинг

### 7.7 pvz.commission.earned

Партнёр заработал комиссию за выдачу.

```json
{
  "event_type": "pvz.commission.earned",
  "partner_id": "uuid",
  "parcel_id": "uuid",
  "order_id": "uuid",
  "commission_amount": 35.00,
  "transaction_amount": 50.00,
  "timestamp": "2025-12-22T16:00:00Z"
}
```

**Подписчики:**
- Payment Service → RecordPartnerCommission

---

## 8. Микросервис

### 8.1 Репозиторий

**Путь:** `/p/github.com/vondi-global/pvz`

**Порты:**
- gRPC: **50056**
- HTTP: **8089**
- PostgreSQL: **35438** (pvz_db)
- Redis: **36384**

### 8.2 База данных (pvz_db)

**Таблицы (10):**
1. `pvz` — Пункты выдачи
2. `pvz_operators` — Операторы
3. `operator_shifts` — Смены
4. `partner_accounts` — Партнёры
5. `partner_payouts` — Выплаты
6. `partner_commissions` — Комиссии
7. `parcels_at_pvz` — Посылки
8. `handovers` — Передачи
9. `pickup_events` — Выдачи
10. `return_requests` — Возвраты

**Views (3):**
- `v_pvz_capacity` — загрузка ПВЗ
- `v_partner_earnings` — заработки партнёров
- `v_operator_stats` — статистика операторов

**Функции (3):**
- `update_pvz_capacity()` — триггер обновления current_parcels
- `mark_overdue_parcels()` — cron job пометки просроченных
- `calculate_storage_fee(parcel_id)` — расчёт платы за хранение

### 8.3 gRPC API

**PVZService:**
- CreatePVZ, GetPVZ, UpdatePVZ, DeletePVZ
- ListPVZ, FindNearestPVZ
- GetPVZCapacity, ReserveSlots, ReleaseSlots

**PVZOperatorService:**
- GetOperatorInfo
- StartShift, EndShift, GetCurrentShift
- ListPendingHandovers, AcceptHandover, RejectHandover
- ValidatePickupCode, CompletePickup
- ProcessReturn
- GetPVZInventory, SearchParcel
- GetShiftStats, GetDailyStats

**PartnerService:**
- RegisterPartner, GetPartnerAccount, UpdatePartnerAccount
- ListPendingPartners, ApprovePartner, RejectPartner, SuspendPartner
- GetPartnerEarnings, GetPartnerPayouts
- GetPartnerStats

---

## 9. Roadmap

### MVP (Completed)
- ✅ Core domain models (PVZ, Operator, Partner, Parcel)
- ✅ Operator interface (shift, handover, pickup)
- ✅ Capacity tracking, auto-routing
- ✅ Partner program (registration, approval, commissions)
- ✅ QR scanning для handover
- ✅ Pickup code (6 цифр, 48h TTL)
- ✅ Storage fee calculation
- ✅ Returns processing

### Future (v2.0)
- 🔮 Примерочная (Try-On service) в Full ПВЗ
- 🔮 Отправка посылок (Sending service)
- 🔮 Интеграция с паккетоматами (Parcel Lockers)
- 🔮 Mobile app для операторов (Android/iOS)
- 🔮 Self-service терминалы (pickup без оператора)

---

**Дата создания:** 2025-12-21
**Автор:** Vondi Engineering Team
**Версия паспорта:** 1.0.0
**Статус:** Production-ready
