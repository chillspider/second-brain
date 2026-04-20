# Silver Layer Design Log

This document tracks the step-by-step design decisions for the silver layer.

---

## Design Principles

1. **Always use IDs to link tables, never names**
   - Foreign keys: `station_id`, `zone_id`, `customer_id`
   - Names are for display only, denormalized where convenient

2. **Placeholders for manual data**
   - Silver layer includes all required columns even if NULL initially
   - Tools will check for NULL/empty required fields
   - Data fed via landing zone, pipeline re-run fills the gaps

3. **JSONB for flexible/extended data**
   - Use `extended_data` JSONB for URLs, configs, etc.

4. **Reference tables (ref_*) require editing tools**
   - All `ref_` prefixed tables are manually maintained
   - Need to build CLI/UI tools to add, edit, delete entries
   - Examples: `ref_station_zones`, future `ref_contacts`

5. **Layer responsibilities - NO mixing!**

   **Dimension tables (dim_):**
   - Descriptive attributes of an entity
   - Answer: "who, what, where"
   - Examples: name, type, category, status
   - Can have stable derived attributes that describe the entity (e.g., `customer_tier = 'gold'`)
   - Should NOT have rolling aggregations that change with every transaction

   **Fact tables (fct_):**
   - Measurements of business events
   - Answer: "how much, how many"
   - One row per event/transaction
   - Contains: amounts, quantities, foreign keys to dimensions

   **Gold layer:**
   - Analytics, aggregations, KPIs
   - Rolling counts, fraud scores, trends
   - Derived conclusions (is_suspicious, is_fraud, etc.)

---

## Tables Built

### 1. ref_station_zones (Reference Table)

**Purpose:** Define geographic hierarchy for stations (region → city → zone)

**Columns:**
| Column | Type | Description |
|--------|------|-------------|
| zone_id | UUID | PK |
| zone_code | STRING | e.g., 'hcm-east', 'hanoi-central' |
| zone_name | STRING | e.g., 'HCM East', 'Hanoi Central' |
| city_province | STRING | e.g., 'Hồ Chí Minh', 'Hà Nội' |
| region | STRING | e.g., 'HCM Metro', 'Hanoi Metro', 'North', 'South' |

**Data Source:** Manual (dbt seed CSV)

**Files:**
- Seed: `transform/seeds/ref_station_zones.csv`
- Schema: `transform/seeds/schema.yml`

**Status:** ✅ Complete

---

### 2. bronze_iot_devices (Landing Table)

**Purpose:** IoT devices (cameras, sensors, routers) per station

**Columns:**
| Column | Type | Description |
|--------|------|-------------|
| iot_device_id | UUID | PK |
| station_id | UUID | FK → stations |
| device_type | STRING | 'camera', 'sensor', 'router', 'gateway' |
| device_name | STRING | Friendly name |
| device_code | STRING | Serial number |
| ip_address | STRING | Local IP |
| tailscale_ip | STRING | Remote access IP |
| mac_address | STRING | Hardware address |
| status | STRING | 'online', 'offline', 'maintenance' |
| installed_at | TIMESTAMP | When installed |
| last_seen_at | TIMESTAMP | Last heartbeat |
| extended_data | STRING (JSON) | URLs, configs, credentials |
| contact_id | UUID | FK → contacts (future CRM) |
| notes | STRING | Free text |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

**Data Source:** Manual (landing zone CSV/API)

**Files:**
- Schema: `landing/schemas/bronze_iot_devices.json`
- Template: `landing/templates/bronze_iot_devices.csv`

**Status:** ✅ Schema defined

---

### 2.5. station_zone_mapping (Landing Table)

**Purpose:** Map stations to zones (manual enrichment for dim_stations)

**Columns:**
| Column | Type | Description |
|--------|------|-------------|
| station_id | UUID | FK → stations (must exist) |
| zone_id | UUID | FK → ref_station_zones |
| station_type | STRING | 'owned' or 'franchise' |
| partner_name | STRING | Partner name (if franchise) |
| extended_data | STRING (JSON) | Additional configs |

**Data Source:** Manual (landing zone CSV)

**Files:**
- Schema: `landing/schemas/station_zone_mapping.json`
- Template: `landing/templates/station_zone_mapping.csv`

**Status:** ✅ Schema defined

---

### 3. dim_stations (Silver - Updated)

**Purpose:** Station dimension with all attributes

