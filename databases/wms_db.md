# Паспорт: База данных wms_dev_db

> Обновлено: 2025-12-21

## Идентификация

| Параметр | Значение |
|----------|----------|
| Тип | PostgreSQL 15 Alpine |
| Image | postgres:15-alpine |
| Хост | localhost |
| Порт | 35435 |
| Имя БД | wms_dev_db |
| Пользователь | wms_user |
| Пароль | wms_secret |
| Контейнер | wms_postgres |

## КРИТИЧНО

**Это ОТДЕЛЬНАЯ база данных WMS микросервиса!**
- НЕ используй порт 5433 (это монолит vondi_db)
- НЕ используй порт 35434 (это Listings микросервис)
- WMS имеет СВОЮ независимую БД для управления складами
- **Расширения:** pgcrypto (для UUID generation)

## Подключение

```bash
# Через localhost
psql "postgres://wms_user:wms_secret@localhost:35435/wms_dev_db?sslmode=disable"

# Через Docker
docker exec -it wms_postgres psql -U wms_user -d wms_dev_db
```

## Статистика

| Параметр | Значение |
|----------|----------|
| Всего таблиц | 32+ |
| Бизнес-таблиц | 31 |
| Миграций | 25 |
| Путь к миграциям | /p/github.com/vondi-global/warehouse/migrations/ |

## Таблицы (полный список - 32 таблицы)

### 1. Складская Инфраструктура (Warehouse Infrastructure)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| warehouses | Склады (main, regional, partner, dark_store) | BIGSERIAL |
| warehouse_zones | Зоны склада (receiving, storage, picking, packing, shipping, returns) | BIGSERIAL |
| storage_locations | Ячейки хранения (полки, паллеты, ячейки) | BIGSERIAL |

**Особенности warehouses:**
- Типы: main (основной), regional (региональный), partner (партнёрский), dark_store (тёмный склад)
- Геолокация: `latitude`, `longitude` для построения карт
- Владение: `owner_user_id` (franchise owner), `manager_user_id` (менеджер)
- Cross-DB референсы: user_id из монолита (БЕЗ FK constraint - разные базы)
- Часы работы: `working_hours` JSONB
- Уникальность: `code` (VARCHAR 50) UNIQUE

**Особенности warehouse_zones:**
- Типы зон: receiving (приёмка), storage (хранение), picking (комплектация), packing (упаковка), shipping (отгрузка), returns (возвраты)
- Температурные режимы: ambient (комнатная), cold (холодная), frozen (морозильная)
- Иерархия: `warehouse_id` → zones → locations
- Unique: `(warehouse_id, code)`

**Особенности storage_locations:**
- Адресация: `aisle` (проход), `rack` (стеллаж), `shelf` (полка), `bin` (ячейка)
- Типы: shelf (полка), pallet (паллета), floor (пол), bin (ячейка)
- Статусы: available (доступна), occupied (занята), reserved (зарезервирована), maintenance (на обслуживании)
- Физические размеры: `width`, `height`, `depth`, `max_weight`
- Unique: `(warehouse_id, code)`

### 2. Управление Инвентарём (Inventory Management)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| product_batches | Партии товаров (batch tracking) | BIGSERIAL |
| product_locations | Размещение товаров на локациях | BIGSERIAL |
| wms_reservations | Резервации товаров (для заказов) | BIGSERIAL |
| inventory_movements | История движений инвентаря | BIGSERIAL |

**Особенности product_locations:**
- Связь: `listing_id` (товар из Listings MS), `location_id`, `warehouse_id`, `batch_id`
- Количество: `quantity`, `reserved_quantity`, `min_quantity`, `max_quantity`
- Constraint: `reserved_quantity <= quantity` (логическая целостность)
- Unique: `(location_id, listing_id, batch_id)` - один товар одной партии на одной локации
- Автоматика: при резервировании обновляется `reserved_quantity`

**Особенности wms_reservations:**
- Референс: `order_id` (заказ из монолита/Listings)
- Связь: `product_location_id` (конкретная локация с товаром)
- TTL: `expires_at` (автоочистка истекших резерваций)
- Статусы: reserved (зарезервировано), released (освобождено), expired (истекло)

**Особенности inventory_movements:**
- Типы движений: receive (приёмка), pick (комплектация), pack (упаковка), ship (отгрузка), return (возврат), adjust (корректировка), transfer (перемещение)
- Источник/назначение: `from_location_id`, `to_location_id` (NULL для in/out)
- Event sourcing: НИКОГДА не удаляются, только INSERT (аудит лог)
- Трекинг: `performed_by` (worker_id), `reason`, `reference_id`, `reference_type`

