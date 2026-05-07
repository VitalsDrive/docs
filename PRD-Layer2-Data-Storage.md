# VitalsDrive - Data Storage Layer PRD

**Document Version:** 2.0
**Last Updated:** 2026-03-29
**Layer:** L2 - Data Storage (Supabase PostgreSQL)
**Status:** Draft for Review

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Database Schema](#2-database-schema)
3. [Supabase Realtime Configuration](#3-supabase-realtime-configuration)
4. [Row Level Security (RLS) Policies](#4-row-level-security-rls-policies)
5. [API Design](#5-api-design)
6. [Database Functions and Triggers](#6-database-functions-and-triggers)
7. [Indexing Strategy](#7-indexing-strategy)
8. [Backup and Recovery](#8-backup-and-recovery)
9. [Scaling](#9-scaling)
10. [Cost Estimation](#10-cost-estimation)

---

## 1. Executive Summary

### Purpose
The Data Storage Layer provides the persistence foundation for VitalsDrive's fleet health monitoring platform, leveraging Supabase PostgreSQL with realtime capabilities for real-time telemetry ingestion and WebSocket distribution to Angular clients.

### Scope
- Primary data store for vehicle telemetry logs
- User and fleet management
- Alert/notification persistence
- Realtime pub/sub for live telemetry updates

### Goals
- **Reliability:** 99.9% uptime with automated backups
- **Performance:** Sub-100ms read latency for dashboard queries
- **Security:** Tenant isolation via Row Level Security
- **Cost Efficiency:** Stay within Supabase free tier for MVP

### Key Decisions
| Decision | Choice | Rationale |
|----------|--------|-----------|
| Database | Supabase PostgreSQL | Built-in auth, realtime, RLS |
| Realtime | Postgres Changes | Native Supabase feature |
| Multi-tenancy | RLS policies | Row-level isolation without separate DBs |
| Backup | Supabase managed | Daily automated, point-in-time recovery |

---

## 2. Database Schema

### 2.1 Core Tables

#### telemetry_logs

High-frequency time-series data from IoT sensors. Each row is a single telemetry reading.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | BIGSERIAL | PRIMARY KEY | Auto-incrementing identifier |
| vehicle_id | UUID | NOT NULL, FK → vehicles | Which vehicle this reading is from |
| lat | DECIMAL(10,7) | CHECK -90..90 | Latitude |
| lng | DECIMAL(10,7) | CHECK -180..180 | Longitude |
| temp | INTEGER | | Coolant temperature (°C) |
| voltage | DECIMAL(5,2) | | Battery voltage (V) |
| rpm | INTEGER | CHECK >= 0 | Engine RPM |
| dtc_codes | TEXT[] | | Array of active diagnostic trouble codes (empty = no faults) |
| gps_valid | BOOLEAN | DEFAULT true | Whether GPS signal was valid |
| direction | INTEGER | | Heading 0-360 degrees |
| gsm_signal | INTEGER | | Cellular signal strength 0-31 |
| parser_version | TEXT | | Parser version for debugging |
| device_imei | VARCHAR(20) | | Device identifier from login packet |
| timestamp | TIMESTAMPTZ | DEFAULT NOW() | Server-side ingestion time |

**Expected Volume:** ~10 readings/vehicle/minute. At 50 vehicles: ~720K rows/day.

#### vehicles

Registered vehicles belonging to fleets.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY, gen_random_uuid | Vehicle identifier |
| fleet_id | UUID | NOT NULL, FK → fleets | Fleet the vehicle belongs to |
| vin | VARCHAR(17) | UNIQUE, NOT NULL | Vehicle Identification Number |
| make | VARCHAR(50) | NOT NULL | Vehicle manufacturer |
| model | VARCHAR(50) | NOT NULL | Vehicle model |
| year | INTEGER | | Model year |
| license_plate | VARCHAR(20) | | License plate number |
| status | VARCHAR(20) | DEFAULT 'active' | active, inactive, or maintenance |
| metadata | JSONB | DEFAULT '{}' | Extensible vehicle attributes |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Registration time |

#### fleets

Fleet groupings for organizing vehicles.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY, gen_random_uuid | Fleet identifier |
| name | VARCHAR(100) | NOT NULL | Display name |
| owner_id | UUID | FK → users | Fleet owner (user who created it) |
| settings | JSONB | DEFAULT '{}' | Fleet-level settings (timezone, data retention) |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Creation time |

#### fleet_members

Junction table linking users to fleets with role-based access.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Membership identifier |
| fleet_id | UUID | FK → fleets, ON DELETE CASCADE | Fleet |
| user_id | UUID | FK → users, ON DELETE CASCADE | User |
| role | VARCHAR(20) | DEFAULT 'member' | owner, admin, member, or viewer |
| joined_at | TIMESTAMPTZ | DEFAULT NOW() | Join time |

**Constraint:** UNIQUE(fleet_id, user_id) — a user can only have one membership per fleet.

#### alerts

Generated alerts from telemetry threshold violations.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | BIGSERIAL | PRIMARY KEY | Alert identifier |
| vehicle_id | UUID | NOT NULL, FK → vehicles, ON DELETE CASCADE | Affected vehicle |
| fleet_id | UUID | NOT NULL, FK → fleets, ON DELETE CASCADE | Parent fleet |
| severity | VARCHAR(10) | NOT NULL | critical, warning, or info |
| code | VARCHAR(50) | NOT NULL | Alert code (e.g., 'ENGINE_OVERTEMP', 'LOW_VOLTAGE') |
| message | TEXT | NOT NULL | Human-readable description |
| dtc_codes | TEXT[] | | Related DTC codes |
| lat, lng | DECIMAL(10,7) | | Location where alert was triggered |
| acknowledged | BOOLEAN | DEFAULT FALSE | Whether alert has been seen |
| acknowledged_by | UUID | FK → users | User who acknowledged |
| acknowledged_at | TIMESTAMPTZ | | When acknowledged |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Alert creation time |
| resolved_at | TIMESTAMPTZ | | When resolved |

#### telemetry_rules

User-defined threshold rules for generating alerts.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | BIGSERIAL | PRIMARY KEY | Rule identifier |
| fleet_id | UUID | NOT NULL, FK → fleets | Fleet this rule applies to |
| name | VARCHAR(100) | NOT NULL | Display name |
| metric | VARCHAR(20) | NOT NULL | temp, voltage, or rpm |
| operator | VARCHAR(5) | NOT NULL | gt, lt, gte, lte, or eq |
| threshold | FLOAT | NOT NULL | Threshold value |
| severity | VARCHAR(10) | NOT NULL | critical, warning, or info |
| enabled | BOOLEAN | DEFAULT TRUE | Whether rule is active |
| cooldown_seconds | INTEGER | DEFAULT 300 | Min time between repeated alerts |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Rule creation time |

### 2.2 Relationships

```
fleets ───1:N──▶ vehicles ───1:N──▶ telemetry_logs
  │                  │                      │
  │                  │                      │ (trigger)
  │                  │                      ▼
  │                  │                  alerts
  │                  │
  │               1:N
  │                  │
  ▼                  ▼
fleet_members (M:N users ↔ fleets)
```

### 2.3 Optional Tables (Post-MVP)

**audit_logs** — Audit trail for compliance. Records user actions with old/new data snapshots.

**scheduled_maintenance** — Vehicle maintenance schedule tracking with due dates, mileage, and completion status.

---

## 3. Supabase Realtime Configuration

### 3.1 Published Tables

The following tables must be published to `supabase_realtime`:

| Table | Event | Purpose |
|---|---|---|
| telemetry_logs | INSERT | Push new readings to dashboard in real-time |
| alerts | INSERT, UPDATE | Push new alerts and acknowledgment updates |

### 3.2 Subscription Patterns

Clients subscribe to realtime channels with vehicle-specific filters to minimize payload. For fleet-wide views, clients subscribe to all telemetry and filter client-side based on their fleet membership (enforced by RLS).

### 3.3 Realtime Constraints

- Maximum 10 concurrent subscriptions per client (Supabase free tier)
- Payloads should stay under 1KB per event
- Supabase SDK handles reconnection with exponential backoff
- Channels must be cleaned up on component destroy to prevent memory leaks

---

## 4. Row Level Security (RLS) Policies

### 4.1 Core Principle

All data access is controlled at the database level. The Supabase service role key is used only by the ingestion server (Layer 1) and bypasses RLS entirely.

### 4.2 Helper Function

A `get_user_fleet_ids()` function returns all fleet IDs the current user has access to. This is used as a subquery in RLS policies to avoid repeated complex joins.

### 4.3 Policy Matrix

| Table | Operation | Who | Condition |
|---|---|---|---|
| telemetry_logs | SELECT | Fleet members | vehicle's fleet is in user's fleet list |
| telemetry_logs | INSERT | Service role only | Service role bypasses RLS |
| vehicles | SELECT | Fleet members | fleet_id in user's fleet list |
| vehicles | INSERT/UPDATE/DELETE | Fleet owners/admins | user has owner or admin role |
| fleets | SELECT | Fleet members | user is a member |
| fleets | INSERT | Any authenticated user | Sets creator as owner |
| fleets | UPDATE | Fleet owners/admins | user has owner or admin role |
| fleets | DELETE | Fleet owners only | owner_id matches user |
| fleet_members | SELECT | Fleet members | user is a member of the fleet |
| fleet_members | INSERT/DELETE | Fleet owners/admins | user has owner or admin role |
| alerts | SELECT | Fleet members | vehicle's fleet is in user's fleet list |
| alerts | INSERT | System/service role | Service role bypasses RLS |
| alerts | UPDATE | Fleet owners/admins | For acknowledgment |
| telemetry_rules | SELECT | Fleet members | fleet_id in user's fleet list |
| telemetry_rules | INSERT/UPDATE/DELETE | Fleet owners/admins | user has owner or admin role |

---

## 5. API Design

### 5.1 Telemetry Endpoints

| Operation | Method | Supabase API | Description |
|---|---|---|---|
| Ingest telemetry | INSERT | POST /rest/v1/telemetry_logs | Parser pushes decoded data (service role) |
| Get latest | SELECT | GET /rest/v1/telemetry_logs?vehicle_id=eq.X | Dashboard reads latest reading per vehicle |
| Get history | SELECT | GET /rest/v1/telemetry_logs?vehicle_id=eq.X&order=timestamp.desc | Historical view |
| Subscribe | Realtime | `channel('telemetry-live').on('postgres_changes', ...)` | Live updates |

### 5.2 Vehicle Endpoints

| Operation | Method | Supabase API | Description |
|---|---|---|---|
| List vehicles | SELECT | GET /rest/v1/vehicles?fleet_id=eq.X | Fleet vehicle list |
| Add vehicle | INSERT | POST /rest/v1/vehicles | Register new vehicle |
| Update vehicle | UPDATE | PATCH /rest/v1/vehicles | Edit vehicle details |
| Get details | SELECT | GET /rest/v1/vehicles?id=eq.X | Single vehicle detail |

### 5.3 Fleet Endpoints

| Operation | Method | Supabase API | Description |
|---|---|---|---|
| List fleets | SELECT | GET /rest/v1/fleets | User's fleet list |
| Create fleet | INSERT | POST /rest/v1/fleets | New fleet (caller becomes owner) |
| Update fleet | UPDATE | PATCH /rest/v1/fleets | Edit fleet settings |
| Delete fleet | DELETE | DELETE /rest/v1/fleets | Remove fleet (owner only) |

### 5.4 Alert Endpoints

| Operation | Method | Supabase API | Description |
|---|---|---|---|
| List alerts | SELECT | GET /rest/v1/alerts?fleet_id=eq.X | Active/past alerts |
| Acknowledge alert | UPDATE | PATCH /rest/v1/alerts | Mark as acknowledged |

---

## 6. Database Functions and Triggers

### 6.1 Alert Generation Trigger

A database trigger on `telemetry_logs` (INSERT) evaluates incoming readings against `telemetry_rules` for the vehicle's fleet. If a threshold is violated, it creates a new row in the `alerts` table. The trigger respects the `cooldown_seconds` field to prevent alert spam.

### 6.2 Auto-Cleanup Function

A scheduled function (`cleanup_old_telemetry()`) runs monthly via pg_cron to delete telemetry rows older than the fleet's `data_retention_days` setting (default 90 days). This keeps the free tier under the 500MB limit.

### 6.3 Updated Timestamp Trigger

An `updated_at` trigger on `vehicles`, `fleets`, and `users` automatically sets `updated_at = NOW()` on every UPDATE, ensuring audit trails are accurate without application-layer logic.

---

## 7. Indexing Strategy

### 7.1 Required Indexes

| Table | Column(s) | Type | Purpose |
|---|---|---|---|
| telemetry_logs | vehicle_id | B-tree | Vehicle history queries |
| telemetry_logs | timestamp DESC | B-tree | Time-range queries |
| telemetry_logs | (vehicle_id, timestamp DESC) | Composite | Latest telemetry per vehicle |
| vehicles | fleet_id | B-tree | Fleet vehicle list |
| vehicles | vin | UNIQUE | Vehicle identification lookup |
| fleet_members | (user_id, fleet_id) | Composite | RLS policy performance |
| fleet_members | user_id | B-tree | User's fleet list |
| alerts | (fleet_id, created_at DESC) | Composite | Fleet alert timeline |
| alerts | vehicle_id | B-tree | Vehicle-specific alerts |
| alerts | (severity, acknowledged) | Composite | Active alert filtering |

---

## 8. Backup and Recovery

### 8.1 Backup Strategy

| Backup Type | Frequency | Retention | Provider |
|---|---|---|---|
| Automated Daily | Daily | 7 days | Supabase managed |
| Point-in-time Recovery | Continuous | 30 days | Supabase managed |
| Manual | On-demand | Until deleted | Supabase Dashboard |

### 8.2 Recovery Procedures

- **Point-in-time:** Use Supabase Dashboard → Database → Backups to select a recovery timestamp
- **Full restore:** Create new Supabase project from backup, update environment variables
- **Partial restore:** Use pg_restore from manual backup dump

---

## 9. Scaling

### 9.1 Growth Projections

| Phase | Vehicles | Rows/Day | Monthly Storage | Supabase Tier |
|---|---|---|---|---|
| MVP (1-10) | 10 | 144K | ~50MB | Free (500MB) |
| Growth (10-50) | 50 | 720K | ~250MB | Free (500MB) |
| Scale (50-200) | 200 | 2.9M | ~1GB | Pro (8GB) |

### 9.2 Scaling Triggers

| Metric | Investigate At | Act At |
|---|---|---|
| Database size | > 300MB | > 400MB (80% of free tier) |
| Monthly bandwidth | > 1GB | > 1.5GB |
| Connection count | > 40 | > 100 |
| Query latency (p99) | > 300ms | > 500ms |

### 9.3 Read Replica Strategy

At scale (200+ vehicles), consider:
- Supabase Pro tier with read replica for dashboard queries
- Ingestion server writes directly to primary
- Dashboard reads from replica
- Connection pooling via Supavisor

---

## 10. Cost Estimation

| Phase | Database Size | Tier | Monthly Cost |
|---|---|---|---|
| MVP (1-10 vehicles) | < 50MB | Free | $0 |
| Growth (10-50) | < 250MB | Free | $0 |
| Scale (50-200) | ~1GB | Pro | $25 |
| Enterprise (200+) | 2-8GB | Pro | $25 |

---

*Layer Owner: Backend Infrastructure
Next Review: End of Phase 1 (Week 1 of development)*
