# Database Passport: Delivery Service

**Microservice:** Delivery Service
**Database:** delivery_db
**Owner:** VONDI GLOBAL D.O.O.
**Last Updated:** 2025-12-21
**Schema Version:** Migration 0007

---

## 1. Connection Information

### Local Development
```bash
# PostgreSQL Connection
Host: localhost
Port: 35432
Database: delivery_db
User: delivery_user
Password: delivery_secure_pass_2025

# Connection String
psql "postgres://delivery_user:delivery_secure_pass_2025@localhost:35432/delivery_db"

# Docker Container (if using docker-compose)
docker exec -it delivery-postgres psql -U delivery_user -d delivery_db
```

### Environment Variables
```env
DB_HOST=localhost
DB_PORT=35432
DB_NAME=delivery_db
DB_USER=delivery_user
DB_PASSWORD=delivery_secure_pass_2025
DB_SSLMODE=disable
```

---

## 2. Database Overview

### Purpose
Manages all delivery operations including shipments, tracking, provider integration (Post Express, BEX, DExpress), delivery zones, pricing rules, and notifications.

### Key Features
- Multi-provider shipment management (Post Express, generic providers)
- Real-time tracking events
- Delivery zone and pricing management
- Location-based delivery options
- Notification system for delivery status updates
- Integration with marketplace and storefront orders

### Statistics
- **Tables:** 17 core tables + 2 auxiliary tables (shipments, tracking_events from 0001 migration)
- **Migrations:** 7 applied migrations
- **Indexes:** 50+ indexes for performance
- **Sequences:** 14 sequences
- **ENUMs:** 2 types (shipment_status, delivery_provider)
- **Foreign Keys:** 8 relationships
- **Extensions:** PostGIS (geometry), uuid-ossp

---

## 3. Table Schemas

### 3.1. shipments (Generic Shipments - Migration 0001)

**Purpose:** Generic shipment tracking for any delivery provider.

```sql
CREATE TYPE shipment_status AS ENUM (
    'pending',
    'confirmed',
    'in_transit',
    'out_for_delivery',
    'delivered',
    'failed',
    'cancelled',
    'returned'
);

CREATE TYPE delivery_provider AS ENUM (
    'dex',
    'post_rs'
);

CREATE TABLE shipments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tracking_number VARCHAR(255) NOT NULL UNIQUE,
    provider delivery_provider NOT NULL,
    status shipment_status NOT NULL DEFAULT 'pending',

    -- From address
    from_street VARCHAR(500) NOT NULL,
    from_city VARCHAR(100) NOT NULL,
    from_state VARCHAR(100),
    from_postal_code VARCHAR(20) NOT NULL,
    from_country VARCHAR(2) NOT NULL,
    from_contact_name VARCHAR(255) NOT NULL,
    from_contact_phone VARCHAR(50) NOT NULL,

    -- To address
    to_street VARCHAR(500) NOT NULL,
    to_city VARCHAR(100) NOT NULL,
    to_state VARCHAR(100),
    to_postal_code VARCHAR(20) NOT NULL,
    to_country VARCHAR(2) NOT NULL,
    to_contact_name VARCHAR(255) NOT NULL,
    to_contact_phone VARCHAR(50) NOT NULL,

    -- Package details
    package_weight DECIMAL(10, 2) NOT NULL,
    package_length DECIMAL(10, 2) NOT NULL,
    package_width DECIMAL(10, 2) NOT NULL,
    package_height DECIMAL(10, 2) NOT NULL,
    package_description TEXT,
    package_declared_value DECIMAL(12, 2),

    -- Cost
    cost DECIMAL(12, 2),
    currency VARCHAR(3) DEFAULT 'RSD',

    -- User reference
    user_id UUID NOT NULL,

    -- Provider data
    provider_shipment_id VARCHAR(255),
    provider_metadata JSONB,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    estimated_delivery_at TIMESTAMP WITH TIME ZONE,
    actual_delivery_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT check_weight CHECK (package_weight > 0),
    CONSTRAINT check_dimensions CHECK (
        package_length > 0 AND
        package_width > 0 AND
        package_height > 0
    )
);

-- Indexes
CREATE INDEX idx_shipments_tracking_number ON shipments(tracking_number);
CREATE INDEX idx_shipments_user_id ON shipments(user_id);
CREATE INDEX idx_shipments_status ON shipments(status);
CREATE INDEX idx_shipments_provider ON shipments(provider);
CREATE INDEX idx_shipments_created_at ON shipments(created_at DESC);

-- Trigger
CREATE TRIGGER update_shipments_updated_at
    BEFORE UPDATE ON shipments
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

---

### 3.2. tracking_events (Shipment Tracking - Migration 0001)

**Purpose:** Stores tracking events for generic shipments.

```sql
CREATE TABLE tracking_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id) ON DELETE CASCADE,
    status shipment_status NOT NULL,
    location VARCHAR(255),
    description TEXT NOT NULL,
    event_timestamp TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_tracking_events_shipment_id ON tracking_events(shipment_id);