### 3. Комплектация (Picking Operations)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| picking_tasks | Задачи на комплектацию заказов | BIGSERIAL |
| picking_task_items | Товары для комплектации | BIGSERIAL |
| batch_picking_tasks | Пакетная комплектация (multi-order) | BIGSERIAL |
| batch_picking_items | Товары пакетной комплектации | BIGSERIAL |

**Особенности picking_tasks:**
- Связь: `order_id`, `warehouse_id`, `worker_id` (NULL до назначения)
- Статусы: pending (ожидает), assigned (назначена), in_progress (в процессе), completed (завершена), cancelled (отменена)
- Приоритеты: low (низкий), normal (обычный), high (высокий), urgent (срочный)
- Прогресс: `total_items`, `picked_items` (авто-обновляется из items)

**Особенности batch_picking_tasks:**
- Массовая комплектация: `order_ids BIGINT[]` (PostgreSQL array) - несколько заказов в одном батче
- Статусы: created (создан), in_progress (в процессе), sorting (сортировка), completed (завершён), cancelled (отменён)
- Оптимизация маршрута: items имеют `route_sequence` для эффективного пути
- Группировка: товары группируются по SKU и локации (один проход - все заказы)

**Особенности batch_picking_items:**
- Агрегация: `total_quantity` (сумма из всех заказов), `picked_quantity`
- Какие заказы: `orders_needing_item BIGINT[]` - какие заказы нуждаются в этом товаре
- Маршрут: `route_sequence` (1, 2, 3, ...) - оптимальный порядок обхода склада
- Статусы: pending (ожидает), picked (собран), short (недостача), skipped (пропущен)

### 4. Упаковка (Packing Workflow)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| packing_tasks | Задачи на упаковку | BIGSERIAL |
| packages | Созданные посылки | BIGSERIAL |

**Особенности packing_tasks:**
- Связь: `order_id`, `picking_task_id` (опционально), `warehouse_id`, `worker_id`
- Статусы: pending, in_progress, packed (упакован), shipped (отправлен), cancelled
- Упаковка: `packaging_type` (box, bag, envelope, pallet, custom)
- Размеры: `weight`, `length`, `width`, `height` (для расчёта доставки)
- Доставка: `tracking_number`, `label_url` (URL этикетки в MinIO/S3)

**Особенности packages:**
- Связь: `packing_task_id`
- Вес и габариты: `weight`, `length`, `width`, `height`
- Shipping: `tracking_number` (UNIQUE), `carrier` (Post Express, DHL, etc)
- Контент: `contents` JSONB (детали упакованных товаров)

### 5. Работники (Workers Management)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| workers | Складские работники (пикеры, пакеры) | BIGSERIAL |
| warehouse_staff | Персонал склада (owners, managers) | BIGSERIAL |
| warehouse_invitations | Приглашения в команду склада | BIGSERIAL |

**Особенности workers:**
- Роли: picker (комплектовщик), packer (упаковщик), supervisor (супервайзер), manager (менеджер)
- Аутентификация: `pin_hash` (bcrypt) для входа на мобильном устройстве WMS
- Уникальный код: `code` (WRK-001, WRK-002, ...) UNIQUE
- Статусы: active (активен), on_break (на перерыве), offline (не в сети)
- Связь: `warehouse_id` (работник привязан к одному складу)
- Тестовые данные: WRK-001 (Marko Petrović), WRK-002 (Ana Jovanović), PIN всех: 1234

**Особенности warehouse_staff:**
- Роли: owner (владелец), manager (менеджер), supervisor (супервайзер), worker (работник) - ENUM type
- Связь с монолитом: `user_id` (БЕЗ FK constraint - cross-database)
- Кеш данных: `user_email`, `user_name` (синхронизация через gRPC из Auth MS)
- Unique: `(warehouse_id, user_id)` - один пользователь = одна роль на один склад
- M:N relationship: пользователь может работать на нескольких складах с разными ролями

**Особенности warehouse_invitations:**
- Типы: email (персональное приглашение), link (многоразовая ссылка)
- Инвайт-код: `invite_code` (формат: wh-a7b3c9f2, VARCHAR 32) для link-приглашений
- Лимиты: `max_uses` (NULL = бесконечно), `current_uses`, `expires_at`
- Статусы: pending (ожидает), accepted (принято), declined (отклонено), expired (истекло), revoked (отозвано)
- Кто пригласил: `invited_by_id` (user из монолита, без FK)
- История: `warehouse_staff.invitation_id` - откуда пришёл сотрудник (для аудита)

### 6. Контроль Качества (Quality Control)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| qc_tasks | Задачи QC инспекции | BIGSERIAL |
| qc_checklist_items | Чек-листы проверки | BIGSERIAL |
| qc_photos | Фото дефектов/повреждений | BIGSERIAL |

