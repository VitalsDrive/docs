# VitalsDrive — Consolidated Data Model

**Version:** 1.0
**Status:** Draft
**Last Updated:** May 2026
**Document Owner:** Backend Infrastructure

---

This document consolidates all database table definitions from across the individual PRDs into a single authoritative schema reference. Each PRD remains the source of truth for its domain logic (alert rules, billing flows, etc.) — this doc is purely the unified schema for cross-referencing.

All tables are PostgreSQL with Supabase PostgreSQL. UUIDs use `gen_random_uuid()` by default unless noted.

---

## 1. Core Telemetry

### telemetry_logs

Source: `PRD-Layer2-Data-Storage.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | BIGSERIAL | PRIMARY KEY | Auto-incrementing identifier |
| vehicle_id | UUID | NOT NULL, FK → vehicles | Which vehicle this reading is from |
| lat | DECIMAL(10,7) | CHECK (-90..90) | Latitude |
| lng | DECIMAL(10,7) | CHECK (-180..180) | Longitude |
| temp | INTEGER | | Coolant temperature (°C) |
| voltage | DECIMAL(5,2) | | Battery voltage (V) |
| rpm | INTEGER | CHECK (>= 0) | Engine RPM |
| dtc_codes | TEXT[] | | Array of active DTC codes |
| gps_valid | BOOLEAN | DEFAULT true | Whether GPS signal was valid |
| direction | INTEGER | | Heading 0-360 degrees |
| gsm_signal | INTEGER | | Cellular signal strength 0-31 |
| parser_version | TEXT | | Parser version for debugging |
| device_imei | VARCHAR(20) | | Device IMEI from login packet |
| timestamp | TIMESTAMPTZ | DEFAULT NOW() | Server-side ingestion time |

---

## 2. Fleet Management

### fleets

Source: `PRD-Layer2-Data-Storage.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Fleet identifier |
| name | VARCHAR(100) | NOT NULL | Display name |
| owner_id | UUID | FK → auth.users | Fleet owner |
| org_id | UUID | FK → organizations | Parent organization |
| settings | JSONB | DEFAULT '{}' | Fleet-level settings |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Creation time |

### vehicles

Source: `PRD-Layer2-Data-Storage.md`, `PRD_CustomerManagement.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Vehicle identifier |
| fleet_id | UUID | NOT NULL, FK → fleets | Fleet the vehicle belongs to |
| account_id | UUID | FK → accounts | Parent customer account |
| vin | VARCHAR(17) | UNIQUE, NOT NULL | Vehicle Identification Number |
| make | VARCHAR(50) | NOT NULL | Manufacturer |
| model | VARCHAR(50) | NOT NULL | Model |
| year | INTEGER | | Model year |
| license_plate | VARCHAR(20) | | License plate number |
| display_name | TEXT | NOT NULL | e.g. "Van #3 - Moshe" |
| status | VARCHAR(20) | DEFAULT 'pending' | pending, active, offline, inactive, maintenance, or retired |
| metadata | JSONB | DEFAULT '{}' | Extensible attributes |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Registration time |

### fleet_members

Source: `PRD-Layer2-Data-Storage.md`, `PRD-Onboarding.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Membership identifier |
| fleet_id | UUID | FK → fleets, ON DELETE CASCADE | Fleet |
| user_id | UUID | FK → auth.users, ON DELETE CASCADE | User |
| organization_id | UUID | FK → organizations | Parent organization |
| role | VARCHAR(20) | DEFAULT 'member' | owner, admin, member, or viewer |
| joined_at | TIMESTAMPTZ | DEFAULT NOW() | Join time |

**Constraint:** UNIQUE(fleet_id, user_id)

---

## 3. Multi-Tenancy (Organizations)

### organizations

Source: `PRD-Onboarding.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Organization identifier |
| name | TEXT | NOT NULL | Display name |
| allowlisted | BOOLEAN | DEFAULT false | Whether org is approved for access |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Creation time |

### accounts

Source: `PRD_CustomerManagement.md`, `PRD_Billing.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Account identifier |
| company_name | TEXT | NOT NULL | Business name |
| email | TEXT | UNIQUE, NOT NULL | Primary billing contact |
| phone | TEXT | | Contact phone |
| plan_tier | TEXT | DEFAULT 'foundation' | foundation, diagnostic_plus, or predictive |
| status | TEXT | DEFAULT 'trial' | trial, active, suspended, or churned |
| trial_ends_at | TIMESTAMPTZ | | When trial expires |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Account creation time |

> **Note:** `organizations` and `accounts` represent the same concept at different stages. MVP uses `organizations` for auth/onboarding. `accounts` adds billing/subscription fields. These may be merged into a single table during implementation.

---

