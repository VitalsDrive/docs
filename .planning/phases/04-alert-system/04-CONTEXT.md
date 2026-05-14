# Phase 4: Alert System - Context

**Gathered:** 2026-05-14
**Status:** Ready for planning

<domain>
## Phase Boundary

Detect threshold violations (battery voltage, coolant temperature) and DTC codes from live telemetry, persist alert records in Supabase, and surface them in the Angular dashboard via toast notifications, a persistent /alerts list page, vehicle card badges, and a nav bell indicator. Detection happens server-side via a PostgreSQL trigger — alerts fire whether or not any user is logged in.

**In scope:**
- PostgreSQL trigger on `telemetry_logs` that evaluates thresholds and writes to `alerts` table
- Static P-code DTC JSON asset bundled in dashboard for plain-English lookup
- `/alerts` route: paginated list of last 7 days of alerts with per-alert acknowledgment
- Toast notification on new alert arrival (reuse Phase 3 ToastComponent)
- Vehicle card alert badge (severity-colored, alert count)
- Fleet map marker: color-coded by severity + pulse animation when active alerts exist
- Shell nav bell icon with unacknowledged count badge, links to /alerts
- RLS on alerts table scoped to fleet membership

**Out of scope (deferred):**
- Email/push notifications (NOTF-01/02/03 — v2 requirements)
- Per-fleet configurable thresholds via telemetry_rules (deferred to future phase)
- Bulk alert acknowledgment
- Alerts older than 7 days in UI (data stays in DB, not shown)

</domain>

<decisions>
## Implementation Decisions

### Alert Detection (Server-Side)
- **D-01:** PostgreSQL `AFTER INSERT` trigger on `telemetry_logs` evaluates each new row against hardcoded thresholds. On violation, inserts a row into the existing `alerts` table. A dedup guard prevents re-alerting for the same vehicle+alert-type combination within a 1-hour window (checks for unacknowledged alert of same code within 1 hour).
- **D-08:** Dedup window is **1 hour** per vehicle+alert-type. An alert acknowledged by the user resets the dedup guard for that type (acknowledged=true rows are excluded from the dedup check).

### Alert Thresholds (Hardcoded)
- **D-04:** Hardcoded constants in the trigger function — no per-fleet configurability for MVP:
  - Battery voltage: **warning < 11.8V**, **critical < 11.5V**
  - Coolant temperature: **warning > 100°C**, **critical > 110°C**
  - DTC codes: **any non-empty `dtc_codes` array** → one alert per distinct DTC code
- **D-10:** DTC codes trigger **one alert per code** (3 simultaneous codes = 3 separate alert rows). Each alert has its own `code`, `message` (plain-English), and `severity`.

### DTC Code Lookup
- **D-02:** Plain-English translation via **static JSON asset bundled in dashboard** — not a DB query. Coverage: **P-codes only** (~2000 codes). Industry-standard OBD-II P-code list. Loaded once at app startup, referenced by DTC service for lookups.
- **D-03:** P-codes only (P0xxx, P1xxx) — matches FMC003 device output. C/B/U codes not needed for current device fleet.

### Alert Display — Toast
- **D-05a:** New alerts arriving via Realtime subscription trigger a toast notification. Reuse the existing `ToastComponent` from Phase 3. Toast shows: severity icon, vehicle name, alert message, "View" link to /alerts.

### Alert Display — /alerts Page
- **D-05b:** `/alerts` route shows a **paginated list** of alerts from the **last 7 days** (newest first). Columns: severity badge, vehicle name, alert type, description, timestamp, acknowledged status, Acknowledge button.
- **D-05c:** Per-alert acknowledgment via Acknowledge button → sets `acknowledged=true` in Supabase. Acknowledged alerts remain visible but visually dimmed (separate "Resolved" visual state, not hidden).

### Alert Display — Vehicle Cards
- **D-05d:** Vehicle cards on the fleet dashboard show a **severity-colored badge** with unacknowledged alert count. Red for critical, yellow for warning. Derived from `VehicleWithHealth.alert_count` + highest severity.

### Alert Display — Fleet Map Markers
- **D-06:** Map markers are **color-coded by highest active alert severity**: red (critical), yellow (warning), green (healthy/no alerts). Markers with active unacknowledged alerts also show a **pulse animation**. Reuses existing Leaflet marker system from Phase 3 (`FleetMapComponent`).

### Alert Display — Shell Nav
- **D-07:** Shell navigation includes a **bell icon** in the header showing the total count of unacknowledged alerts across all fleet vehicles. Count updates via Realtime subscription on the alerts table. Clicking the bell navigates to `/alerts`. Badge disappears when count reaches 0.

### Notifications (Deferred)
- **D-11:** Email and push notifications (NOTF-01/02/03) are **deferred to a future phase**. Phase 4 delivers in-app alerting only. `telemetry_rules` table stays unused for now.

