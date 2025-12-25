# Post Express Serbia - Integration Documentation

> **Status**: Active Production Integration
> **Provider**: Post Express (Pošta Srbije)
> **Service Type**: B2B Courier & Parcel Delivery
> **API Type**: WSP Transaction-based API
> **Partner ID**: 10109 (vondi.rs)
> **Documentation**: https://www.posta.rs/wsp-help/

---

## 1. Обзор Post Express

### 1.1 Описание сервиса

Post Express — курьерская служба Pošta Srbije (Почта Сербии), предоставляющая услуги:
- **Курьерская доставка** — PE_Danas_za_sutra (Today for Tomorrow), PE_Danas_za_danas (Same Day)
- **Parcel Lockers (Паккетоматы)** — Сеть автоматических пунктов выдачи по всей Сербии
- **COD (Cash on Delivery)** — Наложенный платеж с автоматическим переводом средств
- **Tracking** — Полное отслеживание статуса доставки

### 1.2 Возможности

| Функция | Поддержка | Примечание |
|---------|-----------|------------|
| Курьерская доставка | ✅ | До двери (door-to-door) |
| Самовывоз (ПВЗ) | ✅ | 100+ parcel lockers |
| COD (наложенный платеж) | ✅ | Авто-перевод на банк счёт |
| Страхование посылок | ✅ | До объявленной стоимости |
| Отслеживание (tracking) | ✅ | Real-time события |
| SMS уведомления | ✅ | Автоматические |
| Расчёт тарифа | ✅ | TX 11 API |
| Валидация адреса | ✅ | TX 6 API |
| Печать этикеток | ✅ | PDF формат |
| Webhook callbacks | ⚠️ | Not implemented yet |
| Возвраты | ❌ | Planned for v2 |

### 1.3 Географическое покрытие

- **Внутренние отправления**: Вся Сербия (ID страны = 0)
- **Международные**: Planned for future releases
- **Parcel Lockers**: Belgrade, Novi Sad, Niš, Kragujevac, Subotica и др.

---

## 2. API Credentials и Endpoints

### 2.1 Production Environment

```bash
# Endpoint
POST_EXPRESS_API_URL=https://wsp.posta.rs/api

# Credentials (из договора с Post Express)
POST_EXPRESS_USERNAME=your_username
POST_EXPRESS_PASSWORD=your_password

# Брендинг и склад
POST_EXPRESS_BRAND=VONDI
POST_EXPRESS_WAREHOUSE=WAREHOUSE1

# Partner ID (выдается Post Express)
POST_EXPRESS_PARTNER_ID=10109  # vondi.rs partner ID
```

### 2.2 COD Configuration (Наложенный платеж)

```bash
# Банковский счёт для перевода откупнины
POST_EXPRESS_BANK_ACCOUNT=160-12345678-90

# Шифра плаћања (код платежа)
POST_EXPRESS_PAYMENT_CODE=189

# Модель платежа
POST_EXPRESS_PAYMENT_MODEL=97
```

### 2.3 Optional Settings

```bash
# Timeout и retry
POST_EXPRESS_TIMEOUT_SECONDS=30
POST_EXPRESS_RETRY_ATTEMPTS=3
```

### 2.4 API Architecture

**WSP (Web Service Platform) B2B API**:
- **Протокол**: HTTP/HTTPS POST (JSON)
- **Формат**: JSON with embedded JSON strings (StrIn, StrOut)
- **Аутентификация**: Username/Password в ClientData
- **Сервис ID**: 101 (B2B партнёры)
- **Транзакции**: 3, 4, 6, 9, 10, 11, 63, 73 и др.

---

## 3. Сервисы Post Express (IdRukovanje)

### 3.1 Доступные услуги доставки