## 4. Alerts & Rules

### alerts

Source: `PRD-Layer2-Data-Storage.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | BIGSERIAL | PRIMARY KEY | Alert identifier |
| vehicle_id | UUID | NOT NULL, FK → vehicles | Affected vehicle |
| fleet_id | UUID | NOT NULL, FK → fleets | Parent fleet |
| severity | VARCHAR(10) | NOT NULL | critical, warning, or info |
| code | VARCHAR(50) | NOT NULL | Alert code (e.g. ENGINE_OVERTEMP) |
| message | TEXT | NOT NULL | Human-readable description |
| dtc_codes | TEXT[] | | Related DTC codes |
| lat, lng | DECIMAL(10,7) | | Location where triggered |
| acknowledged | BOOLEAN | DEFAULT false | Whether seen by user |
| acknowledged_by | UUID | FK → auth.users | Who acknowledged |
| acknowledged_at | TIMESTAMPTZ | | When acknowledged |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Alert creation time |
| resolved_at | TIMESTAMPTZ | | When resolved |

### telemetry_rules

Source: `PRD-Layer2-Data-Storage.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | BIGSERIAL | PRIMARY KEY | Rule identifier |
| fleet_id | UUID | NOT NULL, FK → fleets | Fleet this rule applies to |
| name | VARCHAR(100) | NOT NULL | Display name |
| metric | VARCHAR(20) | NOT NULL | temp, voltage, or rpm |
| operator | VARCHAR(5) | NOT NULL | gt, lt, gte, lte, or eq |
| threshold | FLOAT | NOT NULL | Threshold value |
| severity | VARCHAR(10) | NOT NULL | critical, warning, or info |
| enabled | BOOLEAN | DEFAULT true | Whether rule is active |
| cooldown_seconds | INTEGER | DEFAULT 300 | Min time between alerts |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Rule creation time |

---

## 5. Billing & Subscriptions

### subscriptions

Source: `PRD_Billing.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Subscription identifier |
| account_id | UUID | FK → accounts, UNIQUE | Customer account |
| stripe_customer_id | TEXT | UNIQUE | Stripe customer object |
| stripe_sub_id | TEXT | UNIQUE | Stripe subscription object |
| plan_tier | TEXT | NOT NULL | foundation, diagnostic_plus, or predictive |
| billing_cycle | TEXT | DEFAULT 'monthly' | monthly or annual |
| vehicle_count | INTEGER | NOT NULL, DEFAULT 1 | Active billable vehicles |
| status | TEXT | NOT NULL | trialing, active, past_due, or canceled |
| current_period_end | TIMESTAMPTZ | | Current billing period end |
| cancel_at_period_end | BOOLEAN | DEFAULT false | Queued cancellation |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Subscription creation time |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | Last update time |

### invoices

Source: `PRD_Billing.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Invoice identifier |
| account_id | UUID | FK → accounts | Customer account |
| stripe_invoice_id | TEXT | UNIQUE | Stripe invoice reference |
| amount_ils | INTEGER | | Amount in agorot (×100) |
| status | TEXT | | draft, open, paid, uncollectible, or void |
| period_start | TIMESTAMPTZ | | Billing period start |
| period_end | TIMESTAMPTZ | | Billing period end |
| paid_at | TIMESTAMPTZ | | When payment was received |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Invoice creation time |

---

## 6. Hardware Lifecycle

### devices

Source: `PRD_HardwareLifecycle.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Device record identifier |
| imei | TEXT | UNIQUE, NOT NULL | 15-digit hardware identifier |
| iccid | TEXT | | SIM card identifier |
| model | TEXT | | e.g. "SinoTrack ST-906" |
| firmware_ver | TEXT | | Device firmware version |
| status | TEXT | DEFAULT 'in_stock' | in_stock, provisioned, shipped, active, offline, faulty, returned, or retired |
| account_id | UUID | FK → accounts | Null until assigned |
| vehicle_id | UUID | FK → vehicles | Null until paired |
| provisioned_at | TIMESTAMPTZ | | When configured |
| shipped_at | TIMESTAMPTZ | | When shipped |
| activated_at | TIMESTAMPTZ | | First telemetry received |
| returned_at | TIMESTAMPTZ | | When returned |
| notes | TEXT | | Manual ops notes |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Record creation time |

### device_events

Source: `PRD_HardwareLifecycle.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Event identifier |
| device_id | UUID | FK → devices | Related device |
| event_type | TEXT | | provisioned, assigned, shipped, activated, went_offline, came_online, fault_detected, replacement_requested, returned, or retired |
| payload | JSONB | | Event-specific metadata |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Event time |

### shipments

