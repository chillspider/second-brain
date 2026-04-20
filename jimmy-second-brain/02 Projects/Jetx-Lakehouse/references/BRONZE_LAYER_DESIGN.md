# JetX Lakehouse - Bronze Layer Design

> Version: 1.0
> Last Updated: 2026-01-10
>
> This document defines the domain-based bronze layer structure for the JetX Lakehouse.

---

## Design Principles

1. **Domain-based organization** - Tables grouped by business function, not source database
2. **Bronze = raw + normalized** - JSONB columns flattened at bronze layer
3. **Single source of truth** - Merged entities where data overlaps (e.g., users)
4. **Full lineage** - Track source database and extraction timestamp
5. **Soft deletes preserved** - Keep deleted_at for historical analysis

---

## Directory Structure

```
//lakehouse/jetx/bronze/
├── operations/                 # Wash operations & transactions
│   ├── orders/
│   ├── order_items/
│   ├── order_discounts/        # Extracted from orders.discounts[]
│   ├── order_transactions/
│   ├── stations/
│   ├── devices/
│   ├── device_logs/
│   └── car_detectors/
│
├── subscriptions/              # Membership & recurring revenue
│   ├── subscriptions/
│   ├── subscription_modes/     # Extracted from subscriptions.modes[]
│   ├── subscription_payments/
│   ├── subscription_vehicles/
│   └── membership_plans/
│
├── vouchers/                   # Voucher/coupon system
│   ├── vouchers/               # From wash24h_prod
│   ├── voucher_orders/         # From voucher_prod.orders
│   ├── voucher_order_items/    # From voucher_prod.order_items
│   ├── voucher_transactions/   # From voucher_prod.order_transactions
│   └── package_vouchers/
│
├── customers/                  # Customer & user data
│   ├── users/                  # Merged from both sources
│   ├── user_roles/
│   └── referrals/
│
├── vehicles/                   # Vehicle registry
│   └── vehicles/
│
└── reference/                  # Shared reference data
    ├── cities/
    ├── districts/
    ├── wards/
    ├── modes/
    └── geo_hierarchy/          # Custom: zone/region mapping
```

---

## Source Mapping

| Domain | Bronze Table | Source DB | Source Table | Notes |
|--------|--------------|-----------|--------------|-------|
| operations | orders | wash24h_prod | orders | Main wash orders |
| operations | order_items | wash24h_prod | order_items | |
| operations | order_discounts | wash24h_prod | orders.discounts[] | JSONB extract |
| operations | order_transactions | wash24h_prod | order_transactions | |
| operations | stations | wash24h_prod | stations | |
| operations | devices | wash24h_prod | devices | |
| operations | device_logs | wash24h_prod | device_logs | |
| operations | car_detectors | wash24h_prod | car_detectors | |
| subscriptions | subscriptions | wash24h_prod | subscriptions | |
| subscriptions | subscription_modes | wash24h_prod | subscriptions.modes[] | JSONB extract |
| subscriptions | subscription_payments | wash24h_prod | subscription_payments | |
| subscriptions | subscription_vehicles | wash24h_prod | subscription_vehicles | |
| subscriptions | membership_plans | wash24h_prod | membership_plans | |
| vouchers | vouchers | wash24h_prod | vouchers | |
| vouchers | voucher_orders | voucher_prod | orders | Prefixed to distinguish |
| vouchers | voucher_order_items | voucher_prod | order_items | |
| vouchers | voucher_transactions | voucher_prod | order_transactions | |
| vouchers | package_vouchers | wash24h_prod | package_vouchers | |
| customers | users | BOTH | users | Merged, deduplicated |
| customers | user_roles | wash24h_prod | user_roles | |
| customers | referrals | wash24h_prod | referrals | |
| vehicles | vehicles | wash24h_prod | vehicles | |
| reference | cities | wash24h_prod | cities | |
| reference | districts | wash24h_prod | districts | |
| reference | wards | wash24h_prod | wards | |
| reference | modes | wash24h_prod | modes | |
| reference | geo_hierarchy | CUSTOM | N/A | Manual mapping |

---

## Table Schemas

### Operations Domain