| ID | Код | Название | Описание | Время доставки |
|----|-----|----------|----------|----------------|
| 30 | PE_Danas_za_danas | Today for Today | Express same-day | До конца дня |
| 55 | PE_Danas_za_odmah | Immediate Delivery | Super express | 2-4 часа |
| 58 | PE_Danas_za_sutra_19 | Tomorrow until 19:00 | Next day evening | До 19:00 следующего дня |
| 59 | PE_Danas_za_odmah_Bg | Immediate (Belgrade only) | Belgrade express | 2-4 часа (только Белград) |
| 71 | PE_Danas_za_sutra_isporuka | **Standard Delivery** | Default option | 1-2 рабочих дня |
| 85 | PE_Klasicna | Parcel Locker | Self-pickup | 2-5 рабочих дней |

### 3.2 Дополнительные услуги (PosebneUsluge)

Передаются как **строка через запятую** (не массив!):

```json
{
  "PosebneUsluge": "PNA,OTK,VD"
}
```

| Код | Название | Обязательность | Описание |
|-----|----------|----------------|----------|
| `PNA` | Pickup Notification | Обязательно для курьера | Уведомление о заборе |
| `OTK` | Cash on Delivery | Для COD | Наложенный платеж |
| `VD` | Valuable Delivery | Для COD | Ценная доставка |
| `SMS` | SMS Notification | Опционально | SMS получателю |

---

## 4. Rate Calculation API (TX 11)

### 4.1 PostarinaPosiljke (Расчёт стоимости доставки)

**Transaction ID**: 11

**Request**:
```json
{
  "IdRukovanje": 71,
  "IdZemlja": 0,
  "PostanskiBrojOdlaska": "11000",
  "PostanskiBrojDolaska": "21000",
  "Masa": 2500,
  "Otkupnina": 500000,
  "Vrednost": 500000,
  "PosebneUsluge": "OTK,VD"
}
```

**Response**:
```json
{
  "Rezultat": 0,
  "Poruka": "",
  "Postarina": 29500,
  "IdRukovanje": 71,
  "NazivUsluge": "PE Danas za sutra isporuka",
  "PostanskiBrojOdlaska": "11000",
  "PostanskiBrojDolaska": "21000",
  "Masa": 2500,
  "Otkupnina": 500000,
  "Vrednost": 500000,
  "PosebneUsluge": "OTK,VD",
  "Napomena": ""
}
```

### 4.2 Важные моменты

- **Masa** — вес в **граммах** (integer), не в кг!
- **Postarina** — стоимость в **para** (1 RSD = 100 para). Пример: 29500 para = 295 RSD
- **Otkupnina** и **Vrednost** — также в **para**
- **IdZemlja = 0** — для внутренних отправлений (Сербия)
- **Rezultat = 0** — успех, иначе ошибка

### 4.3 Volumetric Weight

Для объёмных посылок используется volumetric weight:

```go
volumetricWeight := (length_cm * width_cm * height_cm) / 5000.0
finalWeight := max(actualWeight, volumetricWeight)
```

---

## 5. Shipment Creation API (TX 73)

### 5.1 B2B Manifest API

**Transaction ID**: 73 (CreateManifest)

**Архитектура**: Создание происходит через **манифест** (Manifest), содержащий заказы (Porudzbine) и посылки (Posiljke).

```
Manifest
 └─ Porudzbine[] (Orders)
     └─ Posiljke[] (Shipments)
```

### 5.2 Manifest Request Structure

