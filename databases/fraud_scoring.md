# Fraud Scoring - Storage

> **Тип:** Stateless ML Service
> **Статус:** PRODUCTION READY

---

## Обзор

Fraud Scoring Service НЕ использует постоянную базу данных. Сервис полностью stateless и работает только с ML моделью.

### Storage Strategy

**Model Files:**
- Локальное хранилище: `/models/fraud_model_v{N}.lgb`
- Формат: LightGBM binary
- Размер: ~5-10 MB
- Версионирование: semantic (v1, v2, ...)

**No Database:**
- Нет персистентного хранилища
- Нет feature store (features передаются в request)
- Нет history logs (только application logs)

**Rationale:**
- Максимальная производительность (no DB overhead)
- Простота deployment (no migrations)
- GDPR compliance (no PII storage)
- Легко масштабируется (stateless)

---

## Model Versioning

### Directory Structure

```
/models/
├── fraud_model_v1.lgb (production)
├── fraud_model_v2.lgb (candidate)
└── metadata.json
```

### Metadata

```json
{
  "current_version": "v1",
  "models": {
    "v1": {
      "path": "fraud_model_v1.lgb",
      "trained_at": "2025-12-01T10:00:00Z",
      "accuracy": 0.945,
      "auc_roc": 0.96
    },
    "v2": {
      "path": "fraud_model_v2.lgb",
      "trained_at": "2025-12-15T10:00:00Z",
      "accuracy": 0.950,
      "auc_roc": 0.97,
      "status": "testing"
    }
  }
}
```

---

## Caching (Optional)

**In-memory cache для частых запросов:**
- Redis опционально (TTL: 5 минут)
- Key: `fraud:score:{customer_id}:{features_hash}`
- Value: `{score, decision, timestamp}`

**НЕ используется для:**
- Permanent storage
- Audit trail (используй application logs)

---

## Monitoring Storage

**Metrics хранятся в Prometheus (не в DB):**
- fraud_score_requests_total
- fraud_score_latency_ms
- fraud_decisions_total

**Logs в stdout/stderr:**
- Scoring requests
- Model predictions
- Errors

---

## Compliance

✅ **GDPR Compliant:**
- No PII storage
- Scores не сохраняются
- Features передаются только в request
- Audit через application logs (retention: 30 days)

---

**Вывод:** Fraud Scoring Service не требует database. Stateless by design.
