# Паспорт: База данных pvz_db

> Обновлено: 2025-12-21

## Идентификация

| Параметр | Значение |
|----------|----------|
| Тип | PostgreSQL 15 Alpine |
| Image | postgres:15-alpine |
| Хост | localhost |
| Порт | 35438 |
| Имя БД | pvz_db |
| Пользователь | pvz_user |
| Пароль | pvz_secure_pass_2025 |
| Контейнер | vondi-dev-postgres-pvz |

## КРИТИЧНО

**Это ОТДЕЛЬНАЯ база данных PVZ Service!**
- НЕ используй порт 5433 (это монолит vondi_db)
- НЕ используй порт 25432 (это Auth Service)
- НЕ используй порт 35434 (это Listings microservice)
- PVZ Service имеет СВОЮ независимую БД
- **Назначение:** Управление сетью пунктов выдачи заказов (ПВЗ), партнёрская программа, операторы

## Подключение

```bash
# Через localhost
psql "postgres://pvz_user:pvz_secure_pass_2025@localhost:35438/pvz_db?sslmode=disable"

# Через Docker
docker exec -it vondi-dev-postgres-pvz psql -U pvz_user -d pvz_db

# Для дампа
PGPASSWORD=pvz_secure_pass_2025 pg_dump \
  -h localhost -p 35438 -U pvz_user -d pvz_db \
  --no-owner --no-acl --column-inserts --inserts \
  -f pvz_dump_$(date +%Y%m%d_%H%M%S).sql

# Для импорта
PGPASSWORD=pvz_secure_pass_2025 psql \
  -h localhost -p 35438 -U pvz_user -d pvz_db \
  -f pvz_dump.sql
```

## Статистика

| Параметр | Значение |
|----------|----------|
| Всего таблиц | 10 |
| Бизнес-таблиц | 10 |
| Функций | 3 |
| Триггеров | 5 |
| Миграций | 5 |
| Путь к миграциям | /p/github.com/vondi-global/pvz/migrations/ |

## Таблицы (полный список - 10 таблиц)

### 1. Пункты выдачи (PVZ)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| pvz | Пункты выдачи заказов (full и mikro) | BIGSERIAL |

**Особенности pvz:**
- Типы: `type` (`full` - собственный ПВЗ, `mikro` - партнёрский)
- Код: `code` (уникальный, формат PVZ-BG-001)
- Адрес: `name`, `address`, `city`, `postal_code`
- Геолокация: `latitude`, `longitude` (для поиска ближайших ПВЗ)
- Capacity: `max_capacity`, `current_parcels`, `reserved_slots`
- График работы: `working_hours` (JSONB с расписанием на 7 дней)
- Возможности: `supports_pickup`, `supports_return`, `supports_try_on`
- Статус: `status` (`active`, `inactive`, `temporary_closed`, `overloaded`)
- Финансы: `commission_rate` (decimal, для партнёрских ПВЗ)
- Партнёр: `partner_id` (FK к partner_accounts, NULL для full)

### 2. Операторы и смены (Operators & Shifts)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| pvz_operators | Операторы пунктов выдачи | BIGSERIAL |
| operator_shifts | Смены операторов | BIGSERIAL |

**Особенности pvz_operators:**
- Связь: `user_id` (FK к auth_db.users), `pvz_id` (FK к pvz)
- Роль: `role` (`operator` - рядовой, `manager` - управляющий)
- Статус: `is_active`
- Текущая смена: `current_shift_start` (NULL если смена не начата)
- Статистика: `total_handovers`, `total_pickups`, `total_returns`

**Особенности operator_shifts:**
- Оператор: `operator_id` (FK к pvz_operators), `pvz_id` (FK к pvz)
- Время: `shift_start`, `shift_end` (NULL если смена активна)
- Статистика: `parcels_accepted`, `parcels_issued`, `returns_processed`
- Финансы: `cash_collected`, `card_payments_processed`