```json
{
  "ExtIdManifest": "MANIFEST-1703253600",
  "IdTipPosiljke": 1,
  "Posiljalac": {
    "Naziv": "VONDI Warehouse",
    "Adresa": {
      "Ulica": "Takovska",
      "Broj": "2",
      "Mesto": "Beograd",
      "PostanskiBroj": "11000",
      "OznakaZemlje": "RS"
    },
    "Mesto": "Beograd",
    "PostanskiBroj": "11000",
    "Telefon": "+381641234567",
    "Email": "b2b@vondi.rs",
    "OznakaZemlje": "RS"
  },
  "Porudzbine": [
    {
      "ExtIdPorudzbina": "ORDER-1703253600",
      "Posiljke": [
        {
          "ExtBrend": "VONDI",
          "ExtMagacin": "WAREHOUSE1",
          "ExtReferenca": "VONDI-REF-1703253600",
          "NacinPrijema": "K",
          "ImaPrijemniBrojDN": "N",
          "NacinPlacanja": "POF",
          "Posiljalac": { ... },
          "MestoPreuzimanja": { ... },
          "BrojPosiljke": "VONDI-1703253600",
          "IdRukovanje": 71,
          "Primalac": {
            "Naziv": "Petar Petrovic",
            "Adresa": {
              "Ulica": "Bulevar oslobodjenja",
              "Broj": "1",
              "Mesto": "Novi Sad",
              "PostanskiBroj": "21000",
              "OznakaZemlje": "RS"
            },
            "Mesto": "Novi Sad",
            "PostanskiBroj": "21000",
            "Telefon": "+381621234567",
            "OznakaZemlje": "RS",
            "TipAdrese": "S"
          },
          "Masa": 2500,
          "Otkupnina": 500000,
          "Vrednost": 500000,
          "OtkupninaTekuciRacun": "160-12345678-90",
          "OtkupninaModelPNB": "97",
          "OtkupninaPNB": "7032536000",
          "OtkupninaSifraPlacanja": "189",
          "OtkupninaVrstaDokumenta": "N",
          "PosebneUsluge": "PNA,OTK,VD",
          "Sadrzaj": "Товары из интернет-магазина",
          "ReferencaBroj": "VONDI-1703253600",
          "Napomena": "Fragile items"
        }
      ]
    }
  ],
  "DatumPrijema": "2023-12-22",
  "VremePrijema": "14:30",
  "IdPartnera": 10109,
  "NazivManifesta": "VONDI-20231222-143000"
}
```

### 5.3 Обязательные поля B2B Shipment

| Поле | Тип | Описание | Пример |
|------|-----|----------|--------|
| `ExtBrend` | string | Бренд (из договора) | "VONDI" |
| `ExtMagacin` | string | Склад | "WAREHOUSE1" |
| `ExtReferenca` | string | Уникальная референция | "VONDI-REF-123" |
| `NacinPrijema` | string | K=курьер, O=офис | "K" |
| `ImaPrijemniBrojDN` | string | D=есть приёмный номер, N=нет | "N" |
| `NacinPlacanja` | string | POF, N, K | "POF" |
| `Posiljalac` | object | Отправитель | {...} |
| `MestoPreuzimanja` | object | Место забора (для курьера!) | {...} |
| `BrojPosiljke` | string | Номер посылки | "VONDI-123" |
| `IdRukovanje` | int | ID услуги | 71 |
| `Primalac` | object | Получатель | {...} |
| `Masa` | int | Вес в **граммах** | 2500 |

### 5.4 Структура адреса (AddressInfo)

**ВАЖНО**: Адрес — это **объект**, не строка!

```json
{
  "Ulica": "Takovska",
  "Broj": "2",
  "Mesto": "Beograd",
  "PostanskiBroj": "11000",
  "OznakaZemlje": "RS"
}
```

### 5.5 COD (Otkupnina) Configuration

Для наложенного платежа:

```json
{
  "Otkupnina": 500000,
  "Vrednost": 500000,
  "OtkupninaTekuciRacun": "160-12345678-90",
  "OtkupninaModelPNB": "97",
  "OtkupninaPNB": "7032536000",
  "OtkupninaSifraPlacanja": "189",
  "OtkupninaVrstaDokumenta": "N",
  "PosebneUsluge": "PNA,OTK,VD"
}
```