#### operations.orders

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| increment_id | INT | increment_id | Display ID |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | Soft delete |
| customer_id | UUID | customer_id | FK → users |
| customer_name | STRING | customer_name | Denormalized |
| customer_phone | STRING | customer_phone | Denormalized |
| customer_email | STRING | customer_email | Denormalized |
| station_id | UUID | data→station_id | Extracted from JSONB |
| device_id | UUID | data→device_id | Extracted from JSONB |
| vehicle_id | UUID | data→vehicle_id | Extracted from JSONB |
| license_plate | STRING | data→license_plate | Extracted from JSONB |
| sub_total | BIGINT | sub_total | In VND |
| discount_amount | BIGINT | discount_amount | |
| grand_total | BIGINT | grand_total | |
| tax_amount | BIGINT | tax_amount | |
| extra_fee | BIGINT | extra_fee | |
| membership_amount | BIGINT | membership_amount | |
| item_quantity | INT | item_quantity | |
| status | STRING | status | |
| type | STRING | type | |
| payment_method | STRING | payment_method | |
| payment_provider | STRING | payment_provider | |
| note | STRING | note | |
| failure_reason | STRING | failure_reason | |
| parent_id | UUID | parent_id | For linked orders |
| nflow_id | STRING | nflow_id | External ref |
| ygl_order_id | STRING | ygl_order_id | External ref |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### operations.order_items

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| order_id | UUID | order_id | FK → orders |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| product_id | UUID | product_id | |
| product_name | STRING | name | |
| mode_id | UUID | data→mode_id | Extracted from JSONB |
| mode_name | STRING | data→mode_name | Extracted from JSONB |
| quantity | INT | quantity | |
| unit_price | BIGINT | price | |
| original_price | BIGINT | original_price | |
| discount_amount | BIGINT | discount_amount | |
| total_price | BIGINT | total | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### operations.order_discounts

Extracted from `orders.discounts[]` JSONB array.

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | generated | PK (order_id + index) |
| order_id | UUID | parent order id | FK → orders |
| discount_index | INT | array index | Position in array |
| voucher_id | UUID | discounts[].id | |
| voucher_name | STRING | discounts[].name | |
| voucher_code | STRING | discounts[].code | |
| discount_type | STRING | discounts[].type | percentage/fixed |
| discount_value | BIGINT | discounts[].value | |
| discount_amount | BIGINT | discounts[].amount | Actual deduction |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### operations.order_transactions

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| order_id | UUID | order_id | FK → orders |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| amount | BIGINT | amount | In VND |
| payment_method | STRING | payment_method | |
| payment_provider | STRING | payment_provider | |
| status | STRING | status | |
| transaction_id | STRING | transaction_id | External ref |
| failure_reason | STRING | failure_reason | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### operations.stations

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| name | STRING | name | |
| code | STRING | code | Short code |
| address | STRING | address | |
| city_id | UUID | city_id | FK → cities |
| district_id | UUID | district_id | FK → districts |
| ward_id | UUID | ward_id | FK → wards |
| latitude | DECIMAL | latitude | |
| longitude | DECIMAL | longitude | |
| phone | STRING | phone | |
| status | STRING | status | |
| opening_time | STRING | opening_time | |
| closing_time | STRING | closing_time | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### operations.devices

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| station_id | UUID | station_id | FK → stations |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| name | STRING | name | |
| code | STRING | code | |
| type | STRING | type | |
| status | STRING | status | |
| ip_address | STRING | ip_address | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### operations.car_detectors

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| station_id | UUID | station_id | FK → stations |
| device_id | UUID | device_id | FK → devices |
| created_at | TIMESTAMP | created_at | |
| license_plate | STRING | license_plate | |
| confidence | DECIMAL | data→confidence | Extracted from JSONB |
| image_url | STRING | image_url | |
| status | STRING | status | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

---

### Subscriptions Domain