**Особенности qc_tasks:**
- Типы: RECEIVING (приёмка товара), PRE_SHIPMENT (перед отправкой), RETURN_INSPECTION (инспекция возврата)
- Связь: `reference_id` (ID связанной задачи - picking/packing/return_task)
- Статусы: PENDING (ожидает), IN_PROGRESS (в процессе), COMPLETED (завершена), CANCELLED (отменена)
- Результаты: PENDING, PASS (прошёл), FAIL (не прошёл), CONDITIONAL_PASS (условный проход)
- Инспектор: `inspector_id` (FK к workers)

**Особенности qc_checklist_items:**
- Связь: `qc_task_id` (CASCADE delete)
- Автоматика: триггер заполняет `checked_at` при `is_checked = TRUE`
- Откат: если `is_checked = FALSE`, `checked_at` сбрасывается в NULL
- Заметки: `notes` для каждого пункта чек-листа отдельно

**Особенности qc_photos:**
- Хранение: `photo_url` (MinIO/S3 bucket s3.vondi.rs)
- Описание: `description` (например, "Повреждение упаковки слева")
- Привязка: `qc_task_id` (CASCADE delete - удаление QC задачи удаляет фото)

### 7. Возвраты (Returns Management)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| return_tasks | Задачи на обработку возвратов | BIGSERIAL |
| return_items | Товары в возврате | BIGSERIAL |
| return_item_photos | Фото возвращаемых товаров | BIGSERIAL |

**Особенности return_tasks:**
- Связь: `order_id`, `customer_id`, `qc_task_id` (автоматически создаётся QC), `inspector_id`
- Статусы: CREATED (создан), INSPECTING (инспектируется), COMPLETED (завершён), CANCELLED (отменён)
- Причины: WRONG_ITEM (не тот товар), DAMAGED (повреждён), CHANGED_MIND (передумал), DEFECTIVE (брак), OTHER (другое)
- Workflow: создание return_task → автоматически QC task → инспекция → решение

**Особенности return_items:**
- Связь: `return_task_id` (CASCADE delete), `sku`
- Состояние после инспекции: NEW (новый), LIKE_NEW (как новый), GOOD (хорошее), DAMAGED (повреждён), UNSELLABLE (непродаваемый)
- Решение инспектора: RESTOCK (вернуть на склад), REFURBISH (восстановить), DISPOSE (утилизировать), RETURN_TO_SUPPLIER (вернуть поставщику)
- Фото: `photo_url` (для документации, один URL)
- Multiple photos: через `return_item_photos` таблицу

**Особенности return_item_photos:**
- Связь: `return_item_id` (CASCADE delete)
- Множественные фото: один return_item может иметь несколько фото
- S3/MinIO: `photo_url` (полный URL с bucket)
- Описание: `description` (опционально)

### 8. Инвентаризация (Cycle Counts)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| cycle_counts | Задачи инвентаризации | BIGSERIAL |
| cycle_count_items | Товары для подсчёта | BIGSERIAL |

**Особенности cycle_counts:**
- Связь: `warehouse_id`, `zone_id` (опционально - можно весь склад или только зону), `worker_id`
- Статусы: pending (ожидает), in_progress (в процессе), completed (завершена), cancelled (отменена)
- Метрики: `total_items`, `counted_items`, `discrepancies` (автообновление через триггеры)
- Автопрогресс: при подсчёте items обновляются счётчики в родительской задаче

**Особенности cycle_count_items:**
- Связь: `cycle_count_id` (CASCADE delete), `location_id`, `product_id`, `sku`
- Подсчёт: `expected_qty` (из product_locations), `actual_qty` (фактический), `variance` (расхождение)
- Автоматика: триггер `calculate_cycle_count_variance` вычисляет `variance = actual_qty - expected_qty`
- Триггер прогресса: `update_cycle_count_progress` обновляет `counted_items`, `discrepancies` в cycle_counts
- Кто считал: `counted_by_id`, `counted_at`

### 9. Дополнительные Операции (Stock Operations)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| stock_adjustments | Ручные корректировки стока | BIGSERIAL |
| location_transfers | Перемещения между локациями | BIGSERIAL |
| substitutions | Замены товаров при недостаче | BIGSERIAL |
| substitution_options | Варианты замены | BIGSERIAL |

**Особенности stock_adjustments:**
- Связь: `product_location_id`
- Типы: damage (повреждение), loss (потеря), found (найдено), correction (исправление)
- Изменение: `quantity_change` (может быть < 0 для списания)
- Аудит: `reason` (обязательное объяснение), `performed_by` (worker_id)
- История: записи НЕ удаляются (audit log)