**Важные моменты**:
- `Otkupnina` — **простое число** в para, НЕ объект!
- `Vrednost` должна быть минимум равна `Otkupnina`
- `OtkupninaTekuciRacun` — банковский счёт для перевода откупнины
- `OtkupninaModelPNB` — модель платежа (обычно "97")
- `OtkupninaPNB` — позив на број (payment reference), генерируется из timestamp
- `OtkupninaSifraPlacanja` — код платежа (обычно "189")
- `OtkupninaVrstaDokumenta` — "N" = налоговый документ

### 5.6 Parcel Locker (IdRukovanje = 85)

Для доставки в паккетомат:

```json
{
  "IdRukovanje": 85,
  "Primalac": {
    "TipAdrese": "P",
    "PAK": "BG-PL-001"
  },
  "ParcelLockerCode": "BG-PL-001"
}
```

**Обязательно**:
- `IdRukovanje = 85`
- `ParcelLockerCode` — код паккетомата (получить через TX 10)
- `Primalac.TipAdrese = "P"` (Parcel Locker)
- `Primalac.PAK` — дублирует ParcelLockerCode

---

## 6. Label Generation

### 6.1 Получение URL этикетки

Post Express возвращает URL этикетки в ответе манифеста:

```json
{
  "Rezultat": 0,
  "IdManifesta": 12345,
  "Porudzbine": [
    {
      "Posiljke": [
        {
          "PrijemniBroj": "PE123456789",
          "IDPosiljke": 999,
          "LabelURL": "https://wsp.posta.rs/labels/PE123456789.pdf"
        }
      ]
    }
  ]
}
```

### 6.2 Печать этикетки (TX 20)

**Transaction ID**: 20 (PrintLabel)

**Request**:
```json
{
  "IdPosiljke": "999",
  "TipFajla": "PDF"
}
```

**Response**:
```json
{
  "Rezultat": 0,
  "SadrzajPDF": "<base64 encoded PDF>"
}
```

**TODO**: Реализовать декодирование base64 в []byte для сохранения файла.

---

## 7. Tracking API (TX 63)

### 7.1 GetShipmentStatus (Отслеживание)

**Transaction ID**: 63

**Request**:
```json
{
  "BrojPosiljke": "PE123456789"
}
```

**Response**:
```json
{
  "Rezultat": 0,
  "BrojPosiljke": "PE123456789",
  "Status": "DELIVERED",
  "StatusText": "Доставлено",
  "Dogadjaji": [
    {
      "Datum": "2024-01-15",
      "Vreme": "10:30",
      "Mesto": "Belgrade",
      "Opis": "Order confirmed",
      "Kod": "confirmed"
    },
    {
      "Datum": "2024-01-16",
      "Vreme": "14:00",
      "Mesto": "Warehouse",
      "Opis": "In transit",
      "Kod": "in_transit"
    },
    {
      "Datum": "2024-01-17",
      "Vreme": "09:15",
      "Mesto": "Novi Sad",
      "Opis": "Delivered to recipient",
      "Kod": "delivered"
    }
  ]
}
```

### 7.2 Status Mapping

| Post Express Status | Internal Status | Description |
|---------------------|-----------------|-------------|
| `confirmed` | `confirmed` | Заказ подтверждён |
| `picked_up` | `picked_up` | Забрано курьером |
| `in_transit` | `in_transit` | В пути |
| `arrived` | `arrived` | Прибыло в пункт назначения |
| `out_for_delivery` | `out_for_delivery` | Передано курьеру для доставки |
| `DELIVERED` | `delivered` | Доставлено |
| `delivery_attempted` | `delivery_attempted` | Попытка доставки неудачна |
| `CANCELED` | `canceled` | Отменено |
| `returning` | `returning` | Возврат отправителю |
| `returned` | `returned` | Возвращено |

---

## 8. Webhook Callbacks

### 8.1 Status

⚠️ **Not Implemented Yet** — Planned for v2

### 8.2 Expected Payload Structure