CREATE INDEX idx_tracking_events_timestamp ON tracking_events(event_timestamp DESC);
```

---

### 3.3. deliveries (C2C Delivery Orders - Migration 0002)

**Purpose:** C2C marketplace delivery orders with courier assignment.

```sql
CREATE TABLE deliveries (
    id INTEGER PRIMARY KEY,
    order_id INTEGER,
    courier_id INTEGER,
    status VARCHAR(50) DEFAULT 'pending' NOT NULL,

    -- Pickup
    pickup_address TEXT NOT NULL,
    pickup_latitude NUMERIC(10,8),
    pickup_longitude NUMERIC(11,8),
    pickup_contact_name VARCHAR(255),
    pickup_contact_phone VARCHAR(50),

    -- Delivery
    delivery_address TEXT NOT NULL,
    delivery_latitude NUMERIC(10,8),
    delivery_longitude NUMERIC(11,8),
    delivery_contact_name VARCHAR(255),
    delivery_contact_phone VARCHAR(50),

    -- Timeline
    assigned_at TIMESTAMP WITH TIME ZONE,
    accepted_at TIMESTAMP WITH TIME ZONE,
    picked_up_at TIMESTAMP WITH TIME ZONE,
    delivered_at TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,
    estimated_pickup_time TIMESTAMP WITH TIME ZONE,
    estimated_delivery_time TIMESTAMP WITH TIME ZONE,
    actual_delivery_time TIMESTAMP WITH TIME ZONE,

    -- Tracking
    tracking_token VARCHAR(100) DEFAULT gen_random_uuid()::text NOT NULL,
    tracking_url TEXT,
    share_location_enabled BOOLEAN DEFAULT TRUE,

    -- Route
    distance_meters INTEGER,
    duration_seconds INTEGER,
    route_polyline TEXT,

    -- Payment
    courier_fee NUMERIC(10,2),
    courier_tip NUMERIC(10,2) DEFAULT 0,

    -- Package
    package_size VARCHAR(20),
    package_weight_kg NUMERIC(6,2),
    requires_signature BOOLEAN DEFAULT FALSE,

    -- Proof and feedback
    photo_proof_url TEXT,
    customer_rating INTEGER,
    customer_feedback TEXT,
    notes TEXT,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT deliveries_customer_rating_check
        CHECK (customer_rating >= 1 AND customer_rating <= 5),
    CONSTRAINT deliveries_package_size_check
        CHECK (package_size IN ('small', 'medium', 'large', 'xl')),
    CONSTRAINT delivery_status_check
        CHECK (status IN (
            'pending', 'assigned', 'accepted', 'picked_up',
            'in_transit', 'delivered', 'cancelled', 'failed'
        ))
);

-- Indexes
CREATE INDEX idx_deliveries_courier ON deliveries(courier_id, status);
CREATE INDEX idx_deliveries_order ON deliveries(order_id);
CREATE INDEX idx_deliveries_status ON deliveries(status)
    WHERE status IN ('assigned', 'accepted', 'picked_up', 'in_transit');
CREATE INDEX idx_deliveries_tracking_token ON deliveries(tracking_token);
```

---

### 3.4. delivery_providers (Provider Registry - Migration 0002)

**Purpose:** Delivery provider configuration and capabilities.

```sql
CREATE TABLE delivery_providers (
    id INTEGER PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    logo_url VARCHAR(500),
    is_active BOOLEAN DEFAULT FALSE,

    -- Capabilities
    supports_cod BOOLEAN DEFAULT FALSE,
    supports_insurance BOOLEAN DEFAULT FALSE,
    supports_tracking BOOLEAN DEFAULT TRUE,

    -- Configuration
    api_config JSONB,
    capabilities JSONB,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);