### 3. Партнёры (Partners)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| partner_accounts | Аккаунты партнёров Mikro-ПВЗ | BIGSERIAL |
| partner_payouts | Выплаты партнёрам | BIGSERIAL |
| partner_commissions | Комиссии за выдачи | BIGSERIAL |

**Особенности partner_accounts:**
- Владелец: `user_id` (FK к auth_db.users)
- Бизнес: `business_name`, `business_type` (`cafe`, `shop`, `salon`, `pharmacy`, `gas_station`, `post_office`)
- Контакты: `contact_name`, `phone`, `email`
- Налоги: `tax_id`, `bank_account`
- Статус: `status` (`pending`, `active`, `suspended`, `rejected`)
- Заработки: `total_earnings` (decimal)

**Особенности partner_payouts:**
- Партнёр: `partner_id` (FK к partner_accounts)
- Сумма: `amount` (decimal), `currency` (default: RSD)
- Период: `period_start`, `period_end`
- Статус: `status` (`pending`, `processing`, `completed`, `failed`)
- Банк: `bank_reference`, `payout_date`

**Особенности partner_commissions:**
- Связь: `partner_id` (FK к partner_accounts), `parcel_id`, `order_id`
- Финансы: `commission_amount`, `transaction_amount`, `commission_rate`
- Статус: `status` (`pending`, `paid`, `cancelled`)
- Выплата: `payout_id` (FK к partner_payouts, NULL если не выплачено)

### 4. Посылки в ПВЗ (Parcels)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| parcels_at_pvz | Посылки, находящиеся в ПВЗ | BIGSERIAL |
| handovers | Передачи посылок от курьеров | BIGSERIAL |
| pickup_events | События выдачи посылок клиентам | BIGSERIAL |

**Особенности parcels_at_pvz:**
- Идентификация: `parcel_id` (уникальный), `order_id`, `customer_id`
- ПВЗ: `pvz_id` (FK к pvz)
- Pickup код: `pickup_code` (6 цифр), `pickup_code_expiry` (48 часов)
- Время: `arrived_at`, `storage_deadline` (21 день), `picked_up_at`, `returned_at`
- Статус: `status` (`awaiting_pickup`, `picked_up`, `returned`, `overdue`, `expired`)
- Финансы: `storage_fee` (decimal, 20 RSD/день после 3 дней)
- Попытки: `pickup_attempts` (max 5)

**Особенности handovers:**
- Связь: `parcel_id`, `pvz_id`, `courier_id`, `operator_id` (FK к pvz_operators)
- QR код: `qr_code` (для сканирования), `handover_time`
- Статус: `status` (`pending`, `accepted`, `rejected`)
- Детали: `notes`, `rejection_reason`, `photos[]` (массив URL)

**Особенности pickup_events:**
- Связь: `parcel_id`, `pvz_id`, `customer_id`, `operator_id`
- Код: `pickup_code` (6 цифр, используется для выдачи)
- Время: `pickup_time`, `code_validated_at`
- Оплата: `payment_method` (`cash`, `card`, `wallet`, `paid_online`)
- Финансы: `cod_amount` (decimal, если оплата при получении)

### 5. Возвраты (Returns)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| return_requests | Запросы на возврат товаров | BIGSERIAL |

**Особенности return_requests:**
- Связь: `parcel_id`, `order_id`, `customer_id`, `pvz_id`, `operator_id`
- Причина: `reason` (`wrong_item`, `damaged`, `defective`, `not_as_described`, `changed_mind`)
- Детали: `notes`, `photos[]` (массив URL фото товара)
- Статус: `status` (`pending`, `approved`, `rejected`, `completed`)
- Время: `requested_at`, `processed_at`, `completed_at`
- Возврат: `refund_amount` (decimal), `refund_status`

## Схемы ключевых таблиц

### pvz (Пункты выдачи)