```json
{
  "tracking_number": "PE123456789",
  "status": "delivered",
  "event_time": "2024-01-17T09:15:00Z",
  "location": "Novi Sad",
  "description": "Delivered to recipient",
  "proof_of_delivery": {
    "recipient_name": "Petar Petrovic",
    "signature_url": "https://..."
  }
}
```

### 8.3 Handler Implementation (TODO)

```go
func (a *PostExpressAdapter) HandleWebhook(ctx context.Context, payload []byte, headers map[string]string) (*WebhookResponse, error) {
    // TODO: Parse webhook payload
    // TODO: Validate signature (if provided)
    // TODO: Update shipment status in database
    // TODO: Send notification to customer
    return &WebhookResponse{
        Processed: true,
        Timestamp: time.Now(),
    }, nil
}
```

---

## 9. Error Codes

### 9.1 Transaction-Level Errors

| Rezultat | Значение | Действие |
|----------|----------|----------|
| 0 | Успех | Proceed with response |
| 1 | Общая ошибка | Проверить Poruka для деталей |
| 2 | Валидационная ошибка | Проверить GreskeValidacije[] |
| 3 | Ошибка аутентификации | Проверить credentials |
| 4 | Услуга недоступна | Попробовать позже |

### 9.2 Validation Errors (GreskeValidacije)

```json
{
  "GreskeValidacije": [
    {
      "Polje": "Primalac.PostanskiBroj",
      "Vrednost": "99999",
      "Poruka": "Nevažeći poštanski broj"
    }
  ]
}
```

### 9.3 Common Errors

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `"ExtIdManifest is required"` | Не указан ExtIdManifest | Добавить уникальный ID манифеста |
| `"Невалидан поштански број"` | Неверный почтовый индекс | Проверить формат (5 цифр) |
| `"Услуга није доступна"` | Услуга недоступна в данном городе | Использовать TX 9 для проверки |
| `"Masa mora biti veća od 0"` | Вес = 0 | Установить минимум 500 грамм |
| `"Vrednost mora biti >= Otkupnina"` | Vrednost < Otkupnina | Vrednost >= Otkupnina |

---

## 10. Примеры запросов/ответов

### 10.1 TX 3: GetSettlements (Поиск города)

**Request**:
```json
{
  "Naziv": "Beograd"
}
```

**Response**:
```json
{
  "Rezultat": 0,
  "Naselja": [
    {
      "Id": 1070,
      "Naziv": "BEOGRAD-VOŽDOVAC",
      "PostanskiBroj": "11000",
      "IdOkrug": 10,
      "NazivOkruga": "BEOGRADSKI OKRUG"
    }
  ]
}
```

### 10.2 TX 10: GetParcelLockers (Получение паккетоматов)

**Request**:
```json
{
  "IdNaselje": 1070
}
```

**Response**:
```json
{
  "Rezultat": 0,
  "PostanskeJedinice": [
    {
      "IdPoste": 15001,
      "Sifra": "BG-PL-001",
      "Naziv": "Parcel Locker - Knez Mihailova",
      "Adresa": "Knez Mihailova 12",
      "IdNaselje": 1070,
      "NazivNaselja": "BEOGRAD-VOŽDOVAC",
      "PostanskiBroj": "11000",
      "Telefon": "+381113334444",
      "Email": "",
      "RadnoVreme": "24/7",
      "TipPoste": "PL",
      "Latitude": 44.81784,
      "Longitude": 20.45906,
      "Dostupnost": true
    }
  ]
}
```

### 10.3 TX 11: CalculatePostage (Расчёт тарифа с COD)

**Request**:
```json
{
  "IdRukovanje": 71,
  "IdZemlja": 0,
  "PostanskiBrojOdlaska": "11000",
  "PostanskiBrojDolaska": "21000",
  "Masa": 2500,
  "Otkupnina": 500000,
  "Vrednost": 500000,
  "PosebneUsluge": "OTK,VD"
}
```