#### subscriptions.subscriptions

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| user_id | UUID | user_id | FK → users |
| membership_plan_id | UUID | membership_plan_id | FK → membership_plans |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| status | STRING | status | active/canceled/suspended |
| auto_renew | BOOLEAN | auto_renew | |
| current_price | BIGINT | current_price | |
| original_price | BIGINT | original_price | |
| started_at | TIMESTAMP | started_at | |
| current_period_start | TIMESTAMP | current_period_start_date | |
| current_period_end | TIMESTAMP | current_period_end_date | |
| next_billing_date | TIMESTAMP | next_billing_date | |
| is_trial | BOOLEAN | is_trial | |
| trial_start_date | TIMESTAMP | trial_start_date | |
| trial_end_date | TIMESTAMP | trial_end_date | |
| trial_converted_at | TIMESTAMP | trial_converted_at | |
| canceled_at | TIMESTAMP | canceled_at | |
| cancel_reason | STRING | cancel_reason | |
| suspended_at | TIMESTAMP | suspended_at | |
| max_users | INT | max_users | |
| max_vehicles | INT | max_vehicles | |
| max_locations | INT | max_locations | |
| max_daily_usage | INT | max_daily_usage | |
| billing_cycle | STRING | billing_cycle | monthly/yearly |
| duration_in_days | INT | duration_in_days | |
| attempt | INT | attempt | Retry count |
| next_retry_at | TIMESTAMP | next_retry_at | |
| use_wallet_for_renewal | BOOLEAN | use_wallet_for_renewal | |
| note | STRING | note | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### subscriptions.subscription_modes

Extracted from `subscriptions.modes[]` JSONB array.

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | generated | PK (subscription_id + mode_id) |
| subscription_id | UUID | parent subscription id | FK → subscriptions |
| mode_id | UUID | modes[].id | FK → modes |
| mode_name | STRING | modes[].name | |
| included_washes | INT | modes[].quantity | Per billing cycle |
| used_washes | INT | modes[].used | Current period usage |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### subscriptions.subscription_payments

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| subscription_id | UUID | subscription_id | FK → subscriptions |
| user_id | UUID | user_id | FK → users |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| amount | BIGINT | amount | In VND |
| status | STRING | status | |
| payment_method | STRING | payment_method | |
| payment_provider | STRING | payment_provider | |
| transaction_id | STRING | transaction_id | |
| period_start | TIMESTAMP | period_start | |
| period_end | TIMESTAMP | period_end | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### subscriptions.subscription_vehicles

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| subscription_id | UUID | subscription_id | FK → subscriptions |
| vehicle_id | UUID | vehicle_id | FK → vehicles |
| created_at | TIMESTAMP | created_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

---

### Vouchers Domain

#### vouchers.vouchers

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| name | STRING | name | |
| description | STRING | description | |
| code | STRING | code | Voucher code |
| type | STRING | type | percentage/fixed |
| voucher_model | STRING | voucher_model | |
| profile_application | STRING | profile_application | |
| min_order_value | BIGINT | min_order_value | |
| max_deduction_value | BIGINT | max_deduction_value | |
| percentage | INT | percentage | For % discounts |
| hidden_cash_value | BIGINT | hidden_cash_value | |
| start_at | TIMESTAMP | start_at | Validity start |
| end_at | TIMESTAMP | end_at | Validity end |
| status | STRING | status | |
| issue_type | STRING | issue_type | |
| user_id | UUID | user_id | Assigned user |
| order_id | UUID | order_id | Redeemed order |
| station_ids | ARRAY<UUID> | location→station_ids | Extracted from JSONB |
| valid_days_of_week | ARRAY<INT> | validity→days | Extracted from JSONB |
| valid_start_time | STRING | validity→start_time | Extracted from JSONB |
| valid_end_time | STRING | validity→end_time | Extracted from JSONB |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### vouchers.voucher_orders

From `voucher_prod.orders` - voucher redemption orders.

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| increment_id | INT | increment_id | |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| customer_id | UUID | customer_id | FK → users |
| customer_name | STRING | customer_name | |
| customer_phone | STRING | customer_phone | |
| station_id | UUID | data→station_id | Extracted from JSONB |
| device_id | UUID | data→device_id | Extracted from JSONB |
| sub_total | BIGINT | sub_total | |
| discount_amount | BIGINT | discount_amount | |
| grand_total | BIGINT | grand_total | |
| status | STRING | status | |
| type | STRING | type | |
| payment_method | STRING | payment_method | |
| _source_db | STRING | N/A | 'voucher_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### vouchers.voucher_order_items

From `voucher_prod.order_items`.

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| order_id | UUID | order_id | FK → voucher_orders |
| created_at | TIMESTAMP | created_at | |
| voucher_id | UUID | voucher_id | FK → vouchers |
| voucher_name | STRING | name | |
| quantity | INT | quantity | |
| unit_price | BIGINT | price | |
| total_price | BIGINT | total | |
| _source_db | STRING | N/A | 'voucher_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### vouchers.voucher_transactions