**Columns:**
| Column | Type | Source | Description |
|--------|------|--------|-------------|
| station_id | UUID | bronze | PK |
| station_name | STRING | bronze | Display name |
| station_code | STRING | bronze.site_id | e.g., 'jetx-hcm-ldc' |
| status | STRING | bronze | active/inactive |
| address | STRING | order_item_details | Full address |
| latitude | DECIMAL | order_item_details | GPS |
| longitude | DECIMAL | order_item_details | GPS |
| open_time | TIME | bronze | |
| close_time | TIME | bronze | |
| zone_id | UUID | **manual** | FK → ref_station_zones |
| station_type | STRING | **manual** | 'owned', 'franchise' |
| partner_name | STRING | **manual** | NULL if owned |
| extended_data | STRING (JSON) | **manual** | Additional configs |
| created_at | TIMESTAMP | bronze | |
| updated_at | TIMESTAMP | bronze | |

**Status:** ✅ Built (with placeholders for manual data)

---

## Change Log

| Date | Change | Files Modified |
|------|--------|----------------|
| 2026-01-10 | Initial design - ref_station_zones, bronze_iot_devices, dim_stations | This doc |
| 2026-01-10 | Created ref_station_zones seed CSV | transform/seeds/ref_station_zones.csv |
| 2026-01-10 | Created seeds schema.yml | transform/seeds/schema.yml |
| 2026-01-10 | Updated dim_stations with placeholders | transform/models/silver/dim_stations.sql |
| 2026-01-10 | Created bronze_iot_devices schema | landing/schemas/bronze_iot_devices.json |
| 2026-01-10 | Created station_zone_mapping schema | landing/schemas/station_zone_mapping.json |
| 2026-01-10 | Added principle: ref_ tables need editing tools | Design Principles #4 |
| 2026-01-10 | Updated dim_customers with placeholders | transform/models/silver/dim_customers.sql |
| 2026-01-10 | Updated dim_devices with placeholders, clarified vs IoT | transform/models/silver/dim_devices.sql |
| 2026-01-10 | Reviewed all fact tables - confirmed ID-based joins | fct_washes, fct_transactions, fct_subscriptions |
| 2026-01-10 | Added design notes to all fact tables | fct_*.sql |
| 2026-01-10 | Auto-derive customer_segment='consumer' from source | transform/models/silver/dim_customers.sql |
| 2026-01-10 | Redesigned dim_vehicles: pure facts only, no aggregations | transform/models/silver/dim_vehicles.sql |
| 2026-01-10 | Added Design Principle #5: Layer responsibilities | docs/SILVER_LAYER_DESIGN_LOG.md |
| 2026-01-10 | Renamed dim_geo_hierarchy → ref_geo_vietnam | transform/models/silver/ref_geo_vietnam.sql |
| 2026-01-10 | Fixed fct_subscriptions: removed aggregations | transform/models/silver/fct_subscriptions.sql |
| 2026-01-10 | Merged fct_washes into fct_orders, deleted fct_washes | transform/models/silver/fct_orders.sql |
| 2026-01-10 | Fixed fct_transactions: transaction_type, removed is_successful | transform/models/silver/fct_transactions.sql |
| 2026-01-10 | Created dim_payment_profiles from JSONB normalization | transform/models/silver/dim_payment_profiles.sql |
| 2026-01-10 | Added payment_profile_id link to fct_transactions | transform/models/silver/fct_transactions.sql |
| 2026-01-10 | **Gold layer review started** | - |
| 2026-01-10 | Rebuilt gold_daily_revenue: fixed source (fct_washes → fct_orders + fct_transactions) | transform/models/gold/gold_daily_revenue.sql |
| 2026-01-10 | Added accounting logic: revenue vs cash_inflow vs prepaid_usage | transform/models/gold/gold_daily_revenue.sql |
| 2026-01-10 | Key insight: jetx_cash IS revenue, top-up is NOT revenue (liability) | docs/SILVER_LAYER_DESIGN_LOG.md |
| 2026-01-10 | Fixed gold_station_performance: same source fixes | transform/models/gold/gold_station_performance.sql |
| 2026-01-10 | **Immutability requirement identified** | - |
| 2026-01-10 | Designed immutable station attribution: freeze in bronze layer | docs/SILVER_LAYER_DESIGN_LOG.md |
| 2026-01-10 | Ingested subscription_locations (2,086 rows) | bronze_subscriptions.subscription_locations |
| 2026-01-10 | Created order_attribution extractor (161,716 rows) | bronze_operations.order_attribution |
| 2026-01-10 | Created subscription_attribution extractor (2,224 rows) | bronze_subscriptions.subscription_attribution |
| 2026-01-10 | Updated fct_orders with attributed_station_id | transform/models/silver/fct_orders.sql |
| 2026-01-10 | Updated fct_subscriptions with attributed_station_id, plan_type | transform/models/silver/fct_subscriptions.sql |
| 2026-01-10 | Updated sources.yml with new bronze tables | transform/models/staging/sources.yml |
| 2026-01-10 | Identified plan types: Silver (max_locations=1), Gold (max_locations=0/-1) | - |