**Response**:
```json
{
  "Rezultat": 0,
  "Postarina": 34500,
  "IdRukovanje": 71,
  "NazivUsluge": "PE Danas za sutra isporuka",
  "Masa": 2500,
  "Otkupnina": 500000,
  "Vrednost": 500000,
  "PosebneUsluge": "OTK,VD"
}
```

**Расшифровка**:
- Postarina: 34500 para = **345 RSD**
- Otkupnina: 500000 para = **5000 RSD** (COD сумма)
- Masa: 2500 грамм = **2.5 кг**

### 10.4 TX 73: CreateManifest (Создание посылки)

**Full Request Example** — см. секцию 5.2

**Response**:
```json
{
  "Rezultat": 0,
  "Poruka": "",
  "IdManifesta": 12345,
  "ExtIdManifest": "MANIFEST-1703253600",
  "Porudzbine": [
    {
      "BrojPorudzbine": "ORDER-1703253600",
      "Posiljke": [
        {
          "BrojPosiljke": "VONDI-1703253600",
          "IdPosiljke": 999,
          "PrijemniBroj": "PE123456789",
          "Status": "created",
          "Rezultat": 0,
          "Poruka": "",
          "LabelUrl": "https://wsp.posta.rs/labels/PE123456789.pdf",
          "Postarina": 34500
        }
      ]
    }
  ],
  "GreskeValidacije": []
}
```

**Key Fields**:
- `IdManifesta` — ID манифеста в системе Post Express
- `PrijemniBroj` — **Трек-номер** для отслеживания (PE123456789)
- `LabelUrl` — URL для скачивания этикетки
- `Postarina` — Финальная стоимость в para (34500 = 345 RSD)

---

## 11. Code Examples

### 11.1 Go Service Integration

**Location**: `/p/github.com/vondi-global/delivery/internal/gateway/provider/postexpress_adapter.go`

```go
// NewPostExpressAdapter создает адаптер Post Express
func NewPostExpressAdapter(config *postexpress.Config, logger *zerolog.Logger) (*PostExpressAdapter, error) {
    wspConfig := &service.WSPConfig{
        Endpoint:        config.APIURL,
        Username:        config.Username,
        Password:        config.Password,
        Language:        "SR",
        DeviceType:      "2",
        Timeout:         config.Timeout,
        MaxRetries:      config.RetryAttempts,
        RetryDelay:      time.Second * 2,
        TestMode:        !config.IsProduction,
        DeviceName:      "VONDI-API",
        ApplicationName: "VONDI Marketplace",
        Version:         "1.0.0",
        PartnerID:       10109,
        BankAccount:     config.BankAccount,
        PaymentCode:     config.PaymentCode,
        PaymentModel:    config.PaymentModel,
    }

    wspClient := service.NewWSPClient(wspConfig)

    return &PostExpressAdapter{
        wspClient:    wspClient,
        config:       config,
        logger:       logger,
        providerCode: "post_express",
    }, nil
}
```

### 11.2 Calculate Rate Example

```go
func (a *PostExpressAdapter) CalculateRate(ctx context.Context, req *RateRequest) (*RateResponse, error) {
    // Конвертируем в граммы
    totalWeightGrams := totalWeightKg * 1000

    // Конвертируем в para
    codPara := int(req.CODAmount * 100)
    valuePara := int(req.InsuranceValue * 100)

    peReq := &postexpress.PostageCalculationRequest{
        IdRukovanje:          71,
        IdZemlja:             0,
        PostanskiBrojOdlaska: req.FromAddress.PostalCode,
        PostanskiBrojDolaska: req.ToAddress.PostalCode,
        Masa:                 int(totalWeightGrams),
        Otkupnina:            codPara,
        Vrednost:             valuePara,
        PosebneUsluge:        "OTK,VD",
    }

    resp, err := a.wspClient.CalculatePostage(ctx, peReq)
    if err != nil {
        return nil, err
    }

    totalCost := float64(resp.Postarina) / 100.0 // para → RSD

    return &RateResponse{
        ProviderCode: "post_express",
        DeliveryOptions: []DeliveryOption{
            {
                Type:          DeliveryTypeStandard,
                Name:          resp.NazivUsluge,
                TotalCost:     totalCost,
                EstimatedDays: 2,
            },
        },
        Currency: "RSD",
    }, nil
}
```

