# JetX Lakehouse - Source Database Schema Analysis

> Generated: 2026-01-10 10:24:17
> 
> This document provides a comprehensive analysis of the source databases for the JetX Lakehouse project.

---

## Executive Summary

| Database | Tables | Total Size | Key Tables |
|----------|--------|------------|------------|
| **wash24h_prod** | 63 | 2.5 GB | orders, users, subscriptions, vouchers |
| **voucher_prod** | 47 | 1.1 GB | vouchers, orders, users |

**Shared Customers:** Yes - users exist in both systems with same IDs

---

## Database 1: wash24h_prod (Primary)

### Overview
- **Purpose:** Main car wash application database
- **Tables:** 63
- **Total Size:** 2.5 GB

### Tables by Size

| Table | Size | Est. Rows | Description |
|-------|------|-----------|-------------|
| sync_logs | 628 MB | 404,469 | |
| device_logs | 615 MB | 647,500 | |
| orders | 175 MB | 158,863 | |
| notifications | 144 MB | 239,540 | |
| audit_logs | 143 MB | 87,839 | |
| order_transaction_logs | 131 MB | 112,624 | |
| order_items | 130 MB | 119,378 | |
| invoices | 118 MB | 81,723 | |
| activity_logs | 109 MB | 101,032 | |
| car_detectors | 85 MB | 70,145 | |
| user_notifications | 81 MB | 312,463 | |
| order_transactions | 57 MB | 87,707 | |
| vouchers | 41 MB | 32,469 | |
| invoice_items | 22 MB | 84,450 | |
| users | 21 MB | 43,986 | |
| invoice_billings | 20 MB | 84,567 | |
| user_roles | 9304 kB | 39,523 | |
| subscription_payments | 8744 kB | 3,176 | |
| sms_activity_logs | 8624 kB | 32,567 | |
| webhook_logs | 7344 kB | 4,266 | |
| subscription_logs | 5160 kB | 2,509 | |
| package_vouchers | 4192 kB | 11,658 | |
| wards | 1552 kB | 10,547 | |
| supports | 1456 kB | 1,562 | |
| user_tokens | 1288 kB | 3,215 | |

### Table Categories

**Core Tables (20):** orders, order_items, invoices, car_detectors, order_transactions, vouchers, invoice_items, users, invoice_billings, subscription_payments, package_vouchers, supports, user_tokens, subscription_vehicles, subscriptions

**Log Tables (11):** sync_logs, device_logs, notifications, audit_logs, order_transaction_logs, activity_logs, user_notifications, sms_activity_logs, webhook_logs, subscription_logs, notification_campaigns

**Reference Tables (11):** user_roles, wards, wards_v2, districts, roles, settings, products, role_permissions, station_modes, modes, migrations


### JSONB Columns Requiring Normalization (55)

| Table | Column | Est. Rows |
|-------|--------|-----------|
| sync_logs | value | 404,469 |
| notifications | data | 239,540 |
| orders | discount_ids | 158,863 |
| orders | data | 158,863 |
| orders | discounts | 158,863 |
| orders | membership | 158,863 |
| order_items | discount_ids | 119,378 |
| order_items | data | 119,378 |
| activity_logs | value | 101,032 |
| audit_logs | value | 87,839 |
| invoices | data | 81,723 |
| car_detectors | data | 70,145 |
| users | device_tokens | 43,986 |
| vouchers | location | 32,469 |
| vouchers | validity | 32,469 |
| vouchers | data | 32,469 |
| package_vouchers | station_ids | 11,658 |
| package_vouchers | voucher | 11,658 |
| webhook_logs | payload | 4,266 |
| webhook_logs | headers | 4,266 |


---

## Database 2: voucher_prod (Secondary)

### Overview
- **Purpose:** Voucher/coupon management system
- **Tables:** 47
- **Total Size:** 1.1 GB

### Tables by Size

| Table | Size | Est. Rows | Description |
|-------|------|-----------|-------------|
| device_logs | 361 MB | 444,198 | |
| sync_logs | 266 MB | 176,725 | |
| activity_logs | 99 MB | 94,550 | |
| vouchers | 69 MB | 49,115 | |
| order_transaction_logs | 53 MB | 60,152 | |
| orders | 53 MB | 57,877 | |
| order_items | 43 MB | 47,795 | |
| invoices | 42 MB | 34,561 | |
| notifications | 41 MB | 82,032 | |
| user_notifications | 16 MB | 84,672 | |
| order_transactions | 13 MB | 32,501 | |
| users | 11 MB | 22,167 | |
| car_detectors | 9672 kB | 17,096 | |
| invoice_items | 7208 kB | 35,407 | |
| invoice_billings | 6624 kB | 34,561 | |
| package_vouchers | 4192 kB | 11,657 | |
| user_roles | 4040 kB | 22,170 | |
| wards | 1552 kB | 10,547 | |
| sms_activity_logs | 760 kB | 4,474 | |
| supports | 576 kB | 513 | |