**Особенности location_transfers:**
- Перемещение: `from_location_id` → `to_location_id`
- Связь: `listing_id`, `batch_id` (опционально), `quantity`
- Статусы: pending, in_progress, completed, cancelled
- Worker: `performed_by`, timestamps: `started_at`, `completed_at`
- Автоматика: при completed обновляются product_locations (from -=, to +=)

**Особенности substitutions:**
- Связь: `picking_task_id`, `original_listing_id` (товар которого нет в наличии)
- Статусы: pending (предложено), approved (одобрено покупателем), rejected (отклонено)
- Workflow: WMS обнаружил out-of-stock → создал substitution с options → покупатель выбрал

**Особенности substitution_options:**
- Связь: `substitution_id`, `substitute_listing_id` (товар-заменитель)
- Доступность: `quantity` (сколько есть в наличии заменителя)
- Выбор: `is_selected` (TRUE если покупатель выбрал эту замену)
- Один substitution может иметь несколько options (WMS предлагает несколько вариантов)

### 10. Геймификация (Gamification)

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| gamification_metrics | Метрики работников (баллы, уровни) | BIGSERIAL |

**Особенности gamification_metrics:**
- Связь: `worker_id` (ONE-TO-ONE с workers)
- KPI: `tasks_completed`, `accuracy_rate`, `speed_score`, `error_count`
- Уровни: автоматический расчёт на основе баллов
- Бейджи: `badges` JSONB (массив достижений: ["speed_demon", "accuracy_king", ...])
- Рейтинг: индексы для быстрого построения leaderboards

### 11. Системные Таблицы

| Таблица | Назначение | Тип ID |
|---------|------------|--------|
| schema_migrations | Версии миграций (goose) | - |

## Схемы ключевых таблиц

### warehouses (Склады)

```sql
CREATE TABLE warehouses (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(20) NOT NULL DEFAULT 'main',
    address TEXT NOT NULL,
    city VARCHAR(100) NOT NULL,
    country VARCHAR(2) NOT NULL DEFAULT 'RS',
    latitude NUMERIC(10, 8),
    longitude NUMERIC(11, 8),
    total_area NUMERIC(10, 2),
    contact_name VARCHAR(255),
    contact_phone VARCHAR(50),
    contact_email VARCHAR(255),
    working_hours JSONB DEFAULT '{}',

    -- Ownership (references users in monolith - no FK constraint)
    owner_user_id BIGINT,
    manager_user_id BIGINT,

    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_warehouse_type CHECK (type IN ('main', 'regional', 'partner', 'dark_store'))
);

-- Indexes
CREATE INDEX idx_warehouses_type ON warehouses(type);
CREATE INDEX idx_warehouses_city ON warehouses(city);
CREATE INDEX idx_warehouses_active ON warehouses(is_active) WHERE is_active = true;
CREATE INDEX idx_warehouses_owner_user_id ON warehouses(owner_user_id) WHERE owner_user_id IS NOT NULL;
CREATE INDEX idx_warehouses_manager_user_id ON warehouses(manager_user_id) WHERE manager_user_id IS NOT NULL;
```

### product_locations (Размещение товаров)

```sql
CREATE TABLE product_locations (
    id BIGSERIAL PRIMARY KEY,
    listing_id BIGINT NOT NULL,  -- Reference to Listings service
    location_id BIGINT NOT NULL REFERENCES storage_locations(id) ON DELETE RESTRICT,
    warehouse_id BIGINT NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    batch_id BIGINT REFERENCES product_batches(id) ON DELETE SET NULL,
    quantity INT NOT NULL DEFAULT 0,
    reserved_quantity INT NOT NULL DEFAULT 0,
    min_quantity INT DEFAULT 0,
    max_quantity INT,
    received_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_counted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_product_locations_location_listing_batch UNIQUE(location_id, listing_id, batch_id),
    CONSTRAINT chk_quantity_non_negative CHECK (quantity >= 0),
    CONSTRAINT chk_reserved_quantity_non_negative CHECK (reserved_quantity >= 0),
    CONSTRAINT chk_reserved_lte_quantity CHECK (reserved_quantity <= quantity),
    CONSTRAINT chk_min_quantity_non_negative CHECK (min_quantity >= 0),
    CONSTRAINT chk_max_quantity_gte_min CHECK (max_quantity IS NULL OR max_quantity >= min_quantity)
);

-- Indexes
CREATE INDEX idx_product_locations_listing_id ON product_locations(listing_id);
CREATE INDEX idx_product_locations_warehouse_id ON product_locations(warehouse_id);
CREATE INDEX idx_product_locations_location_id ON product_locations(location_id);
CREATE INDEX idx_product_locations_batch_id ON product_locations(batch_id) WHERE batch_id IS NOT NULL;
CREATE INDEX idx_product_locations_quantity ON product_locations(quantity) WHERE quantity > 0;
```