### 11.3 Create Shipment Example

```go
func (a *PostExpressAdapter) CreateShipment(ctx context.Context, req *ShipmentRequest) (*ShipmentResponse, error) {
    wspReq := &service.WSPShipmentRequest{
        SenderName:          req.FromAddress.Name,
        SenderAddress:       req.FromAddress.Street,
        SenderCity:          req.FromAddress.City,
        SenderPostalCode:    req.FromAddress.PostalCode,
        SenderPhone:         req.FromAddress.Phone,
        RecipientName:       req.ToAddress.Name,
        RecipientAddress:    req.ToAddress.Street,
        RecipientCity:       req.ToAddress.City,
        RecipientPostalCode: req.ToAddress.PostalCode,
        RecipientPhone:      req.ToAddress.Phone,
        Weight:              totalWeight,
        CODAmount:           req.CODAmount,
        InsuranceAmount:     req.InsuranceValue,
        ServiceType:         "PE_Danas_za_sutra_isporuka",
        Content:             "Товары из интернет-магазина",
        Note:                req.Notes,
    }

    manifestResp, err := a.wspClient.CreateShipmentViaManifest(ctx, wspReq)
    if err != nil {
        return nil, err
    }

    if manifestResp.Rezultat != 0 {
        return nil, fmt.Errorf("manifest creation failed: %s", manifestResp.Poruka)
    }

    // Извлекаем tracking number и label URL
    shipment := manifestResp.Porudzbine[0].Posiljke[0]

    return &ShipmentResponse{
        ShipmentID:     fmt.Sprintf("%d", shipment.IDPosiljke),
        TrackingNumber: shipment.PrijemniBroj,
        Status:         StatusConfirmed,
        TotalCost:      float64(shipment.Postarina) / 100.0,
        Labels: []LabelInfo{
            {
                Type:   "shipping",
                Format: "pdf",
                URL:    shipment.LabelURL,
            },
        },
    }, nil
}
```

---

## 12. Testing

### 12.1 Unit Tests

**Location**: `/p/github.com/vondi-global/delivery/internal/gateway/provider/postexpress_adapter_test.go`

```bash
cd /p/github.com/vondi-global/delivery
go test -v ./internal/gateway/provider -run TestPostExpressAdapter
```

### 12.2 Integration Tests

```bash
# Require real credentials
POST_EXPRESS_USERNAME=test_user \
POST_EXPRESS_PASSWORD=test_pass \
go test -v ./internal/gateway/provider -run TestPostExpressAdapter_Integration
```

### 12.3 Test Coverage

```bash
go test -coverprofile=coverage.out ./internal/gateway/provider
go tool cover -html=coverage.out
```

---

## 13. Rate Limits & Performance

### 13.1 API Rate Limits

- **Production**: Не ограничено (по договору)
- **Recommended**: 100 requests/minute для стабильности
- **Retry Strategy**: Exponential backoff (2 сек между попытками)

### 13.2 Performance Metrics

| Operation | Avg Response Time | Timeout |
|-----------|-------------------|---------|
| CalculatePostage (TX 11) | 200-500ms | 30s |
| CreateManifest (TX 73) | 500-1000ms | 30s |
| GetShipmentStatus (TX 63) | 100-300ms | 30s |
| GetSettlements (TX 3) | 50-150ms | 30s |
| GetParcelLockers (TX 10) | 100-250ms | 30s |

### 13.3 Caching Strategy