### Table Categories

**Core Tables (14):** vouchers, orders, order_items, invoices, order_transactions, users, car_detectors, invoice_items, invoice_billings, package_vouchers, vehicles, user_tokens, stations, devices


### JSONB Columns Requiring Normalization (31)

| Table | Column | Est. Rows |
|-------|--------|-----------|
| sync_logs | value | 176,725 |
| activity_logs | value | 94,550 |
| notifications | data | 82,032 |
| orders | discount_ids | 57,877 |
| orders | data | 57,877 |
| orders | discounts | 57,877 |
| orders | membership | 57,877 |
| vouchers | location | 49,115 |
| vouchers | validity | 49,115 |
| vouchers | data | 49,115 |
| order_items | discount_ids | 47,795 |
| order_items | data | 47,795 |
| invoices | data | 34,561 |
| users | device_tokens | 22,167 |
| car_detectors | data | 17,096 |


---

## Key Table Schemas

### orders (wash24h_prod)

**Rows:** 158,863 | **Size:** 175 MB

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | uuid | NO | Primary key |
| created_at | timestamp | NO |  |
| created_by | uuid | YES |  |
| updated_at | timestamp | NO |  |
| updated_by | uuid | YES |  |
| deleted_at | timestamp | YES |  |
| deleted_by | uuid | YES |  |
| increment_id | int4 | NO | Foreign key |
| customer_id | varchar | YES | Foreign key |
| customer_name | varchar | YES |  |
| customer_email | varchar | YES |  |
| customer_phone | varchar | YES |  |
| note | varchar | YES |  |
| sub_total | int8 | YES |  |
| grand_total | int8 | NO |  |
| item_quantity | int4 | NO |  |
| discount_amount | int8 | NO |  |
| tax_amount | int8 | YES |  |
| status | varchar | NO |  |
| discount_ids | jsonb | NO | **JSONB - needs normalization** |
| payment_method | varchar | NO |  |
| payment_provider | varchar | YES |  |
| data | jsonb | YES | **JSONB - needs normalization** |
| discounts | jsonb | NO | **JSONB - needs normalization** |
| membership | jsonb | YES | **JSONB - needs normalization** |
| membership_amount | int8 | YES |  |
| nflow_id | varchar | YES | Foreign key |
| type | varchar | NO |  |
| extra_fee | int8 | YES |  |
| parent_id | uuid | YES | Foreign key |
| ygl_order_id | varchar | YES | Foreign key |
| failure_reason | text | YES |  |


### users (wash24h_prod)

**Rows:** 43,986 | **Size:** 21 MB

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | uuid | NO | Primary key |
| created_at | timestamp | NO |  |
| created_by | uuid | YES |  |
| updated_at | timestamp | NO |  |
| updated_by | uuid | YES |  |
| deleted_at | timestamp | YES |  |
| deleted_by | uuid | YES |  |
| first_name | varchar | YES |  |
| last_name | varchar | YES |  |
| email | varchar | YES |  |
| password | varchar | YES |  |
| phone | varchar | YES |  |
| avatar | varchar | YES |  |
| provider | varchar | YES |  |
| social_id | varchar | YES |  |
| status | varchar | NO |  |
| type | varchar | NO |  |
| device_tokens | jsonb | YES | **JSONB - needs normalization** |
| note | varchar | YES |  |
| nflow_id | varchar | YES |  |
| referral_code | varchar | YES |  |
| station_id | varchar | YES |  |
| kiosk_id | varchar | YES |  |
| is_phone_verified | bool | NO |  |
| is_email_verified | bool | NO |  |
| address | varchar | YES |  |
| city | varchar | YES |  |
| country | varchar | YES |  |


### subscriptions (wash24h_prod)