```sql
CREATE TABLE pvz (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(20) UNIQUE NOT NULL,           -- PVZ-BG-001
    type VARCHAR(10) NOT NULL CHECK (type IN ('full', 'mikro')),
    status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'temporary_closed', 'overloaded')),

    -- Адрес и геолокация
    name VARCHAR(255) NOT NULL,
    address TEXT NOT NULL,
    city VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20),
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,

    -- Capacity tracking
    max_capacity INTEGER NOT NULL,
    current_parcels INTEGER DEFAULT 0,
    reserved_slots INTEGER DEFAULT 0,

    -- График работы
    working_hours JSONB NOT NULL,              -- {"monday": {"open": "09:00", "close": "21:00"}, ...}

    -- Возможности
    supports_pickup BOOLEAN DEFAULT TRUE,
    supports_return BOOLEAN DEFAULT TRUE,
    supports_try_on BOOLEAN DEFAULT FALSE,

    -- Финансы (для mikro ПВЗ)
    commission_rate DECIMAL(5,2),               -- 65.00 означает 65% партнёру

    -- Связь с партнёром
    partner_id BIGINT REFERENCES partner_accounts(id) ON DELETE SET NULL,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_pvz_status ON pvz(status);
CREATE INDEX idx_pvz_type ON pvz(type);
CREATE INDEX idx_pvz_city ON pvz(city);
CREATE INDEX idx_pvz_partner ON pvz(partner_id);
-- GiST index для геопоиска (требует PostGIS)
CREATE INDEX idx_pvz_location ON pvz USING GIST (ll_to_earth(latitude, longitude));
```

### parcels_at_pvz (Посылки в ПВЗ)

```sql
CREATE TABLE parcels_at_pvz (
    id BIGSERIAL PRIMARY KEY,
    parcel_id VARCHAR(50) UNIQUE NOT NULL,
    order_id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    pvz_id BIGINT NOT NULL REFERENCES pvz(id) ON DELETE RESTRICT,

    -- Pickup код
    pickup_code VARCHAR(6) NOT NULL,            -- 123456
    pickup_code_expiry TIMESTAMP NOT NULL,      -- TTL 48h
    pickup_attempts INTEGER DEFAULT 0,

    -- Даты
    arrived_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    storage_deadline TIMESTAMP NOT NULL,        -- arrived_at + 21 days
    picked_up_at TIMESTAMP,
    returned_at TIMESTAMP,

    -- Статус
    status VARCHAR(20) NOT NULL DEFAULT 'awaiting_pickup'
        CHECK (status IN ('awaiting_pickup', 'picked_up', 'returned', 'overdue', 'expired')),

    -- Хранение
    storage_fee DECIMAL(10,2) DEFAULT 0.00,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_parcels_pvz ON parcels_at_pvz(pvz_id);
CREATE INDEX idx_parcels_customer ON parcels_at_pvz(customer_id);
CREATE INDEX idx_parcels_status ON parcels_at_pvz(status);
CREATE INDEX idx_parcels_pickup_code ON parcels_at_pvz(pickup_code);
CREATE INDEX idx_parcels_arrived ON parcels_at_pvz(arrived_at);
```

### partner_accounts (Партнёры)

```sql
CREATE TABLE partner_accounts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,                    -- FK to auth_db.users

    -- Бизнес информация
    business_name VARCHAR(255) NOT NULL,
    business_type VARCHAR(20) NOT NULL CHECK (business_type IN
        ('cafe', 'shop', 'salon', 'pharmacy', 'gas_station', 'post_office')),

    -- Контакты
    contact_name VARCHAR(255) NOT NULL,
    phone VARCHAR(30) NOT NULL,
    email VARCHAR(255) NOT NULL,

    -- Налоги и финансы
    tax_id VARCHAR(50),
    bank_account VARCHAR(50),

    -- Статус
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'active', 'suspended', 'rejected')),

    -- Заработки
    total_earnings DECIMAL(12,2) DEFAULT 0.00,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE UNIQUE INDEX idx_partner_user ON partner_accounts(user_id);
CREATE INDEX idx_partner_status ON partner_accounts(status);
CREATE INDEX idx_partner_business_type ON partner_accounts(business_type);
```