- **Settlements (TX 3)**: Cache for 24 hours
- **Parcel Lockers (TX 10)**: Cache for 6 hours
- **Postage Rates (TX 11)**: Cache for 1 hour (зависит от тарифов)
- **Tracking (TX 63)**: No caching (real-time)

---

## 14. Troubleshooting

### 14.1 Common Issues

**Issue**: `"Nevažeći poštanski broj"`
**Solution**: Проверить формат почтового индекса (5 цифр, например "11000")

**Issue**: `"Vrednost mora biti >= Otkupnina"`
**Solution**: Установить `Vrednost >= Otkupnina` для COD отправлений

**Issue**: `"Услуга није доступна"`
**Solution**: Использовать TX 9 (CheckServiceAvailability) для проверки доступности услуги в городе

**Issue**: `"ExtIdManifest already exists"`
**Solution**: Использовать уникальный ID манифеста (timestamp + random)

**Issue**: `"Masa mora biti veća od 0"`
**Solution**: Минимальный вес 500 грамм (Masa=500)

### 14.2 Debug Logging

```bash
# Включить debug logging для диагностики
export LOG_LEVEL=debug

# Логи будут содержать:
# - JSON запросов (до 2000 символов)
# - HTTP status codes
# - Время выполнения (execution_time_ms)
# - Результаты валидации (GreskeValidacije)
```

### 14.3 Monitoring

**OpenTelemetry трейсинг**:
- Span name: `PostExpressAPICall`
- Attributes: `http.status_code`, `transaction.type`, `wsp.success`
- Errors: Автоматически записываются в span

---

## 15. Security Considerations

### 15.1 Credentials Storage

- **NEVER** commit credentials в репозиторий
- Использовать environment variables
- Для production — использовать secrets manager (AWS Secrets Manager, HashiCorp Vault)

### 15.2 TLS/SSL

```go
TLSClientConfig: &tls.Config{
    InsecureSkipVerify: config.TestMode, // ТОЛЬКО для тестов!
}
```

**Production**: Всегда `InsecureSkipVerify: false`

### 15.3 Input Validation

- Валидировать postal codes (5 цифр)
- Валидировать phone numbers (формат +381...)
- Санитизировать адреса (без спецсимволов)
- Лимитировать вес (max 50 кг)

---

## 16. Future Improvements

### 16.1 Planned Features (v2)

- [ ] Webhook callbacks для автоматического обновления статусов
- [ ] Возвраты (return shipments)
- [ ] Bulk operations (массовое создание посылок)
- [ ] Advanced tracking (proof of delivery с фото)
- [ ] SMS notifications API integration
- [ ] International shipments (IdZemlja != 0)

### 16.2 Optimization Opportunities

- [ ] Batch label printing (печать нескольких этикеток за раз)
- [ ] Redis caching для settlements/parcel lockers
- [ ] Async webhook processing
- [ ] Metrics dashboard (Grafana)

---

## 17. References

### 17.1 Official Documentation

- **WSP API Help**: https://www.posta.rs/wsp-help/
- **Transaction List**: https://www.posta.rs/wsp-help/transakcije/
- **B2B Manifest (TX 73)**: https://www.posta.rs/wsp-help/transakcije/b2b-manifest.aspx
- **Postage Calculation (TX 11)**: https://www.posta.rs/wsp-help/transakcije/postarina-posiljke.aspx

### 17.2 Internal Documentation

- **Delivery Service**: `/p/github.com/vondi-global/delivery/docs/API.md`
- **Provider Interface**: `/p/github.com/vondi-global/delivery/internal/gateway/provider/`
- **System Passport**: `/p/github.com/vondi-global/.passport/services/delivery.md`

### 17.3 Support Contacts

- **Post Express B2B Support**: b2b@posta.rs
- **Technical Issues**: Nikola Dmitrašinović (контакт в договоре)
- **Emergency**: +381 11 3334444 (24/7 hotline)

---

**Last Updated**: 2025-12-21
**Version**: 1.0.0
**Maintainer**: VONDI Development Team
