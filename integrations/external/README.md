# External Integrations - System Passport

Этот раздел содержит технические паспорта внешних интеграций платформы VONDI.

---

## 📦 Delivery Providers (Провайдеры доставки)

### Post Express Serbia

**Файл**: [post-express.md](./post-express.md)

**Статус**: Production Ready

**Описание**: Интеграция с Post Express (Pošta Srbije) — курьерская доставка и сеть parcel lockers по всей Сербии.

**Основные возможности**:
- ✅ Курьерская доставка (door-to-door)
- ✅ Parcel Lockers (100+ пунктов выдачи)
- ✅ COD (наложенный платеж с автопереводом)
- ✅ Real-time tracking
- ✅ Расчёт тарифов (TX 11)
- ✅ Валидация адресов (TX 6)
- ✅ Печать этикеток (PDF)

**API**: WSP B2B (Web Service Platform)

**Endpoint**: https://wsp.posta.rs/api

**Документация**: 983 строки (API reference, examples, troubleshooting)

---

## 🔜 Planned Integrations

### D-Express (TODO)
- Альтернативный курьер для Сербии
- API документация: pending

### Bex Express (TODO)
- Региональная доставка (Балканы)
- API документация: pending

### DHL Express (TODO)
- Международная доставка
- API документация: pending

---

## 📚 Navigation

- **Main Passport**: `/p/github.com/vondi-global/SYSTEM_PASSPORT.md`
- **Delivery Service**: `/p/github.com/vondi-global/.passport/services/delivery.md`
- **Internal Integrations**: `/p/github.com/vondi-global/.passport/integrations/grpc-contracts.md`
- **Events**: `/p/github.com/vondi-global/.passport/integrations/events.md`

---

**Last Updated**: 2025-12-21