---

## Silver Layer Table Inventory

### Reference Tables
| Table | Purpose | Source | Manual |
|-------|---------|--------|--------|
| ref_station_zones | Our business zones (region → city → zone) | dbt seed | Yes |
| ref_geo_vietnam | Vietnam admin (city → district → ward) | bronze source | No |

### Landing Tables (manual upload)
| Table | Purpose | Schema File |
|-------|---------|-------------|
| bronze_iot_devices | IoT devices per station | landing/schemas/bronze_iot_devices.json |
| station_zone_mapping | Station → Zone mapping | landing/schemas/station_zone_mapping.json |

### Dimension Tables
| Table | Primary Key | Manual Fields |
|-------|-------------|---------------|
| dim_stations | station_id | zone_id, station_type, partner_name, extended_data |
| dim_customers | user_id | contact_id, extended_data (customer_segment auto-derived) |
| dim_devices | device_id | maintenance_status, last_maintenance_at, extended_data |
| dim_vehicles | vehicle_id | vehicle_class, vehicle_type, license_plate_color, extended_data |
| dim_vouchers | voucher_id | - |
| dim_membership_plans | plan_id | - |
| dim_wash_modes | mode_id | - |
| ref_geo_vietnam | ward_id | - (Vietnam admin, not our zones) |
| dim_payment_profiles | payment_profile_id | - (normalized from JSONB) |

### Fact Tables
| Table | Primary Key | Grain |
|-------|-------------|-------|
| fct_orders | order_id | One row per order (includes wash details) |
| fct_transactions | transaction_id | One row per payment |
| fct_subscriptions | subscription_id | One row per subscription |
| fct_daily_active_users | (report_date, station_id) | One row per day per station + network-wide |
| fct_daily_churn | (report_date, station_id) | One row per day per station + network-wide |

---

## Silver Layer Review Summary (2026-01-10)

### Tables Reviewed & Changes Made