**Rows:** 2,155 | **Size:** 992 kB

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | uuid | NO | Primary key |
| created_at | timestamp | NO |  |
| created_by | uuid | YES |  |
| updated_at | timestamp | NO |  |
| updated_by | uuid | YES |  |
| deleted_at | timestamp | YES |  |
| deleted_by | uuid | YES |  |
| membership_plan_id | uuid | NO | Foreign key |
| user_id | uuid | NO | Foreign key |
| user_phone | varchar | YES |  |
| user_token_id | uuid | YES | Foreign key |
| status | varchar | NO |  |
| auto_renew | bool | NO |  |
| current_price | int8 | NO |  |
| original_price | int8 | NO |  |
| started_at | timestamptz | NO |  |
| current_period_start_date | timestamptz | NO |  |
| current_period_end_date | timestamptz | NO |  |
| next_billing_date | timestamptz | NO |  |
| is_trial | bool | NO |  |
| trial_start_date | timestamptz | YES |  |
| trial_end_date | timestamptz | YES |  |
| trial_converted_at | timestamptz | YES |  |
| canceled_at | timestamptz | YES |  |
| cancel_reason | varchar | YES |  |
| max_users | int4 | NO |  |
| max_vehicles | int4 | NO |  |
| max_locations | int4 | YES |  |
| modes | jsonb | NO | **JSONB - needs normalization** |
| billing_cycle | varchar | NO |  |
| attempt | int4 | NO |  |
| next_retry_at | timestamptz | YES |  |
| suspended_at | timestamptz | YES |  |
| max_daily_usage | int4 | NO |  |
| note | varchar | YES |  |
| use_wallet_for_renewal | bool | NO |  |
| external_subscription_id | varchar | YES | Foreign key |
| duration_in_days | int4 | YES |  |


### vouchers (wash24h_prod)

**Rows:** 32,469 | **Size:** 41 MB

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | uuid | NO | Primary key |
| created_at | timestamp | NO |  |
| created_by | uuid | YES |  |
| updated_at | timestamp | NO |  |
| updated_by | uuid | YES |  |
| deleted_at | timestamp | YES |  |
| deleted_by | uuid | YES |  |
| name | varchar | NO |  |
| description | varchar | YES |  |
| type | varchar | NO |  |
| profile_application | varchar | NO |  |
| voucher_model | varchar | NO |  |
| min_order_value | int8 | NO |  |
| max_deduction_value | int8 | YES |  |
| hidden_cash_value | int8 | NO |  |
| start_at | timestamptz | YES |  |
| end_at | timestamptz | YES |  |
| location | jsonb | YES | **JSONB - needs normalization** |
| status | varchar | NO |  |
| user_id | uuid | YES |  |
| order_id | uuid | YES |  |
| percentage | int8 | NO |  |
| nflow_id | varchar | YES |  |
| validity | jsonb | YES | **JSONB - needs normalization** |
| data | jsonb | YES | **JSONB - needs normalization** |
| issue_type | varchar | YES |  |


---

## Common Patterns

### Soft Deletes
All tables use soft delete pattern with `deleted_at` column:
- `deleted_at IS NULL` = active record
- `deleted_at IS NOT NULL` = deleted record

### Audit Columns
All tables have standard audit columns:
- `created_at` - Record creation timestamp
- `updated_at` - Last update timestamp
- `created_by` - User ID who created (nullable)
- `updated_by` - User ID who updated (nullable)
- `deleted_by` - User ID who deleted (nullable)

### ID Format
- All primary keys are `uuid` type
- Foreign keys are either `uuid` or `varchar` (some legacy)

---

## Recommended Bronze Layer Tables

### Priority 1 (Core Business)
From **wash24h_prod**:
- `orders` - 159K rows, core transaction data
- `order_items` - 119K rows, order line items
- `users` - 44K rows, customer data
- `stations` - 18 rows, wash locations
- `devices` - 18 rows, wash machines
- `subscriptions` - 2K rows, membership plans
- `order_transactions` - 88K rows, payment records

### Priority 2 (Extended)
From **wash24h_prod**:
- `vouchers` - 32K rows
- `vehicles` - 3K rows
- `subscription_payments` - 3K rows
- `car_detectors` - 70K rows (plate detection)

From **voucher_prod**:
- `vouchers` - 49K rows
- `orders` - 58K rows
- `order_items` - 48K rows
- `order_transactions` - 33K rows

### Priority 3 (Analytics/Logs)
- `device_logs` - 647K rows (operational)
- `car_detectors` - plate recognition data
- `activity_logs` - user activity tracking

---

## Data Volume Estimates

### Current State
| Metric | Value |
|--------|-------|
| Total source data | ~3.5 GB |
| wash24h_prod | ~2.5 GB |
| voucher_prod | ~1.0 GB |
| Total rows (core tables) | ~500K |
| Growth rate (orders) | ~5K/month |

### Projected (5 years)
| Metric | Value |
|--------|-------|
| Orders | ~500K |
| Total data | ~15 GB |
| Recommended partitioning | Monthly by created_at |

---

## Next Steps

1. **Approve this schema analysis**
2. **Design bronze layer** - Iceberg table schemas
3. **Design silver layer** - Unified dimensions and facts
4. **Build ingestion pipeline** - Start with stations table