## Миграции

| Миграция | Описание | Файл |
|----------|----------|------|
| 001 | Initial Schema - Основные таблицы (pvz, operators, parcels) | 001_initial_schema.up.sql |
| 002 | Seed Data - Тестовые ПВЗ и партнёры | 002_seed_data.up.sql |
| 003 | Partner Payouts & Commissions - Выплаты партнёрам | 003_partner_payouts_commissions.up.sql |
| 004 | Returns - Система возвратов | 004_returns.up.sql |
| 005 | Analytics Views - Аналитические представления | 005_analytics_views.up.sql |

## Триггеры

### 1. Capacity Tracking

```sql
-- Обновление current_parcels при добавлении/удалении посылки
CREATE OR REPLACE FUNCTION update_pvz_capacity()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE pvz SET current_parcels = current_parcels + 1 WHERE id = NEW.pvz_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE pvz SET current_parcels = current_parcels - 1 WHERE id = OLD.pvz_id;
    ELSIF TG_OP = 'UPDATE' AND OLD.status = 'awaiting_pickup' AND NEW.status IN ('picked_up', 'returned') THEN
        UPDATE pvz SET current_parcels = current_parcels - 1 WHERE id = NEW.pvz_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_capacity
AFTER INSERT OR UPDATE OR DELETE ON parcels_at_pvz
FOR EACH ROW EXECUTE FUNCTION update_pvz_capacity();
```

### 2. Overdue Detection

```sql
-- Автоматическая пометка просроченных посылок
CREATE OR REPLACE FUNCTION mark_overdue_parcels()
RETURNS void AS $$
BEGIN
    UPDATE parcels_at_pvz
    SET status = 'overdue'
    WHERE status = 'awaiting_pickup'
      AND arrived_at < NOW() - INTERVAL '3 days'
      AND arrived_at >= NOW() - INTERVAL '21 days';

    UPDATE parcels_at_pvz
    SET status = 'expired'
    WHERE status IN ('awaiting_pickup', 'overdue')
      AND arrived_at < NOW() - INTERVAL '21 days';
END;
$$ LANGUAGE plpgsql;
```

### 3. Storage Fee Calculation

```sql
-- Расчёт платы за хранение
CREATE OR REPLACE FUNCTION calculate_storage_fee(parcel_id BIGINT)
RETURNS DECIMAL AS $$
DECLARE
    days_stored INTEGER;
    fee DECIMAL := 0;
BEGIN
    SELECT EXTRACT(DAY FROM (NOW() - arrived_at))::INTEGER
    INTO days_stored
    FROM parcels_at_pvz
    WHERE id = parcel_id;

    IF days_stored > 3 THEN
        fee := (days_stored - 3) * 20.00;  -- 20 RSD за день
    END IF;

    RETURN fee;
END;
$$ LANGUAGE plpgsql;
```

## Views

### v_pvz_capacity

```sql
CREATE VIEW v_pvz_capacity AS
SELECT
    p.id,
    p.code,
    p.name,
    p.type,
    p.status,
    p.max_capacity,
    p.current_parcels,
    p.reserved_slots,
    ROUND((p.current_parcels::DECIMAL / p.max_capacity) * 100, 2) AS utilization_pct,
    CASE
        WHEN p.current_parcels >= p.max_capacity THEN 'critical'
        WHEN p.current_parcels >= p.max_capacity * 0.9 THEN 'warning'
        ELSE 'normal'
    END AS capacity_status
FROM pvz p;
```

### v_partner_earnings