From `voucher_prod.order_transactions`.

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| order_id | UUID | order_id | FK → voucher_orders |
| created_at | TIMESTAMP | created_at | |
| amount | BIGINT | amount | |
| status | STRING | status | |
| payment_method | STRING | payment_method | |
| transaction_id | STRING | transaction_id | |
| _source_db | STRING | N/A | 'voucher_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### vouchers.package_vouchers

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| name | STRING | name | |
| description | STRING | description | |
| price | BIGINT | price | |
| original_price | BIGINT | original_price | |
| voucher_count | INT | voucher→count | Extracted from JSONB |
| voucher_value | BIGINT | voucher→value | Extracted from JSONB |
| station_ids | ARRAY<UUID> | station_ids | |
| status | STRING | status | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

---

### Customers Domain

#### customers.users

Merged from both `wash24h_prod.users` and `voucher_prod.users`.

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK - same across both DBs |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | Latest from either |
| deleted_at | TIMESTAMP | deleted_at | |
| first_name | STRING | first_name | |
| last_name | STRING | last_name | |
| email | STRING | email | |
| phone | STRING | phone | |
| avatar | STRING | avatar | |
| status | STRING | status | |
| type | STRING | type | customer/staff/admin |
| provider | STRING | provider | Auth provider |
| social_id | STRING | social_id | |
| referral_code | STRING | referral_code | |
| is_phone_verified | BOOLEAN | is_phone_verified | |
| is_email_verified | BOOLEAN | is_email_verified | |
| address | STRING | address | |
| city | STRING | city | |
| country | STRING | country | |
| note | STRING | note | |
| station_id | UUID | station_id | For staff users |
| in_wash24h | BOOLEAN | computed | Exists in wash24h_prod |
| in_voucher | BOOLEAN | computed | Exists in voucher_prod |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### customers.user_roles

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| user_id | UUID | user_id | FK → users |
| role_id | UUID | role_id | FK → roles |
| created_at | TIMESTAMP | created_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

---

### Vehicles Domain

#### vehicles.vehicles

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| user_id | UUID | user_id | FK → users |
| created_at | TIMESTAMP | created_at | |
| updated_at | TIMESTAMP | updated_at | |
| deleted_at | TIMESTAMP | deleted_at | |
| license_plate | STRING | license_plate | Unique |
| brand | STRING | brand | |
| model | STRING | model | |
| color | STRING | color | |
| type | STRING | type | car/motorcycle |
| is_default | BOOLEAN | is_default | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

---

### Reference Domain

#### reference.cities

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| name | STRING | name | Vietnamese name |
| name_en | STRING | name_en | English name |
| code | STRING | code | Province code |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### reference.districts

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| city_id | UUID | city_id | FK → cities |
| name | STRING | name | |
| name_en | STRING | name_en | |
| code | STRING | code | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### reference.wards

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| district_id | UUID | district_id | FK → districts |
| name | STRING | name | |
| name_en | STRING | name_en | |
| code | STRING | code | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### reference.modes

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| id | UUID | id | PK |
| created_at | TIMESTAMP | created_at | |
| name | STRING | name | |
| description | STRING | description | |
| duration_minutes | INT | duration | |
| price | BIGINT | price | |
| status | STRING | status | |
| _source_db | STRING | N/A | 'wash24h_prod' |
| _extracted_at | TIMESTAMP | N/A | ETL timestamp |

#### reference.geo_hierarchy

Custom table for geographic drill-down capability.

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| station_id | UUID | manual | FK → stations |
| station_name | STRING | manual | |
| zone_id | STRING | manual | Custom zone code |
| zone_name | STRING | manual | e.g., "HCM East" |
| region_id | STRING | manual | Custom region code |
| region_name | STRING | manual | e.g., "HCM Metro" |
| city_id | UUID | stations.city_id | FK → cities |
| city_name | STRING | cities.name | |
| country_code | STRING | manual | 'VN' |
| country_name | STRING | manual | 'Vietnam' |
| _updated_at | TIMESTAMP | manual | Last mapping update |

---