### picking_tasks (Комплектация)

```sql
CREATE TABLE picking_tasks (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    warehouse_id BIGINT NOT NULL REFERENCES warehouses(id),
    worker_id BIGINT,  -- NULL until assigned
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
    total_items INT NOT NULL DEFAULT 0,
    picked_items INT NOT NULL DEFAULT 0,
    notes TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ,
    cancellation_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_status CHECK (status IN ('pending', 'assigned', 'in_progress', 'completed', 'cancelled')),
    CONSTRAINT chk_priority CHECK (priority IN ('low', 'normal', 'high', 'urgent'))
);

-- Indexes
CREATE INDEX idx_picking_tasks_order_id ON picking_tasks(order_id);
CREATE INDEX idx_picking_tasks_warehouse_id ON picking_tasks(warehouse_id);
CREATE INDEX idx_picking_tasks_worker_id ON picking_tasks(worker_id);
CREATE INDEX idx_picking_tasks_status ON picking_tasks(status);
CREATE INDEX idx_picking_tasks_priority_status ON picking_tasks(priority, status);
```

### batch_picking_tasks (Пакетная комплектация)

```sql
CREATE TABLE batch_picking_tasks (
    id BIGSERIAL PRIMARY KEY,
    warehouse_id BIGINT NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    worker_id BIGINT REFERENCES workers(id) ON DELETE SET NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'created',
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
    order_ids BIGINT[] NOT NULL,  -- PostgreSQL array of order IDs
    total_items INTEGER NOT NULL DEFAULT 0,
    picked_items INTEGER NOT NULL DEFAULT 0,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT batch_picking_tasks_status_check
        CHECK (status IN ('created', 'in_progress', 'sorting', 'completed', 'cancelled')),
    CONSTRAINT batch_picking_tasks_priority_check
        CHECK (priority IN ('low', 'normal', 'high', 'urgent')),
    CONSTRAINT batch_picking_tasks_items_check
        CHECK (total_items >= 0 AND picked_items >= 0 AND picked_items <= total_items)
);

-- Indexes
CREATE INDEX idx_batch_picking_tasks_warehouse_id ON batch_picking_tasks(warehouse_id);
CREATE INDEX idx_batch_picking_tasks_worker_id ON batch_picking_tasks(worker_id);
CREATE INDEX idx_batch_picking_tasks_status ON batch_picking_tasks(status);
CREATE INDEX idx_batch_picking_tasks_order_ids ON batch_picking_tasks USING GIN(order_ids);
```

### workers (Складские работники)

```sql
CREATE TABLE workers (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(20) UNIQUE NOT NULL,  -- WRK-001, WRK-002, etc.
    name VARCHAR(100) NOT NULL,
    pin_hash VARCHAR(255) NOT NULL,    -- bcrypt hashed PIN for authentication
    warehouse_id BIGINT NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL DEFAULT 'picker',
    status VARCHAR(20) NOT NULL DEFAULT 'offline',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT workers_role_check CHECK (role IN ('picker', 'packer', 'supervisor', 'manager')),
    CONSTRAINT workers_status_check CHECK (status IN ('active', 'on_break', 'offline'))
);

-- Indexes
CREATE INDEX idx_workers_warehouse ON workers(warehouse_id);
CREATE INDEX idx_workers_status ON workers(status);
CREATE INDEX idx_workers_role ON workers(role);
CREATE INDEX idx_workers_code ON workers(code);
```

### warehouse_staff (Персонал)

```sql
CREATE TYPE warehouse_staff_role AS ENUM ('owner', 'manager', 'supervisor', 'worker');

CREATE TABLE warehouse_staff (
    id BIGSERIAL PRIMARY KEY,
    warehouse_id BIGINT NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL,  -- References users.id in monolith DB (no FK constraint)
    user_email VARCHAR(255),  -- Cached for display
    user_name VARCHAR(255),   -- Cached for display
    role warehouse_staff_role NOT NULL DEFAULT 'worker',
    is_active BOOLEAN NOT NULL DEFAULT true,
    invitation_id BIGINT REFERENCES warehouse_invitations(id) ON DELETE SET NULL,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_warehouse_staff_user_warehouse UNIQUE(warehouse_id, user_id)
);

-- Indexes
CREATE INDEX idx_warehouse_staff_warehouse_id ON warehouse_staff(warehouse_id) WHERE is_active = true;
CREATE INDEX idx_warehouse_staff_user_id ON warehouse_staff(user_id) WHERE is_active = true;
CREATE INDEX idx_warehouse_staff_role ON warehouse_staff(role) WHERE is_active = true;
CREATE INDEX idx_warehouse_staff_invitation_id ON warehouse_staff(invitation_id) WHERE invitation_id IS NOT NULL;
```