Source: `PRD_HardwareLifecycle.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Shipment identifier |
| device_id | UUID | FK → devices | Shipped device |
| account_id | UUID | FK → accounts | Destination customer |
| direction | TEXT | | outbound or return |
| tracking_number | TEXT | | Carrier tracking number |
| carrier | TEXT | | e.g. "Israel Post", "DHL" |
| status | TEXT | DEFAULT 'pending' | pending, shipped, delivered, or failed |
| shipped_at | TIMESTAMPTZ | | When shipped |
| delivered_at | TIMESTAMPTZ | | When delivered |
| address_snapshot | JSONB | | Delivery address at ship time |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Record creation time |

---

## 7. Support & Tickets

### tickets

Source: `PRD_Support.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Ticket identifier |
| account_id | UUID | FK → accounts | Customer account |
| vehicle_id | UUID | FK → vehicles, nullable | Related vehicle |
| submitted_by | UUID | FK → auth.users | User who submitted |
| category | TEXT | | device_offline, missing_data, alert, dashboard, billing, or other |
| subject | TEXT | NOT NULL | Ticket subject |
| status | TEXT | DEFAULT 'open' | open, in_progress, waiting_customer, resolved, or closed |
| priority | TEXT | DEFAULT 'normal' | low, normal, high, or critical |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Ticket creation time |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | Last update time |
| resolved_at | TIMESTAMPTZ | | When resolved |

### ticket_messages

Source: `PRD_Support.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Message identifier |
| ticket_id | UUID | FK → tickets | Parent ticket |
| author_id | UUID | FK → auth.users, nullable | Null = system message |
| author_role | TEXT | | customer, support, or system |
| body | TEXT | NOT NULL | Message content |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Message time |

### ticket_diagnostics

Source: `PRD_Support.md`

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Diagnostic record identifier |
| ticket_id | UUID | FK → tickets | Related ticket |
| last_telemetry_at | TIMESTAMPTZ | | Last telemetry timestamp |
| last_voltage | FLOAT | | Last known battery voltage |
| last_temp | FLOAT | | Last known coolant temp |
| device_imei | TEXT | | Device IMEI at ticket time |
| sim_iccid | TEXT | | SIM ICCID at ticket time |
| account_status | TEXT | | Account status at ticket time |
| vehicle_status | TEXT | | Vehicle status at ticket time |
| snapshot_at | TIMESTAMPTZ | DEFAULT NOW() | When snapshot captured |

---

## 8. Relationship Diagram

```
organizations ──1:N──▶ fleets ──1:N──▶ vehicles ──1:N──▶ telemetry_logs
     │                    │               │                      │
     │                    │               │                      │ (trigger)
     │                    │               │                      ▼
     │                    │               │                  alerts
     │                    │               │
     │                    └────1:N────▶ fleet_members (M:N with auth.users)
     │
     └──1:N──▶ accounts ──1:1──▶ subscriptions
                    │                    │
                    │                    └──1:N──▶ invoices
                    │
                    ├──1:N──▶ devices ──1:N──▶ device_events
                    │              │
                    │              └──1:N──▶ shipments
                    │
                    └──1:N──▶ tickets ──1:N──▶ ticket_messages
                                                │
                                                └──1:1──▶ ticket_diagnostics
```

---

## 9. Known Inconsistencies to Resolve

| Issue | Tables | Notes |
|---|---|---|
| `organizations` vs `accounts` | Both represent the customer tenant | `organizations` has `allowlisted`, `accounts` has billing fields. Likely to be merged. |
| `vehicles.fleet_id` vs `vehicles.account_id` | Both exist in different PRDs | Layer2 uses `fleet_id`, CustomerMgmt uses `account_id`. Implementation must reconcile. |
| `users` table | Referenced in PRD-Onboarding but maps to `auth.users` in Supabase | Custom `users` table may not be needed — Supabase Auth provides the user table. |
| `fleet_members.organization_id` | Added in PRD-Onboarding but not in Layer2 PRD | Layer2 PRD doesn't have this FK. Needs alignment. |

---

*This document should be updated whenever a new column, table, or constraint is added to any PRD. Source PRDs remain authoritative for domain-specific logic.*

### Related Documents

| Document | Path |
|---|---|
| System Architecture | `system-overview.md` |
| Auth Architecture | `auth-architecture.md` |
| Main PRD | `../VitalsDrive_PRD.md` |
| Data Storage PRD | `../PRD-Layer2-Data-Storage.md` |
| Billing PRD | `../PRD_Billing.md` |
| Customer Management PRD | `../PRD_CustomerManagement.md` |
| Hardware Lifecycle PRD | `../PRD_HardwareLifecycle.md` |
| Support PRD | `../PRD_Support.md` |
| Onboarding PRD | `../PRD-Onboarding.md` |