### Claude's Discretion
- Exact SQL trigger function structure and PL/pgSQL implementation
- Angular component structure for AlertsPageComponent and AlertListItemComponent
- Realtime subscription management (single shared subscription vs per-component)
- Pagination strategy for /alerts (offset vs cursor-based)
- RLS policy details for alerts table (read: fleet members; write: trigger only)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase Scope
- `docs/.planning/ROADMAP.md` §Phase 4 — Goal, requirements list, success criteria
- `docs/.planning/REQUIREMENTS.md` §DTC Alerts, §Battery Health, §Coolant Temperature — DTC-01–03, BATT-01–03, COOL-01–03

### Database Schema
- `packages/supabase/migrations/` — Existing `alerts` table schema (severity, code, message, dtc_codes, acknowledged, vehicle_id, fleet_id, lat/lng). `telemetry_rules` table (unused, available for future). `telemetry_logs` table (voltage, temp, dtc_codes columns).

### Dashboard — Angular Patterns
- `packages/dashboard/src/app/core/services/vehicle.service.ts` — `VehicleWithHealth`, `calculateHealthScore()`, `alert_count`, `resource()` pattern, `processBatch()`
- `packages/dashboard/src/app/core/models/telemetry.model.ts` — TelemetryRecord interface
- `packages/dashboard/src/app/core/models/vehicle.model.ts` — VehicleWithHealth model
- `packages/dashboard/src/app/shell/` — ShellComponent, ConnectionPillComponent (nav integration point for bell icon)
- `packages/dashboard/src/app/app.routes.ts` — `/alerts` route already exists

### Reusable UI
- `packages/dashboard/src/app/shared/components/toast/` — ToastComponent from Phase 3 (reuse for alert toasts)
- `packages/dashboard/src/app/features/fleet/` — FleetMapComponent, vehicle card components (alert badge integration points)

### Map Integration
- `packages/dashboard/src/app/features/map/` — Leaflet marker implementation from Phase 3 (color-coding + pulse animation extension point)

### Prior Phase Context
- `docs/.planning/phases/03-live-fleet-dashboard/03-CONTEXT.md` — Phase 3 decisions (resource() pattern, VehicleService architecture, realtime subscription ownership)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ToastComponent` (Phase 3) — already wired for connection state; extend for alert notifications
- `VehicleWithHealth.alert_count` — already computed in VehicleService; needs to be wired to visible badge
- `calculateHealthScore()` in VehicleService — already uses voltage/coolant thresholds; review for alignment with trigger thresholds
- Leaflet markers in FleetMapComponent — existing color/icon system; extend with severity-based colors and pulse CSS

### Established Patterns
- `resource()` API with `params` property — use for AlertService data loading (mirrors OrganizationService, VehicleService)
- Angular signals: `signal()`, `computed()`, `effect()` — no NgRx; alerts state as signals
- `auth.jwt()->>'sub'` for RLS — all new alert RLS policies must use this, not `auth.uid()`
- Supabase Realtime: existing fleet subscription in VehicleService — alerts table needs its own subscription (same pattern)
- Angular Material — all UI components (mat-badge, mat-icon for bell, mat-list for alerts page, mat-chip for severity)
- Soft-delete: vehicles use `status: 'inactive'` — alert acknowledgment uses `acknowledged=true` field (already on table)

### Integration Points
- `ShellComponent` — nav integration point for bell icon + badge count
- `VehicleService` — owns telemetry subscription; alert subscription can live in a new `AlertService`
- `FleetMapComponent.updateMarkers()` — extend to accept alert state per vehicle for color/pulse
- `/alerts` route — already defined in app.routes.ts, needs a component (AlertsPageComponent) wired up

</code_context>

<specifics>
## Specific Ideas

- Map markers: color-coded severity (red/yellow/green) **AND** pulsing animation when active alerts exist — user specifically requested both
- The existing `alerts` table already has the right schema — no schema additions needed beyond the trigger function and any missing indexes
- Dedup guard logic: `NOT EXISTS (SELECT 1 FROM alerts WHERE vehicle_id = NEW.vehicle_id AND code = $code AND acknowledged = false AND created_at > NOW() - INTERVAL '1 hour')`

</specifics>

<deferred>
## Deferred Ideas

- **Email notifications** (NOTF-01) — high user value, deferred to Phase 6 or future milestone
- **Push notifications** (NOTF-02/03) — deferred to future phase
- **Per-fleet configurable thresholds** via `telemetry_rules` table — table exists, deferred to after MVP
- **Bulk alert acknowledgment** (acknowledge all / by vehicle) — deferred, can be added as enhancement
- **Alerts older than 7 days** in UI — data retained in DB, UI filter can be expanded later

</deferred>

---

*Phase: 4-alert-system*
*Context gathered: 2026-05-14*