### qc_tasks (Контроль качества)

```sql
CREATE TABLE qc_tasks (
    id BIGSERIAL PRIMARY KEY,
    type VARCHAR(50) NOT NULL,
    reference_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    result VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    inspector_id BIGINT REFERENCES workers(id) ON DELETE SET NULL,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMPTZ,

    CONSTRAINT chk_qc_type CHECK (type IN ('RECEIVING', 'PRE_SHIPMENT', 'RETURN_INSPECTION')),
    CONSTRAINT chk_qc_status CHECK (status IN ('PENDING', 'IN_PROGRESS', 'COMPLETED', 'CANCELLED')),
    CONSTRAINT chk_qc_result CHECK (result IN ('PENDING', 'PASS', 'FAIL', 'CONDITIONAL_PASS'))
);

-- Indexes
CREATE INDEX idx_qc_tasks_type ON qc_tasks(type);
CREATE INDEX idx_qc_tasks_status ON qc_tasks(status);
CREATE INDEX idx_qc_tasks_reference_id ON qc_tasks(reference_id);
CREATE INDEX idx_qc_tasks_inspector_id ON qc_tasks(inspector_id);
```

## Полезные запросы

```sql
-- Список складов с владельцами
SELECT
    w.id, w.code, w.name, w.type, w.city,
    w.owner_user_id, w.manager_user_id,
    w.is_active
FROM warehouses w
ORDER BY w.is_active DESC, w.city;

-- Персонал склада (с ролями)
SELECT
    ws.id, w.name as warehouse_name,
    ws.user_id, ws.user_email, ws.user_name,
    ws.role, ws.is_active
FROM warehouse_staff ws
JOIN warehouses w ON w.id = ws.warehouse_id
WHERE ws.is_active = true
ORDER BY w.name, ws.role;

-- Активные приглашения (не истекшие)
SELECT
    wi.id, w.name as warehouse_name,
    wi.type, wi.role,
    wi.invited_email, wi.invite_code,
    wi.status, wi.expires_at,
    wi.max_uses, wi.current_uses
FROM warehouse_invitations wi
JOIN warehouses w ON w.id = wi.warehouse_id
WHERE wi.status = 'pending'
  AND (wi.expires_at IS NULL OR wi.expires_at > NOW())
ORDER BY wi.created_at DESC;

-- Инвентарь на складе (группировка по товарам)
SELECT
    pl.listing_id,
    pl.warehouse_id,
    SUM(pl.quantity) as total_quantity,
    SUM(pl.reserved_quantity) as total_reserved,
    SUM(pl.quantity - pl.reserved_quantity) as available,
    COUNT(DISTINCT pl.location_id) as locations_count
FROM product_locations pl
WHERE pl.warehouse_id = 1
GROUP BY pl.listing_id, pl.warehouse_id
HAVING SUM(pl.quantity) > 0
ORDER BY total_quantity DESC;

-- Товары с низким стоком (ниже min_quantity)
SELECT
    pl.id, pl.listing_id, sl.code as location_code,
    pl.quantity, pl.min_quantity,
    (pl.min_quantity - pl.quantity) as shortage
FROM product_locations pl
JOIN storage_locations sl ON sl.id = pl.location_id
WHERE pl.quantity < pl.min_quantity
  AND pl.warehouse_id = 1
ORDER BY shortage DESC;

-- Активные задачи комплектации (pending, assigned, in_progress)
SELECT
    pt.id, pt.order_id, w.name as warehouse_name,
    wk.name as worker_name, pt.status, pt.priority,
    pt.total_items, pt.picked_items,
    pt.created_at
FROM picking_tasks pt
JOIN warehouses w ON w.id = pt.warehouse_id
LEFT JOIN workers wk ON wk.id = pt.worker_id
WHERE pt.status IN ('pending', 'assigned', 'in_progress')
ORDER BY pt.priority DESC, pt.created_at;

-- Статистика работников (за сегодня)
SELECT
    w.code, w.name, w.role, w.status,
    COUNT(pt.id) as tasks_today,
    SUM(pt.picked_items) as items_picked_today
FROM workers w
LEFT JOIN picking_tasks pt ON pt.worker_id = w.id
    AND pt.created_at >= CURRENT_DATE
WHERE w.warehouse_id = 1
GROUP BY w.id, w.code, w.name, w.role, w.status
ORDER BY tasks_today DESC;

-- Незавершённые возвраты
SELECT
    rt.id, rt.order_id, rt.customer_id,
    rt.status, rt.reason,
    COUNT(ri.id) as items_count,
    rt.created_at
FROM return_tasks rt
LEFT JOIN return_items ri ON ri.return_task_id = rt.id
WHERE rt.status IN ('CREATED', 'INSPECTING')
GROUP BY rt.id
ORDER BY rt.created_at;

-- QC задачи с результатами FAIL
SELECT
    qc.id, qc.type, qc.reference_id,
    qc.status, qc.result,
    wk.name as inspector_name,
    qc.notes,
    COUNT(qp.id) as photos_count
FROM qc_tasks qc
LEFT JOIN workers wk ON wk.id = qc.inspector_id
LEFT JOIN qc_photos qp ON qp.qc_task_id = qc.id
WHERE qc.result = 'FAIL'
GROUP BY qc.id, wk.name
ORDER BY qc.created_at DESC;

-- Инвентаризация: текущие расхождения
SELECT
    cc.id, w.name as warehouse_name,
    cc.total_items, cc.counted_items, cc.discrepancies,
    ROUND(cc.discrepancies::numeric / NULLIF(cc.counted_items, 0) * 100, 2) as discrepancy_rate,
    cc.status, cc.created_at
FROM cycle_counts cc
JOIN warehouses w ON w.id = cc.warehouse_id
WHERE cc.discrepancies > 0
ORDER BY discrepancy_rate DESC;

-- Локации с максимальной загрузкой
SELECT
    sl.code, wz.name as zone_name,
    COUNT(pl.id) as products_count,
    SUM(pl.quantity) as total_quantity,
    sl.status
FROM storage_locations sl
JOIN warehouse_zones wz ON wz.id = sl.zone_id
LEFT JOIN product_locations pl ON pl.location_id = sl.id
WHERE sl.warehouse_id = 1
GROUP BY sl.id, sl.code, wz.name, sl.status
HAVING COUNT(pl.id) > 0
ORDER BY total_quantity DESC
LIMIT 20;

-- Batch picking эффективность
SELECT
    bpt.id,
    array_length(bpt.order_ids, 1) as orders_in_batch,
    bpt.total_items,
    bpt.picked_items,
    ROUND(bpt.picked_items::numeric / NULLIF(bpt.total_items, 0) * 100, 2) as completion_rate,
    bpt.status,
    bpt.started_at,
    bpt.completed_at,
    EXTRACT(EPOCH FROM (COALESCE(bpt.completed_at, NOW()) - bpt.started_at)) / 60 as duration_minutes
FROM batch_picking_tasks bpt
WHERE bpt.warehouse_id = 1
ORDER BY bpt.created_at DESC
LIMIT 10;

-- История движений товара
SELECT
    im.movement_type,
    sl_from.code as from_location,
    sl_to.code as to_location,
    im.quantity,
    im.reason,
    w.name as performed_by,
    im.created_at
FROM inventory_movements im
LEFT JOIN storage_locations sl_from ON im.from_location_id = sl_from.id
LEFT JOIN storage_locations sl_to ON im.to_location_id = sl_to.id
LEFT JOIN workers w ON im.performed_by = w.id
WHERE im.listing_id = 12345
ORDER BY im.created_at DESC
LIMIT 50;
```