## Geographic Hierarchy Mapping

```
Country: Vietnam (VN)
├── Region: HCM Metro (HCM)
│   ├── Zone: HCM Central (HCM-C)
│   │   ├── Station: JetX Quận 1
│   │   └── Station: JetX Quận 3
│   ├── Zone: HCM East (HCM-E)
│   │   ├── Station: JetX Vạn Phúc City
│   │   └── Station: JetX Thủ Đức
│   └── Zone: HCM West (HCM-W)
│       └── Station: JetX Bình Chánh
│
├── Region: Hanoi Metro (HN)
│   ├── Zone: HN Central (HN-C)
│   │   └── Station: JetX Linh Đàm
│   └── Zone: HN West (HN-W)
│       └── Station: JetX Đan Phượng
│
├── Region: Northern (NORTH)
│   └── Zone: Bắc Giang (BG)
│       └── Station: JetX Bắc Giang
│
├── Region: Central (CENTRAL)
│   └── Zone: Đà Nẵng (DN)
│       └── Station: JetX Đà Nẵng
│
└── Region: Mekong (MEKONG)
    └── Zone: Cần Thơ (CT)
        └── Station: JetX Cần Thơ
```

*Note: Actual station mapping to be populated based on current station list.*

---

## JSONB Normalization Summary

| Source Table | JSONB Column | Bronze Table(s) |
|--------------|--------------|-----------------|
| orders.discounts | Array of applied discounts | order_discounts |
| orders.data | Order metadata | orders (flattened) |
| orders.membership | Membership info at order time | orders (flattened) |
| order_items.data | Item details | order_items (flattened) |
| subscriptions.modes | Included wash modes | subscription_modes |
| vouchers.location | Station restrictions | vouchers (flattened) |
| vouchers.validity | Time restrictions | vouchers (flattened) |
| package_vouchers.voucher | Voucher details | package_vouchers (flattened) |
| car_detectors.data | Detection metadata | car_detectors (flattened) |

---

## Partitioning Strategy

| Table | Partition Key | Strategy |
|-------|---------------|----------|
| orders | created_at | Monthly |
| order_items | created_at | Monthly |
| order_transactions | created_at | Monthly |
| voucher_orders | created_at | Monthly |
| device_logs | created_at | Daily |
| car_detectors | created_at | Daily |
| subscription_payments | created_at | Monthly |
| All others | None | Unpartitioned |

---

## Iceberg Table Properties

All bronze tables will use these common properties:

```python
{
    "format-version": "2",
    "write.format.default": "parquet",
    "write.parquet.compression-codec": "zstd",
    "write.metadata.compression-codec": "gzip",
    "write.target-file-size-bytes": "134217728",  # 128 MB
    "write.metadata.delete-after-commit.enabled": "true",
    "write.metadata.previous-versions-max": "100",
    "history.expire.max-snapshot-age-ms": "604800000",  # 7 days
}
```

---

## Implementation Priority

### Phase 1: Core Operations
1. `reference.modes`
2. `reference.cities`, `reference.districts`, `reference.wards`
3. `operations.stations`
4. `operations.devices`
5. `customers.users`

### Phase 2: Transactions
6. `vehicles.vehicles`
7. `operations.orders`
8. `operations.order_items`
9. `operations.order_discounts`
10. `operations.order_transactions`

### Phase 3: Subscriptions
11. `subscriptions.membership_plans`
12. `subscriptions.subscriptions`
13. `subscriptions.subscription_modes`
14. `subscriptions.subscription_payments`
15. `subscriptions.subscription_vehicles`

### Phase 4: Vouchers
16. `vouchers.vouchers`
17. `vouchers.package_vouchers`
18. `vouchers.voucher_orders`
19. `vouchers.voucher_order_items`
20. `vouchers.voucher_transactions`

### Phase 5: Analytics
21. `operations.car_detectors`
22. `operations.device_logs`
23. `reference.geo_hierarchy`

---

## Next Steps

1. **Create `reference.geo_hierarchy` mapping** - Define zone/region assignments for each station
2. **Build first ingestion pipeline** - Start with `reference.modes` and `operations.stations`
3. **Test JSONB extraction** - Validate data extraction from complex JSONB columns
4. **Define incremental strategy** - CDC vs. full refresh per table