| # | Table | Status | Changes |
|---|-------|--------|---------|
| 1 | dim_stations | ✅ Updated | Added zone_id, station_type, partner_name, extended_data placeholders |
| 2 | dim_customers | ✅ Updated | Added customer_segment (auto-derived from source), contact_id, extended_data |
| 3 | dim_devices | ✅ Updated | Added maintenance_status, last_maintenance_at, extended_data; clarified these are wash machines (not IoT) |
| 4 | dim_vehicles | ✅ Redesigned | Removed all aggregations (plate counts, fraud flags); added vehicle_class, vehicle_type, license_plate_color |
| 5 | dim_vouchers | ✅ Fixed | Removed is_used, is_expired (calculations → gold layer) |
| 6 | dim_membership_plans | ✅ No changes | Already pure facts |
| 7 | dim_wash_modes | ✅ No changes | Already pure facts |
| 8 | dim_geo_hierarchy | ✅ Renamed | → ref_geo_vietnam (it's reference data, not dimension) |
| 9 | dim_payment_profiles | ✅ Created | NEW - normalized from subscription_payments.account_info JSONB |
| 10 | fct_orders | ✅ Merged | Merged fct_washes into fct_orders; removed lat/long, aggregations |
| 11 | fct_washes | ❌ Deleted | Merged into fct_orders |
| 12 | fct_transactions | ✅ Fixed | Renamed transaction_source → transaction_type; removed is_successful; added payment_profile_id link |
| 13 | fct_subscriptions | ✅ Fixed | Removed payment_count, total_paid, derived_status (aggregations → gold layer) |

### Key Decisions Made

1. **Order = Wash** - One order equals one wash, so fct_orders and fct_washes merged into single table

2. **Vehicles tracked independently** - Same license plate can have multiple vehicle_id records (anonymous wash, subscription link, different users over time)

3. **Payment profiles normalized** - Created dim_payment_profiles from JSONB to enable faster gold layer analysis

4. **Status values aligned** - Verified order_transactions and subscription_payments share same status values (succeeded, failed, pending)

5. **ref_ vs dim_** - Vietnam admin hierarchy is reference data (ref_geo_vietnam), not a dimension we're tracking

### Files Created

| File | Purpose |
|------|---------|
| transform/seeds/ref_station_zones.csv | Zone hierarchy seed data |
| transform/seeds/schema.yml | Seed schema definitions |
| landing/schemas/bronze_iot_devices.json | IoT devices landing schema |
| landing/schemas/station_zone_mapping.json | Station-zone mapping schema |
| landing/templates/bronze_iot_devices.csv | Empty CSV template |
| landing/templates/station_zone_mapping.csv | Empty CSV template |
| transform/models/silver/dim_payment_profiles.sql | Payment profiles dimension |
| transform/models/silver/ref_geo_vietnam.sql | Renamed from dim_geo_hierarchy |

### Files Deleted

| File | Reason |
|------|--------|
| transform/models/silver/fct_washes.sql | Merged into fct_orders |

### Final Table Count

- **Reference tables (ref_):** 2
- **Dimension tables (dim_):** 8
- **Fact tables (fct_):** 3
- **Total:** 13 silver layer tables

---

## Next: Tools to Build

1. **Landing zone uploader** - CLI to upload CSV to landing zone tables
2. **ref_ table editor** - CLI to add/edit/delete reference table entries
3. **Null field checker** - CLI to report empty required fields in silver layer

---

## Gold Layer Review

### Accounting Logic (Critical!)

| Concept | Definition | Includes | Excludes |
|---------|------------|----------|----------|
| **Cash Inflow** | Money deposited to bank | qrpay, credit (all order types), subscription payments | jetx_cash (prepaid) |
| **Revenue** | Service delivered | All wash payments (qrpay, credit, jetx_cash) | Top-ups (liability until used) |
| **Prepaid Usage** | Wallet redemption | jetx_cash for washes | - |
| **Top-up** | Wallet deposit | - | NOT revenue (liability) |

Key insight:
- **jetx_cash IS revenue** (service delivered) but NOT cash inflow (prepaid)
- **Top-up IS cash inflow** but NOT revenue (liability until used)

---

### 1. gold_daily_revenue ✅ Rebuilt

**Purpose:** Daily revenue & cash metrics by station

**Key fixes:**
- Source changed from deleted `fct_washes` → `fct_orders` + `fct_transactions`
- Revenue from `fct_transactions` (succeeded payments), not order amounts
- Separated revenue vs cash_inflow correctly
- Top-ups included in cash_inflow, excluded from revenue
- jetx_cash included in revenue, excluded from cash_inflow

**Columns:**
| Column | Description |
|--------|-------------|
| revenue | Wash payments (qrpay + credit + jetx_cash) |
| cash_inflow | Real money in (qrpay + credit for all orders) |
| topup_cash_inflow | Subset: wallet deposits |
| prepaid_usage | jetx_cash wash payments |

**Note:** Subscription cash inflow tracked separately in gold_subscription_mrr.

---

### Gold layer models to review:
1. ~~gold_daily_revenue~~ 🔄 In Progress (needs attribution fix)
2. gold_station_performance
3. gold_subscription_mrr
4. gold_customer_segments
5. gold_hourly_usage

---

## Immutable Station Attribution Design

### Problem
When user's home_station changes (quarterly recalc), historical data attribution should NOT change.

### Solution: Freeze Attribution in Bronze

| Event Type | Attribution Source | Frozen In |
|------------|-------------------|-----------|
| Wash orders | order_item_details.station_id | Bronze (already exists) ✅ |
| Top-up orders | user.home_station_id at ingestion | Bronze (need to add) |
| Silver subscriptions | subscription_locations.station_id | Bronze (need to ingest) |
| Gold subscriptions | user.home_station_id at ingestion | Bronze (need to add) |

### How It Works

```
Bronze Layer (append-only, immutable)
├── orders: attributed_station_id captured at ingestion
├── subscription_locations: station_id for Silver subs
└── subscriptions: attributed_station_id captured at ingestion (for Gold)

Silver Layer (recalculates, but uses frozen bronze values)
├── dim_customers: home_station_id CAN change (quarterly recalc)
├── fct_orders: attributed_station_id FROM BRONZE (frozen)
└── fct_subscriptions: attributed_station_id FROM BRONZE (frozen)

Gold Layer (aggregates using frozen attribution)
└── gold_daily_revenue: uses fct_*.attributed_station_id (never changes)
```

### Implementation Steps

1. **Ingest subscription_locations** → bronze_subscriptions.subscription_locations
2. **Add attributed_station_id to bronze ingestion** for orders and subscriptions
3. **Update fct_orders** to use attributed_station_id from bronze
4. **Update fct_subscriptions** to use attributed_station_id from bronze
5. **Rebuild gold_daily_revenue** with proper attribution

### Plan Type Identification

| Plan Type | Condition | Station Attribution |
|-----------|-----------|---------------------|
| Silver (Bạc) | max_locations = 1 | subscription_locations.station_id |
| Gold (Vàng) | max_locations = 0 or -1 | user.home_station_id |

---

## Active Users & Churn Tracking (2026-01-14)

### 4. fct_daily_active_users (Silver - NEW)

**Purpose:** Track rolling active user counts daily for trend analysis and gold layer reports.

**Grain:** 1 row per day per station + 1 row per day for network-wide (station_id = NULL)

**Columns:**
| Column | Type | Description |
|--------|------|-------------|
| report_date | DATE | The snapshot date |
| station_id | UUID | Station (NULL for network-wide totals) |
| a30 | INT | Distinct users who washed in last 30 days |
| a90 | INT | Distinct users who washed in last 90 days |
| s30 | INT | Distinct active subscribers who washed in last 30 days |
| s90 | INT | Distinct active subscribers who washed in last 90 days |
| non_sub_a30 | INT | a30 - s30 (non-subscriber active users) |
| non_sub_a90 | INT | a90 - s90 |

**Definitions:**
- **A30/A90:** Rolling 30/90-day window of distinct users with at least one wash
- **S30/S90:** Subset of A30/A90 who had an active subscription at time of wash
- **Station attribution:** Station where the user washed (for station-level breakdown)

**Use Cases:**
- Track user growth/decline trends
- Compare subscriber vs non-subscriber activity
- Station-level user engagement

**Status:** ✅ Complete

---

### 5. fct_daily_churn (Silver - NEW)

**Purpose:** Track user and subscriber churn/reactivation daily.

**Grain:** 1 row per day per station + 1 row per day for network-wide (station_id = NULL)

**Columns:**
| Column | Type | Description |
|--------|------|-------------|
| report_date | DATE | The snapshot date |
| station_id | UUID | Station (NULL for network-wide totals) |
| user_churned_count | INT | Users in A90 before, no wash in 90 days |
| user_reactivated_1_count | INT | Churned users who came back within 90 days |
| user_reactivated_2_count | INT | Churned users who came back after 90+ days |
| sub_at_risk_count | INT | Had subscription, no active sub, within 90 days |
| sub_churned_count | INT | Had subscription, no active sub, no wash 90+ days |
| sub_reactivated_1_count | INT | At-risk/churned subscribers, came back within 90 days |
| sub_reactivated_2_count | INT | Churned subscribers 90+ days, then came back |

**User Churn Definitions (90-day window):**
| Status | Definition |
|--------|------------|
| churned | Was in A90 before, but no wash in last 90 days |
| reactivated_1 | Was churned, came back within 90 days of churning |
| reactivated_2 | Was churned 90+ days, then came back |

**Subscriber Churn Definitions:**
| Status | Definition |
|--------|------------|
| at_risk | Had subscription, now no active sub, but within 90 days (might come back) |
| churned | Had subscription, no active sub, AND no wash for 90+ days |
| reactivated_1 | Was at-risk/churned, came back within 90 days |
| reactivated_2 | Was churned 90+ days, then came back |

**Station Attribution:** User's last wash station

**Use Cases:**
- Churn rate dashboards
- Reactivation campaign effectiveness
- Station-level retention analysis

**Status:** ✅ Complete

---

### Change Log (cont.)

| Date | Change | Files Modified |
|------|--------|----------------|
| 2026-01-14 | Created fct_daily_active_users (A30/A90/S30/S90 metrics) | transform/models/silver/fct_daily_active_users.sql |
| 2026-01-14 | Created fct_daily_churn (user + subscriber churn tracking) | transform/models/silver/fct_daily_churn.sql |
| 2026-01-14 | Updated fact table inventory | docs/SILVER_LAYER_DESIGN_LOG.md |