```

---

### 3.5. delivery_shipments (Provider Shipments - Migration 0002)

**Purpose:** Shipments created with specific delivery providers.

```sql
CREATE TABLE delivery_shipments (
    id INTEGER PRIMARY KEY,
    uuid UUID DEFAULT uuid_generate_v4() UNIQUE NOT NULL,  -- Added in 0003
    user_uuid UUID,                                        -- Added in 0003
    order_uuid UUID,                                       -- Added in 0003
    provider_id INTEGER REFERENCES delivery_providers(id),
    order_id INTEGER,
    external_id VARCHAR(255),
    tracking_number VARCHAR(255) UNIQUE,
    status VARCHAR(50) DEFAULT 'pending' NOT NULL,

    -- Addresses (JSONB)
    sender_info JSONB NOT NULL,
    recipient_info JSONB NOT NULL,

    -- Package
    package_info JSONB NOT NULL,

    -- Pricing
    delivery_cost NUMERIC(10,2),
    insurance_cost NUMERIC(10,2),
    cod_amount NUMERIC(10,2),
    cost_breakdown JSONB,

    -- Schedule
    pickup_date DATE,
    estimated_delivery DATE,
    actual_delivery_date TIMESTAMP WITH TIME ZONE,

    -- Provider response
    provider_response JSONB,
    labels JSONB,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Indexes (0002 + 0003)
CREATE INDEX idx_delivery_shipments_order ON delivery_shipments(order_id);
CREATE INDEX idx_delivery_shipments_status ON delivery_shipments(status);
CREATE INDEX idx_delivery_shipments_tracking ON delivery_shipments(tracking_number);
CREATE INDEX idx_delivery_shipments_uuid ON delivery_shipments(uuid);
CREATE INDEX idx_delivery_shipments_user_uuid ON delivery_shipments(user_uuid);
CREATE INDEX idx_delivery_shipments_order_uuid ON delivery_shipments(order_uuid);
```

---

### 3.6. delivery_tracking_events (Provider Tracking - Migration 0002)

**Purpose:** Tracking events from delivery providers.

```sql
CREATE TABLE delivery_tracking_events (
    id INTEGER PRIMARY KEY,
    shipment_id INTEGER REFERENCES delivery_shipments(id),
    shipment_uuid UUID REFERENCES delivery_shipments(uuid),  -- Added in 0003
    provider_id INTEGER REFERENCES delivery_providers(id),
    event_time TIMESTAMP WITH TIME ZONE NOT NULL,
    status VARCHAR(100) NOT NULL,
    location VARCHAR(500),
    description TEXT,
    raw_data JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Indexes
CREATE INDEX idx_delivery_tracking_events_shipment ON delivery_tracking_events(shipment_id);
CREATE INDEX idx_delivery_tracking_events_time ON delivery_tracking_events(event_time);
CREATE INDEX idx_delivery_tracking_events_shipment_uuid
    ON delivery_tracking_events(shipment_uuid);

-- Trigger (0003)
CREATE TRIGGER sync_delivery_tracking_events_uuid
    BEFORE INSERT OR UPDATE ON delivery_tracking_events
    FOR EACH ROW
    EXECUTE FUNCTION sync_shipment_uuid();
```

---

### 3.7. delivery_zones (Geographic Zones - Migration 0002)

**Purpose:** Define delivery zones for pricing and availability.

```sql
CREATE TABLE delivery_zones (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL,

    -- Geographic filters
    countries TEXT[],
    regions TEXT[],
    cities TEXT[],
    postal_codes TEXT[],

    -- Geometry (PostGIS)
    boundary GEOMETRY(Polygon, 4326),
    center_point GEOMETRY(Point, 4326),
    radius_km NUMERIC(10,2),

    created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Indexes
CREATE INDEX idx_delivery_zones_boundary ON delivery_zones USING gist(boundary);
CREATE INDEX idx_delivery_zones_center ON delivery_zones USING gist(center_point);
```

---

### 3.8. delivery_pricing_rules (Dynamic Pricing - Migration 0002)

**Purpose:** Flexible pricing rules based on weight, volume, zones.

```sql
CREATE TABLE delivery_pricing_rules (
    id INTEGER PRIMARY KEY,
    provider_id INTEGER REFERENCES delivery_providers(id),
    rule_type VARCHAR(50) NOT NULL,

    -- Ranges (JSONB)
    weight_ranges JSONB,
    volume_ranges JSONB,
    zone_multipliers JSONB,

    -- Surcharges
    fragile_surcharge NUMERIC(10,2) DEFAULT 0,
    oversized_surcharge NUMERIC(10,2) DEFAULT 0,
    special_handling_surcharge NUMERIC(10,2) DEFAULT 0,

    -- Limits
    min_price NUMERIC(10,2),
    max_price NUMERIC(10,2),

    -- Custom
    custom_formula TEXT,
    priority INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Indexes
CREATE INDEX idx_delivery_pricing_rules_provider ON delivery_pricing_rules(provider_id);
CREATE INDEX idx_delivery_pricing_rules_active ON delivery_pricing_rules(is_active)
    WHERE is_active = TRUE;
```

---

### 3.9. delivery_category_defaults (Package Defaults - Migration 0002, 0007)

**⚠️ ВАЖНО:** Категории хранятся в Listings Microservice, НЕ в монолите!

**Purpose:** Default package dimensions for product categories.

```sql
CREATE TABLE delivery_category_defaults (
    id INTEGER PRIMARY KEY,
    category_id UUID NOT NULL UNIQUE,  -- Changed to UUID in 0007
    default_weight_kg NUMERIC(10,3),
    default_length_cm NUMERIC(10,2),
    default_width_cm NUMERIC(10,2),
    default_height_cm NUMERIC(10,2),
    default_packaging_type VARCHAR(50),
    is_typically_fragile BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Indexes
CREATE INDEX idx_delivery_category_defaults_category
    ON delivery_category_defaults(category_id);

-- Comment
COMMENT ON COLUMN delivery_category_defaults.category_id IS
    'UUID reference to category from Listings microservice (Source of Truth)';
```

---

### 3.10. delivery_notifications (Notification Queue - Migration 0002)

**Purpose:** Delivery status notification queue.

```sql
CREATE TABLE delivery_notifications (
    id INTEGER PRIMARY KEY,
    shipment_id INTEGER REFERENCES delivery_shipments(id) ON DELETE CASCADE,
    shipment_uuid UUID REFERENCES delivery_shipments(uuid),  -- Added in 0003
    user_id INTEGER,
    channel VARCHAR(20) NOT NULL,  -- email, sms, push, webhook
    status VARCHAR(20) DEFAULT 'pending' NOT NULL,
    template VARCHAR(50),
    data JSONB,
    sent_at TIMESTAMP WITH TIME ZONE,
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Indexes
CREATE INDEX idx_delivery_notifications_shipment ON delivery_notifications(shipment_id);
CREATE INDEX idx_delivery_notifications_user ON delivery_notifications(user_id);
CREATE INDEX idx_delivery_notifications_channel ON delivery_notifications(channel);
CREATE INDEX idx_delivery_notifications_status ON delivery_notifications(status);
CREATE INDEX idx_delivery_notifications_created ON delivery_notifications(created_at);

-- Trigger (0003)
CREATE TRIGGER sync_delivery_notifications_uuid
    BEFORE INSERT OR UPDATE ON delivery_notifications
    FOR EACH ROW
    EXECUTE FUNCTION sync_shipment_uuid();
```

---

### 3.11. post_express_locations (Serbian Locations - Migration 0002)

**Purpose:** Post Express settlement database for Serbia.

```sql
CREATE TABLE post_express_locations (
    id INTEGER PRIMARY KEY,
    post_express_id INTEGER UNIQUE,
    name VARCHAR(255) NOT NULL,
    name_cyrillic VARCHAR(255),
    postal_code VARCHAR(20),
    municipality VARCHAR(255),
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    region VARCHAR(255),
    district VARCHAR(255),
    delivery_zone VARCHAR(50),
    is_active BOOLEAN DEFAULT TRUE,
    supports_cod BOOLEAN DEFAULT TRUE,
    supports_express BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_post_express_locations_name ON post_express_locations(name);
CREATE INDEX idx_post_express_locations_postal_code ON post_express_locations(postal_code);
```

---

### 3.12. post_express_offices (Post Offices - Migration 0002)

**Purpose:** Post Express office locations.

```sql
CREATE TABLE post_express_offices (
    id INTEGER PRIMARY KEY,
    office_code VARCHAR(50) NOT NULL UNIQUE,
    location_id INTEGER REFERENCES post_express_locations(id),
    name VARCHAR(255) NOT NULL,
    address VARCHAR(500) NOT NULL,
    phone VARCHAR(50),
    email VARCHAR(255),
    working_hours JSONB,
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,

    -- Services
    accepts_packages BOOLEAN DEFAULT TRUE,
    issues_packages BOOLEAN DEFAULT TRUE,

    -- Amenities
    has_atm BOOLEAN DEFAULT FALSE,
    has_parking BOOLEAN DEFAULT FALSE,
    wheelchair_accessible BOOLEAN DEFAULT FALSE,

    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    temporary_closed BOOLEAN DEFAULT FALSE,
    closed_until TIMESTAMP WITH TIME ZONE,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_post_express_offices_location_id ON post_express_offices(location_id);
CREATE INDEX idx_post_express_offices_is_active ON post_express_offices(is_active);
```

---

### 3.13. post_express_rates (Shipping Rates - Migration 0002)

**Purpose:** Post Express pricing tiers.

```sql
CREATE TABLE post_express_rates (
    id INTEGER PRIMARY KEY,
    weight_from NUMERIC(10,3) NOT NULL,
    weight_to NUMERIC(10,3) NOT NULL,
    base_price NUMERIC(10,2) NOT NULL,
    insurance_included_up_to NUMERIC(10,2) DEFAULT 15000,
    insurance_rate_percent NUMERIC(5,2) DEFAULT 1.0,
    cod_fee NUMERIC(10,2) DEFAULT 45,

    -- Package limits
    max_length_cm INTEGER DEFAULT 60,
    max_width_cm INTEGER DEFAULT 60,
    max_height_cm INTEGER DEFAULT 60,
    max_dimensions_sum_cm INTEGER DEFAULT 180,

    -- Delivery time
    delivery_days_min INTEGER DEFAULT 1,
    delivery_days_max INTEGER DEFAULT 3,

    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    is_special_offer BOOLEAN DEFAULT FALSE,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

### 3.14. post_express_settings (API Configuration - Migration 0002)

**Purpose:** Post Express API credentials and sender info.

```sql
CREATE TABLE post_express_settings (
    id INTEGER PRIMARY KEY,

    -- API credentials
    api_username VARCHAR(255) NOT NULL,
    api_password VARCHAR(255) NOT NULL,
    api_endpoint VARCHAR(500) DEFAULT 'https://wsp.postexpress.rs/api' NOT NULL,

    -- Sender info
    sender_name VARCHAR(255) NOT NULL,
    sender_address VARCHAR(500) NOT NULL,
    sender_city VARCHAR(255) NOT NULL,
    sender_postal_code VARCHAR(20) NOT NULL,
    sender_phone VARCHAR(50) NOT NULL,
    sender_email VARCHAR(255),

    -- Settings
    enabled BOOLEAN DEFAULT TRUE,
    test_mode BOOLEAN DEFAULT TRUE,
    auto_print_labels BOOLEAN DEFAULT FALSE,
    auto_track_shipments BOOLEAN DEFAULT FALSE,

    -- Notifications
    notify_on_pickup BOOLEAN DEFAULT FALSE,
    notify_on_delivery BOOLEAN DEFAULT TRUE,
    notify_on_failed_delivery BOOLEAN DEFAULT TRUE,

    -- Statistics
    total_shipments INTEGER DEFAULT 0,
    successful_deliveries INTEGER DEFAULT 0,
    failed_deliveries INTEGER DEFAULT 0,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

### 3.15. post_express_shipments (Serbian Shipments - Migration 0002)

**Purpose:** Shipments created via Post Express API.

```sql
CREATE TABLE post_express_shipments (
    id INTEGER PRIMARY KEY,

    -- Order references
    marketplace_order_id INTEGER,
    storefront_order_id BIGINT,

    -- Tracking
    tracking_number VARCHAR(255) UNIQUE,
    barcode VARCHAR(255),
    post_express_id VARCHAR(255),

    -- Sender
    sender_name VARCHAR(255) NOT NULL,
    sender_address VARCHAR(500) NOT NULL,
    sender_city VARCHAR(255) NOT NULL,
    sender_postal_code VARCHAR(20) NOT NULL,
    sender_phone VARCHAR(50) NOT NULL,
    sender_email VARCHAR(255),
    sender_location_id INTEGER,

    -- Recipient
    recipient_name VARCHAR(255) NOT NULL,
    recipient_address VARCHAR(500) NOT NULL,
    recipient_city VARCHAR(255) NOT NULL,
    recipient_postal_code VARCHAR(20) NOT NULL,
    recipient_phone VARCHAR(50) NOT NULL,
    recipient_email VARCHAR(255),
    recipient_location_id INTEGER,

    -- Package
    weight NUMERIC(10,3) NOT NULL,
    length_cm INTEGER,
    width_cm INTEGER,
    height_cm INTEGER,
    package_contents TEXT,
    declared_value NUMERIC(10,2),

    -- Service
    service_type VARCHAR(50) DEFAULT 'standard',
    cod_amount NUMERIC(10,2),
    cod_reference VARCHAR(255),
    insurance_amount NUMERIC(10,2),
    express_delivery BOOLEAN DEFAULT FALSE,
    office_pickup BOOLEAN DEFAULT FALSE,
    office_code VARCHAR(50),

    -- Status
    status VARCHAR(50) DEFAULT 'created',
    status_description TEXT,
    delivery_status VARCHAR(100),
    status_history JSONB DEFAULT '[]',

    -- Timeline
    last_tracking_update TIMESTAMP WITH TIME ZONE,
    pickup_date TIMESTAMP WITH TIME ZONE,
    delivery_date TIMESTAMP WITH TIME ZONE,
    registered_at TIMESTAMP WITH TIME ZONE,
    picked_up_at TIMESTAMP WITH TIME ZONE,
    delivered_at TIMESTAMP WITH TIME ZONE,
    failed_at TIMESTAMP WITH TIME ZONE,
    returned_at TIMESTAMP WITH TIME ZONE,

    -- Pricing
    base_price NUMERIC(10,2),
    insurance_fee NUMERIC(10,2),
    cod_fee NUMERIC(10,2),
    total_price NUMERIC(10,2),

    -- Documents
    label_url TEXT,
    label_printed_at TIMESTAMP WITH TIME ZONE,
    receipt_url TEXT,
    invoice_url TEXT,
    invoice_number VARCHAR(255),
    invoice_date TIMESTAMP WITH TIME ZONE,
    pod_url TEXT,

    -- Instructions
    delivery_instructions TEXT,
    notes TEXT,
    internal_notes TEXT,
    failed_reason TEXT,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_post_express_shipments_marketplace_order_id
    ON post_express_shipments(marketplace_order_id);
CREATE INDEX idx_post_express_shipments_storefront_order_id
    ON post_express_shipments(storefront_order_id);
CREATE INDEX idx_post_express_shipments_tracking_number
    ON post_express_shipments(tracking_number);
CREATE INDEX idx_post_express_shipments_status
    ON post_express_shipments(status);
```

---

### 3.16. post_express_tracking_events (Serbian Tracking - Migration 0002)

**Purpose:** Tracking events from Post Express.

```sql
CREATE TABLE post_express_tracking_events (
    id INTEGER PRIMARY KEY,
    shipment_id INTEGER REFERENCES post_express_shipments(id) ON DELETE CASCADE,
    event_code VARCHAR(50),
    event_description TEXT,
    event_location VARCHAR(255),
    event_timestamp TIMESTAMP WITH TIME ZONE,
    additional_info JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_post_express_tracking_events_shipment_id
    ON post_express_tracking_events(shipment_id);
CREATE INDEX idx_post_express_tracking_events_timestamp
    ON post_express_tracking_events(event_timestamp);
```

---

### 3.17. delivery_addresses (Checkout Addresses - Migration 0005)

**Purpose:** Delivery addresses for specific orders (copied from auth.user_addresses).

```sql
CREATE TABLE delivery_addresses (
    id INTEGER PRIMARY KEY,

    -- References
    order_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    source_address_id INTEGER,  -- Original from auth.user_addresses

    -- Recipient
    recipient_name VARCHAR(255) NOT NULL,
    recipient_phone VARCHAR(50) NOT NULL,
    recipient_email VARCHAR(255),

    -- Address (sr_latin for providers)
    street VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    district VARCHAR(100),
    postal_code VARCHAR(20) NOT NULL,
    country_code VARCHAR(3) NOT NULL DEFAULT 'RS',

    -- Building
    building VARCHAR(50),
    apartment VARCHAR(50),
    entrance VARCHAR(50),
    floor VARCHAR(20),
    intercom VARCHAR(50),
    notes TEXT,

    -- Geolocation
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,

    -- Post Express specific
    post_express_settlement_id INTEGER,
    post_express_street_id INTEGER,

    -- Delivery method
    delivery_provider VARCHAR(50),  -- post_express, bex, dexpress
    delivery_method VARCHAR(100),   -- courier, pickup_point, locker
    pickup_point_id VARCHAR(100),

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_delivery_addresses_order_id ON delivery_addresses(order_id);
CREATE INDEX idx_delivery_addresses_user_id ON delivery_addresses(user_id);
CREATE INDEX idx_delivery_addresses_source ON delivery_addresses(source_address_id)
    WHERE source_address_id IS NOT NULL;
CREATE INDEX idx_delivery_addresses_provider ON delivery_addresses(delivery_provider)
    WHERE delivery_provider IS NOT NULL;
CREATE INDEX idx_delivery_addresses_city ON delivery_addresses(city);
CREATE INDEX idx_delivery_addresses_postal ON delivery_addresses(postal_code);
CREATE INDEX idx_delivery_addresses_created ON delivery_addresses(created_at);

-- Constraints
ALTER TABLE delivery_addresses
    ADD CONSTRAINT chk_delivery_country_code
    CHECK (country_code ~ '^[A-Z]{2,3}$');

ALTER TABLE delivery_addresses
    ADD CONSTRAINT chk_delivery_postal_code
    CHECK (postal_code ~ '^\d{5}$');

ALTER TABLE delivery_addresses
    ADD CONSTRAINT chk_delivery_latitude
    CHECK (latitude IS NULL OR (latitude >= -90 AND latitude <= 90));

ALTER TABLE delivery_addresses
    ADD CONSTRAINT chk_delivery_longitude
    CHECK (longitude IS NULL OR (longitude >= -180 AND longitude <= 180));
```

---

## 4. Entity Relationship Diagram (ERD)

```
┌─────────────────────────┐
│   delivery_providers    │
│─────────────────────────│
│ id (PK)                 │
│ code (UNIQUE)           │
│ name                    │
│ supports_cod            │
│ supports_tracking       │
└────────┬────────────────┘
         │
         │ 1
         │
         │ N
┌────────▼────────────────┐
│  delivery_shipments     │◄──────┐
│─────────────────────────│       │
│ id (PK)                 │       │
│ uuid (UNIQUE) 🔑        │       │ 1
│ provider_id (FK)        │       │
│ tracking_number         │       │
│ sender_info (JSONB)     │       │
│ recipient_info (JSONB)  │       │ N
│ package_info (JSONB)    │       │
└────────┬────────────────┘       │
         │                        │
         │ 1                      │
         │                        │
         │ N          ┌───────────┴─────────────┐
┌────────▼────────────▼───────────┐             │
│ delivery_tracking_events        │             │
│─────────────────────────────────│             │
│ id (PK)                         │             │
│ shipment_id (FK)                │             │
│ shipment_uuid (FK) 🔑           │             │
│ provider_id (FK)                │             │
│ event_time                      │             │
│ status                          │             │
└─────────────────────────────────┘             │
                                                │
         ┌──────────────────────────────────────┤
         │                                      │
         │ 1                                    │
         │                                      │
         │ N                                    │
┌────────▼────────────────┐                     │
│ delivery_notifications  │                     │
│─────────────────────────│                     │
│ id (PK)                 │                     │
│ shipment_id (FK)        │                     │
│ shipment_uuid (FK) 🔑   │                     │
│ channel                 │                     │
│ status                  │                     │
└─────────────────────────┘                     │
                                                │
┌────────────────────────┐                      │
│ delivery_pricing_rules │                      │
│────────────────────────│                      │
│ id (PK)                │                      │
│ provider_id (FK) ───────┘                      │
│ weight_ranges (JSONB)  │                      │
└────────────────────────┘                      │

┌─────────────────────────┐
│      shipments          │  (Generic, UUID-based)
│─────────────────────────│
│ id (PK, UUID)           │
│ user_id (UUID)          │
│ tracking_number         │
│ provider (ENUM)         │
│ status (ENUM)           │
│ from_* (address)        │
│ to_* (address)          │
│ package_*               │
└────────┬────────────────┘
         │
         │ 1
         │
         │ N
┌────────▼────────────────┐
│   tracking_events       │
│─────────────────────────│
│ id (PK, UUID)           │
│ shipment_id (FK)        │
│ status (ENUM)           │
│ location                │
│ event_timestamp         │
└─────────────────────────┘

┌─────────────────────────┐
│   post_express_locations│
│─────────────────────────│
│ id (PK)                 │
│ post_express_id (UNIQUE)│
│ name                    │
│ postal_code             │
│ latitude, longitude     │
└────────┬────────────────┘
         │
         │ 1
         │
         │ N
┌────────▼────────────────┐
│  post_express_offices   │
│─────────────────────────│
│ id (PK)                 │
│ office_code (UNIQUE)    │
│ location_id (FK)        │
│ address                 │
│ working_hours (JSONB)   │
└─────────────────────────┘

┌─────────────────────────┐
│ post_express_shipments  │
│─────────────────────────│
│ id (PK)                 │
│ marketplace_order_id    │
│ tracking_number         │
│ sender_*                │
│ recipient_*             │
│ cod_amount              │
│ status_history (JSONB)  │
└────────┬────────────────┘
         │
         │ 1
         │
         │ N
┌────────▼─────────────────────┐
│ post_express_tracking_events │
│──────────────────────────────│
│ id (PK)                      │
│ shipment_id (FK)             │
│ event_code                   │
│ event_timestamp              │
└──────────────────────────────┘

┌─────────────────────────┐
│    deliveries           │  (C2C orders)
│─────────────────────────│
│ id (PK)                 │
│ order_id                │
│ courier_id              │
│ status                  │
│ pickup_* (address)      │
│ delivery_* (address)    │
│ tracking_token (UNIQUE) │
│ route_polyline          │
└─────────────────────────┘

┌─────────────────────────┐
│  delivery_addresses     │
│─────────────────────────│
│ id (PK)                 │
│ order_id                │
│ user_id                 │
│ source_address_id       │
│ recipient_*             │
│ street, city, postal    │
│ delivery_provider       │
│ delivery_method         │
└─────────────────────────┘

┌─────────────────────────┐
│ delivery_category_defaults│
│─────────────────────────│
│ id (PK)                 │
│ category_id (UUID) 🔑   │
│ default_weight_kg       │
│ default_*_cm            │
└─────────────────────────┘

┌─────────────────────────┐
│   delivery_zones        │
│─────────────────────────│
│ id (PK)                 │
│ name                    │
│ boundary (GEOMETRY)     │
│ center_point (GEOMETRY) │
└─────────────────────────┘

Legend:
  PK = Primary Key
  FK = Foreign Key
  🔑 = UUID (external reference)
  JSONB = JSON document
  ENUM = PostgreSQL ENUM type
  GEOMETRY = PostGIS spatial type
```

---

## 5. Migration History

| # | File | Description | Applied |
|---|------|-------------|---------|
| 0001 | `0001_create_shipments_table.up.sql` | Generic shipments and tracking (UUID-based) | ✅ |
| 0002 | `0002_delivery_tables.up.sql` | Main delivery tables (providers, zones, Post Express) | ✅ |
| 0003 | `0003_add_uuid_fields.up.sql` | Add UUID support to delivery_shipments | ✅ |
| 0004 | `0004_create_storefronts_tables.up.sql` | B2C storefront tables (REMOVED in 0006) | ❌ |
| 0005 | `0005_create_delivery_addresses.up.sql` | Delivery addresses for checkout | ✅ |
| 0006 | `0006_drop_b2c_rudiments.up.sql` | Remove B2C tables (boundary violation) | ✅ |
| 0007 | `0007_category_uuid.up.sql` | Change category_id to UUID | ✅ |

### Migration Notes

**0001 (Shipments):**
- Created generic shipment tracking system
- UUID-based IDs for external references
- ENUMs for status and provider
- Trigger for `updated_at` column

**0002 (Main Tables):**
- Imported 14 tables from monolith
- Post Express integration tables
- Delivery zones with PostGIS geometry
- JSONB for flexible data (working_hours, zones, status_history)

**0003 (UUID Support):**
- Added `uuid`, `user_uuid`, `order_uuid` to `delivery_shipments`
- Trigger to sync `shipment_uuid` in related tables
- Supports dual ID strategy (integer for FK, UUID for API)

**0004 (B2C Tables - REMOVED):**
- Created B2C storefront tables
- Violated microservice boundaries
- Removed in migration 0006

**0005 (Delivery Addresses):**
- Checkout-specific address storage
- Copied from auth.user_addresses at order time
- Support for Post Express settlement/street IDs
- Constraints for geo coordinates and postal codes

**0006 (B2C Cleanup):**
- Dropped 5 B2C tables (all empty)
- Reason: belong to storefront/b2c-service
- Clean microservice boundaries

**0007 (Category UUID):**
- Changed `category_id` from INTEGER to UUID
- Aligns with Listings microservice (Source of Truth)
- Safe: no production data

---

## 6. Common Queries

### Get shipment with tracking
```sql
SELECT
    s.uuid,
    s.tracking_number,
    s.status,
    json_agg(
        json_build_object(
            'timestamp', te.event_timestamp,
            'status', te.status,
            'location', te.location,
            'description', te.description
        ) ORDER BY te.event_timestamp DESC
    ) AS events
FROM delivery_shipments s
LEFT JOIN delivery_tracking_events te ON te.shipment_uuid = s.uuid
WHERE s.uuid = 'uuid-here'
GROUP BY s.id;
```

### Find delivery zones for city
```sql
SELECT * FROM delivery_zones
WHERE 'Beograd' = ANY(cities)
   OR ST_Contains(boundary, ST_SetSRID(ST_MakePoint(20.4489, 44.7866), 4326));
```

### Active Post Express shipments
```sql
SELECT
    tracking_number,
    recipient_name,
    recipient_city,
    status,
    created_at
FROM post_express_shipments
WHERE status IN ('created', 'registered', 'picked_up')
ORDER BY created_at DESC;
```

### Delivery address for order
```sql
SELECT
    da.recipient_name,
    da.street,
    da.city,
    da.postal_code,
    da.delivery_provider,
    da.delivery_method
FROM delivery_addresses da
WHERE da.order_id = 456;
```

---

## 7. Microservice Integration

### Dependencies
- **Auth Service** (user_id, user_uuid)
- **Listings Service** (order_id, order_uuid, category_id UUID)
- **Notification Service** (delivery status notifications)

### No Direct Foreign Keys
All external references use IDs without FK constraints (microservice boundary).

### Event Publishing
Delivery status changes → NATS events → Notification Service

---

## 8. Backup & Recovery

### Backup
```bash
pg_dump -h localhost -p 35432 -U delivery_user -d delivery_db \
    --no-owner --no-acl --column-inserts \
    -f delivery_db_backup_$(date +%Y%m%d_%H%M%S).sql
```

### Restore
```bash
psql -h localhost -p 35432 -U delivery_user -d delivery_db \
    -f delivery_db_backup_20251221.sql
```

---

**Document Version:** 1.0
**Database Schema Version:** 0007
**Total Tables:** 17 core + 2 auxiliary
**Maintained by:** Delivery Service Team
**Contact:** Malden Simić, Generalni direktor, VONDI GLOBAL D.O.O.