## Связь с другими сервисами

### Интеграция с монолитом (vondi)
- `warehouses.owner_user_id`, `warehouses.manager_user_id` - FK к users (без constraint - cross-DB)
- `warehouse_staff.user_id` - FK к users (без constraint)
- `warehouse_invitations.invited_by_id`, `invited_user_id` - FK к users (без constraint)
- `picking_tasks.order_id`, `packing_tasks.order_id` - FK к orders
- `return_tasks.order_id`, `return_tasks.customer_id` - FK к orders/users
- `batch_picking_tasks.order_ids` - массив FK к orders

### Интеграция с Listings Service
- `product_locations.listing_id` - товар из Listings MS (без FK - microservice boundary)
- `wms_reservations.listing_id` - резервация товара
- `picking_task_items.listing_id`, `batch_picking_items.listing_id`
- Синхронизация: через события NATS (listing.updated, listing.deleted)

### Интеграция с Auth Service
- gRPC запросы для синхронизации данных пользователей
- Кеш: `warehouse_staff.user_email`, `warehouse_staff.user_name`
- Валидация: проверка прав доступа (кто может управлять складом)

### Интеграция с Delivery Service
- `packing_tasks.tracking_number` - трекинг доставки
- События: ShipmentCreated (WMS → Delivery), ShipmentStatusChanged (Delivery → WMS)
- Webhook: обновление статуса доставки

### Интеграция с MinIO (s3.vondi.rs)
- `qc_photos.photo_url` - фото дефектов при QC
- `return_items.photo_url`, `return_item_photos.photo_url` - фото возвратов
- `packing_tasks.label_url` - этикетки доставки (PDF)
- Bucket: `wms-qc-photos`, `wms-return-photos`, `wms-labels`