```sql
CREATE VIEW v_partner_earnings AS
SELECT
    pa.id AS partner_id,
    pa.business_name,
    pa.business_type,
    COUNT(pc.id) AS total_commissions,
    SUM(pc.commission_amount) AS total_earned,
    SUM(CASE WHEN pc.status = 'paid' THEN pc.commission_amount ELSE 0 END) AS total_paid,
    SUM(CASE WHEN pc.status = 'pending' THEN pc.commission_amount ELSE 0 END) AS total_pending
FROM partner_accounts pa
LEFT JOIN partner_commissions pc ON pa.id = pc.partner_id
GROUP BY pa.id, pa.business_name, pa.business_type;
```

### v_operator_stats

```sql
CREATE VIEW v_operator_stats AS
SELECT
    po.id AS operator_id,
    po.user_id,
    po.pvz_id,
    po.role,
    po.total_handovers,
    po.total_pickups,
    po.total_returns,
    COUNT(DISTINCT os.id) AS total_shifts,
    SUM(os.parcels_accepted) AS total_accepted,
    SUM(os.parcels_issued) AS total_issued,
    SUM(os.cash_collected) AS total_cash
FROM pvz_operators po
LEFT JOIN operator_shifts os ON po.id = os.operator_id
GROUP BY po.id;
```

## Индексы производительности

```sql
-- Геопоиск (требует PostGIS extension)
CREATE EXTENSION IF NOT EXISTS cube;
CREATE EXTENSION IF NOT EXISTS earthdistance;

-- Полнотекстовый поиск по ПВЗ
CREATE INDEX idx_pvz_fulltext ON pvz USING GIN(to_tsvector('simple', name || ' ' || address));

-- Composite индексы для частых запросов
CREATE INDEX idx_parcels_pvz_status ON parcels_at_pvz(pvz_id, status);
CREATE INDEX idx_parcels_customer_status ON parcels_at_pvz(customer_id, status);
CREATE INDEX idx_commissions_partner_status ON partner_commissions(partner_id, status);
```

## Бизнес-правила

### Комиссии партнёров

| Тип бизнеса | Комиссия партнёру | Комиссия Vondi |
|-------------|-------------------|----------------|
| Кафе | 70% | 30% |
| Магазин | 65% | 35% |
| Салон | 70% | 30% |
| Аптека | 65% | 35% |
| АЗС | 65% | 35% |
| Почта | 75% | 25% |

### Хранение

- **Бесплатно:** 3 дня
- **Платно:** 20 RSD/день (дни 4-21)
- **Максимум:** 21 день (после этого статус = expired)

### Pickup Code

- **Формат:** 6 цифр (123456)
- **TTL:** 48 часов
- **Попытки:** Максимум 5
- **Регенерация:** Автоматически при истечении или исчерпании попыток

## Связи с другими БД

```sql
-- pvz_operators.user_id → auth_db.users.id
-- partner_accounts.user_id → auth_db.users.id
-- parcels_at_pvz.order_id → vondi_db.orders.id (Monolith)
-- parcels_at_pvz.customer_id → auth_db.users.id
-- partner_commissions.order_id → vondi_db.orders.id
```

## Health Check Queries

```sql
-- Проверка доступности БД
SELECT 1;

-- Статистика ПВЗ
SELECT COUNT(*) AS total_pvz,
       COUNT(*) FILTER (WHERE type = 'full') AS full_pvz,
       COUNT(*) FILTER (WHERE type = 'mikro') AS mikro_pvz,
       COUNT(*) FILTER (WHERE status = 'active') AS active_pvz
FROM pvz;

-- Посылки в ПВЗ
SELECT status, COUNT(*)
FROM parcels_at_pvz
GROUP BY status;

-- Перегруженные ПВЗ
SELECT code, name, current_parcels, max_capacity
FROM pvz
WHERE current_parcels >= max_capacity * 0.9
ORDER BY current_parcels DESC;
```

---

**Статус:** ✅ Ready for Development
**Версия БД:** 1.0.0
**Последнее обновление:** 2025-12-21
