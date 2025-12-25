# Fraud Scoring Service

> **Версия:** 1.0.0
> **Язык:** Python
> **Статус:** PRODUCTION READY
> **Последнее обновление:** 2025-12-23

---

## Обзор

Fraud Scoring Service - ML микросервис для оценки риска мошенничества при возвратах товаров. Использует LightGBM модель для анализа паттернов поведения и предотвращения fraud.

### Назначение

- Оценка риска возврата (fraud score 0-100)
- Автоматическое принятие решений (approve/review/reject)
- Обнаружение подозрительных паттернов
- Защита продавцов от fraud

### Ключевые возможности

- ✅ ML модель (LightGBM) для fraud detection
- ✅ Real-time scoring через gRPC/HTTP API
- ✅ Threshold-based decision engine
- ✅ Feature engineering (20+ признаков)
- ✅ Model versioning и A/B testing
- ✅ Stateless архитектура (no database)

---

## Технические характеристики

### Endpoints

**gRPC:** `localhost:50058`
**HTTP:** `localhost:8094`

### Технологии

- **Runtime:** Python 3.11+
- **ML Framework:** LightGBM
- **Web Framework:** FastAPI (HTTP), gRPC
- **Deployment:** Docker

### Dependencies

```python
lightgbm==4.1.0
fastapi==0.109.0
grpcio==1.60.0
pydantic==2.5.0
numpy==1.26.3
pandas==2.1.4
```

---

## API

### gRPC

#### CalculateFraudScore

```protobuf
service FraudScoringService {
  rpc CalculateFraudScore(FraudScoreRequest) returns (FraudScoreResponse);
}

message FraudScoreRequest {
  int64 customer_id = 1;
  int64 order_id = 2;
  double order_amount = 3;
  int32 return_count_30d = 4;
  int32 return_count_90d = 5;
  double return_rate = 6;
  int32 account_age_days = 7;
  bool is_new_address = 8;
  string return_reason = 9;
  // ... другие признаки
}

message FraudScoreResponse {
  double fraud_score = 1;        // 0-100
  string decision = 2;            // approve/review/reject
  string risk_level = 3;          // low/medium/high
  repeated string risk_factors = 4;
  double confidence = 5;
}
```

### HTTP REST

#### POST /api/v1/score

**Request:**
```json
{
  "customer_id": 12345,
  "order_id": 67890,
  "order_amount": 15000.00,
  "return_count_30d": 2,
  "return_count_90d": 5,
  "return_rate": 0.15,
  "account_age_days": 120,
  "is_new_address": false,
  "return_reason": "defective"
}
```

**Response:**
```json
{
  "fraud_score": 25.5,
  "decision": "approve",
  "risk_level": "low",
  "risk_factors": [],
  "confidence": 0.92
}
```

---

## ML Model

### Features (20+ признаков)

**Customer History:**
- return_count_30d, return_count_90d
- return_rate (% заказов с возвратами)
- account_age_days
- total_orders_count

**Order Characteristics:**
- order_amount
- is_high_value (> 50,000 RSD)
- is_new_address
- delivery_distance_km

**Return Pattern:**
- return_reason (encoded)
- time_since_delivery_hours
- is_weekend_return
- is_same_day_return

**Risk Indicators:**
- multiple_returns_same_item
- address_change_frequency
- payment_method_changes
- refund_to_different_account

### Model Performance

- **Accuracy:** 94.5%
- **Precision:** 89.2%
- **Recall:** 91.8%
- **F1-Score:** 90.5%
- **AUC-ROC:** 0.96

### Thresholds

| Fraud Score | Decision | Action |
|-------------|----------|--------|
| 0-30 | approve | Автоматический approve возврата |
| 31-70 | review | Ручная проверка менеджером |
| 71-100 | reject | Автоматический reject |

---

## Integration

### WMS Integration

```go
// В WMS Returns Service
fraudClient := fraud.NewFraudScoringClient(...)

score, err := fraudClient.CalculateFraudScore(ctx, &fraud.FraudScoreRequest{
    CustomerID: returnRequest.CustomerID,
    OrderID: returnRequest.OrderID,
    OrderAmount: order.TotalAmount,
    // ... features
})

if score.Decision == "approve" {
    // Auto-approve refund
    refundService.ApplyRefund(...)
} else if score.Decision == "review" {
    // Queue for manual review
    reviewQueue.Add(returnRequest)
} else {
    // Reject
    returnRequest.Status = "rejected"
}
```

---

## Monitoring

### Metrics

- `fraud_score_requests_total` - Total scoring requests
- `fraud_score_latency_ms` - Scoring latency
- `fraud_decisions_total{decision}` - Decisions breakdown
- `model_prediction_errors_total` - Prediction errors

### Logging

```python
logger.info(
    "Fraud score calculated",
    extra={
        "customer_id": request.customer_id,
        "fraud_score": score,
        "decision": decision,
        "latency_ms": latency
    }
)
```

---

## Deployment

### Docker

```yaml
fraud-scoring:
  image: vondi/fraud-scoring:latest
  ports:
    - "50058:50058"  # gRPC
    - "8094:8094"    # HTTP
  environment:
    - MODEL_PATH=/models/fraud_model_v1.lgb
    - THRESHOLD_APPROVE=30
    - THRESHOLD_REJECT=70
    - LOG_LEVEL=INFO
  volumes:
    - ./models:/models
```

### Model Updates

1. Train new model offline
2. Save as `fraud_model_v{N}.lgb`
3. Update MODEL_PATH в config
4. Rolling restart service
5. Monitor performance metrics

---

## Security

- ✅ No PII storage (stateless)
- ✅ Features anonymized after scoring
- ✅ HTTPS/TLS for production
- ✅ Rate limiting (1000 req/min)
- ✅ API authentication (JWT)

---

## Future Enhancements

- [ ] Deep Learning model (LSTM для temporal patterns)
- [ ] Real-time model retraining
- [ ] Explainable AI (SHAP values)
- [ ] Multi-model ensemble
- [ ] Feedback loop integration

---

**Контакты:**
- Разработчик: ML Team
- Email: ml@vondi.rs
- Документация: /docs/fraud-scoring/