### Интеграция с Redis
- Кеш доступного стока: `warehouse:{id}:stock:{listing_id}`
- Worker статусы: `worker:{id}:status` (active/on_break/offline)
- Real-time updates: через WebSocket (новые задачи, статусы)
- Session: резервации корзины (TTL 30 минут)

### События (NATS JetStream)
**Публикует:**
- `warehouse.inventory.updated` - изменение стока
- `warehouse.picking.completed` - завершена комплектация
- `warehouse.packing.completed` - завершена упаковка
- `warehouse.return.created` - создан возврат

**Подписывается:**
- `orders.created` - создание picking task
- `orders.cancelled` - отмена задач и освобождение резерваций
- `shipments.status_changed` - обновление tracking_number
- `listings.updated` - обновление SKU кеша

## Триггеры и автоматика

### Автообновление updated_at
Все основные таблицы имеют триггер `update_updated_at_column()` для автоматического обновления поля `updated_at` при UPDATE.

### Cycle Counts автопрогресс
- `trigger_calculate_cycle_count_variance` - автоматический расчёт `variance = actual_qty - expected_qty`
- `trigger_update_cycle_count_progress` - обновление счётчиков `counted_items`, `discrepancies` в родительской задаче cycle_counts

### QC Checklist автоматика
- `trigger_set_qc_checklist_checked_at` - автоматическое заполнение `checked_at` при `is_checked = TRUE`
- Откат: если `is_checked` сбрасывается в FALSE, то `checked_at` также становится NULL

### Reserved Quantity Management
При создании/отмене резервации автоматически обновляется `product_locations.reserved_quantity` (через application logic, не триггер - для контроля).

## Мониторинг и диагностика

```sql
-- Размер базы данных
SELECT pg_size_pretty(pg_database_size('wms_dev_db'));

-- Размер таблиц (топ 10)
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

-- Количество подключений
SELECT COUNT(*) FROM pg_stat_activity WHERE datname = 'wms_dev_db';

-- Активные запросы
SELECT
    pid, usename, state, query,
    now() - query_start AS duration
FROM pg_stat_activity
WHERE datname = 'wms_dev_db' AND state != 'idle'
ORDER BY duration DESC;

-- Неиспользуемые индексы (кандидаты на удаление)
SELECT
    schemaname, tablename, indexname,
    pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size,
    idx_scan as index_scans
FROM pg_stat_user_indexes i
JOIN pg_index USING (indexrelid)
WHERE schemaname = 'public'
  AND idx_scan = 0
  AND indisunique IS FALSE
ORDER BY pg_relation_size(i.indexrelid) DESC;

-- Таблицы с наибольшим количеством записей
SELECT
    schemaname, tablename,
    n_live_tup as row_count,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_live_tup DESC
LIMIT 10;
```

## Environment Variables

```bash
# В .env файле warehouse микросервиса
VONDIWMS_DB_HOST=localhost
VONDIWMS_DB_PORT=35435               # НЕ 5433! НЕ 35434!
VONDIWMS_DB_NAME=wms_dev_db          # НЕ vondi_db!
VONDIWMS_DB_USER=wms_user            # НЕ postgres!
VONDIWMS_DB_PASSWORD=wms_secret
VONDIWMS_DB_SSLMODE=disable

# Redis
VONDIWMS_REDIS_HOST=localhost
VONDIWMS_REDIS_PORT=36381

# gRPC
VONDIWMS_GRPC_PORT=50052

# HTTP
VONDIWMS_HTTP_PORT=8085

# NATS
VONDIWMS_NATS_URL=nats://localhost:4222

# MinIO
VONDIWMS_MINIO_ENDPOINT=s3.vondi.rs
VONDIWMS_MINIO_ACCESS_KEY=...
VONDIWMS_MINIO_SECRET_KEY=...
VONDIWMS_MINIO_BUCKET=wms-qc-photos
```

## Документация

- **CLAUDE.md:** `/p/github.com/vondi-global/warehouse/CLAUDE.md`
- **Миграции:** `/p/github.com/vondi-global/warehouse/migrations/`
- **Domain models:** `/p/github.com/vondi-global/warehouse/internal/domain/`
- **WMS Architecture:** `/p/github.com/vondi-global/docs/wms/WMS_ARCHITECTURE.md`
- **Warehouse Ownership Audit:** `/p/github.com/vondi-global/docs/wms/WMS_WAREHOUSE_OWNERSHIP_AUDIT.md`
- **API Proto:** `/p/github.com/vondi-global/warehouse/api/proto/warehouse.proto`

---

**Дата создания паспорта:** 2025-12-21
**Автор:** Claude Code
**Версия:** 1.0.0
