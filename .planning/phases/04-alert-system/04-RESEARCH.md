# Phase 4: Alert System - Research

**Researched:** 2026-05-14
**Domain:** PostgreSQL triggers + PL/pgSQL, Supabase Realtime, Angular 21 signals, Angular Material
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** PostgreSQL `AFTER INSERT` trigger on `telemetry_logs` evaluates each new row against hardcoded thresholds. On violation, inserts a row into the existing `alerts` table. Dedup guard prevents re-alerting for same vehicle+alert-type combination within 1-hour window (checks for unacknowledged alert of same code within 1 hour).
- **D-02:** Plain-English DTC translation via static JSON asset bundled in dashboard — not a DB query. Coverage: P-codes only (~2000 codes). Loaded once at app startup.
- **D-03:** P-codes only (P0xxx, P1xxx) — matches FMC003 device output. C/B/U codes not needed.
- **D-04:** Hardcoded thresholds in trigger function — battery warning <11.8V, critical <11.5V; coolant warning >100°C, critical >110°C; DTC any non-empty dtc_codes array → one alert per distinct code.
- **D-05a:** New alerts arriving via Realtime subscription trigger a toast. Reuse existing ToastComponent. Shows: severity icon, vehicle name, alert message, "View" link to /alerts.
- **D-05b:** `/alerts` route — paginated list of last 7 days (newest first). Columns: severity badge, vehicle name, alert type, description, timestamp, acknowledged status, Acknowledge button.
- **D-05c:** Per-alert acknowledgment → sets acknowledged=true. Acknowledged alerts remain visible but visually dimmed.
- **D-05d:** Vehicle cards show severity-colored badge with unacknowledged alert count. Red for critical, yellow for warning. Derived from VehicleWithHealth.alert_count + highest severity.
- **D-06:** Map markers color-coded by highest active alert severity: red (critical), yellow (warning), green (healthy). Markers with active unacknowledged alerts show pulse animation. Reuses existing Leaflet marker system.
- **D-07:** Shell nav bell icon in header showing total unacknowledged alert count across all fleet vehicles. Count updates via Realtime. Clicking bell navigates to /alerts. Badge disappears when count = 0.
- **D-08:** Dedup window is 1 hour per vehicle+alert-type. Acknowledged alert resets dedup guard.
- **D-10:** DTC codes trigger one alert per code (3 simultaneous codes = 3 separate alert rows).
- **D-11:** Email/push notifications (NOTF-01/02/03) deferred to future phase.

### Claude's Discretion
- Exact SQL trigger function structure and PL/pgSQL implementation
- Angular component structure for AlertsPageComponent and AlertListItemComponent
- Realtime subscription management (single shared subscription vs per-component)
- Pagination strategy for /alerts (offset vs cursor-based)
- RLS policy details for alerts table (read: fleet members; write: trigger only)

### Deferred Ideas (OUT OF SCOPE)
- Email/push notifications (NOTF-01/02/03)
- Per-fleet configurable thresholds via telemetry_rules table
- Bulk alert acknowledgment
- Alerts older than 7 days in UI
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DTC-01 | System detects when DTC codes appear on a vehicle | PostgreSQL trigger on telemetry_logs.dtc_codes; one alert row per distinct code |
| DTC-02 | User receives alert with specific DTC code and plain-English explanation | DtcTranslationService already exists; AlertService already generates DTC alert messages; trigger writes message to alerts.message |
| DTC-03 | User can view history of DTC alerts for a vehicle | /alerts page filtered by vehicle or alert type; 7-day window query on alerts table |
| BATT-01 | System monitors battery voltage from OBD2 telemetry | telemetry_logs.voltage already stored; trigger evaluates on each INSERT |
| BATT-02 | User receives alert when battery voltage trends toward no-start threshold | Trigger fires at <11.8V (warning) and <11.5V (critical); toast via Realtime |
| BATT-03 | Dashboard displays current battery voltage for each vehicle | VehicleCard already has BatteryStatusComponent reading latestTelemetry.voltage |
| COOL-01 | System monitors engine coolant temperature from OBD2 telemetry | telemetry_logs.temp already stored; trigger evaluates on each INSERT |
| COOL-02 | User receives alert when vehicle runs hot (overheating threshold) | Trigger fires at >100°C (warning) and >110°C (critical); toast via Realtime |
| COOL-03 | Dashboard displays current coolant temperature for each vehicle | VehicleCard already shows coolantTemp computed from latestTelemetry.temp |
</phase_requirements>

---

## Summary

Phase 4 has an unusually high proportion of pre-existing infrastructure to leverage. The `alerts` table, `telemetry_logs` table, indexes, and an early `generate_alert_if_needed()` trigger function all exist from migration 003 — but the trigger function uses the old `telemetry_rules` table (deferred) rather than hardcoded thresholds. Migration 014 must replace this function with a new hardcoded-threshold version. On the dashboard side, `AlertService`, `DtcTranslationService`, `DTC_DATABASE`, `ToastComponent`, `HeaderComponent` (with bell icon + badge signals), `AlertsComponent`, `FleetMapComponent` (with alert-aware state), and `VehicleCardComponent` (with alert badge CSS classes) are all already implemented — they operate on in-memory signals, not Supabase. Phase 4 wires all of this to the database: alerts become persistent, bell count comes from Supabase, and the /alerts page reads from the `alerts` table with pagination.

The two main work streams are: (1) database layer — new migration with a corrected trigger function + RLS policies for alerts; (2) Angular layer — refactor `AlertService` to load/subscribe from Supabase `alerts` table instead of generating in-memory alerts from telemetry, add `AlertService.loadAlerts()` resource(), add Realtime subscription on alerts table, wire the bell count, and rewrite AlertsComponent to be a proper paginated history page backed by Supabase.

**Primary recommendation:** Replace the existing in-memory AlertService with a Supabase-backed AlertService that loads from the `alerts` table via resource() + Realtime subscription, mirroring the VehicleService pattern exactly. The trigger handles detection server-side; the dashboard only reads and acknowledges.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Alert detection (threshold evaluation) | Database (PostgreSQL trigger) | — | Fires even when no user is logged in; server-side guarantees consistency |
| DTC dedup & 1-hour window | Database (trigger function) | — | Must be atomic; client-side dedup would race |
| Alert persistence | Database (alerts table) | — | Source of truth; survives browser refresh |
| DTC plain-English lookup | Browser (static JSON) | — | D-02 locked; no DB round-trip |
| Alert acknowledgment | API (Supabase RPC / direct UPDATE) | Browser (optimistic UI) | Direct UPDATE via supabase-js is sufficient; no edge function needed |
| Real-time alert delivery | Browser via Supabase Realtime | — | INSERT events on alerts table broadcast to subscribed dashboards |
| Bell badge count | Browser (computed signal) | — | Derived from Realtime subscription state; no separate query needed |
| /alerts history page | Browser (resource() + Supabase query) | — | Paginated SELECT on alerts table scoped by fleet |
| Map marker color/pulse | Browser (FleetMapComponent) | — | Extend existing getMarkerIcon() with per-vehicle alert severity |
| Vehicle card badge | Browser (VehicleCardComponent) | — | Extend existing cardBorderClass with mat-badge using Supabase alert count |
| RLS enforcement | Database | — | All reads must pass through fleet_members JOIN |

---

## Standard Stack

### Core (all already in package.json — no new installs)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `@supabase/supabase-js` | ^2.39.0 | Realtime subscription, SELECT, UPDATE on alerts table | Already in use; established pattern in VehicleService |
| `@angular/core` | ^21.0.0 | signals, resource(), effect(), computed() | Project standard; all services use this pattern |
| `@angular/material` | ^21.0.0 | mat-badge, mat-icon (bell), mat-list, mat-chip, mat-paginator | Project standard; all UI uses Angular Material |
| PostgreSQL / PL/pgSQL | via Supabase | AFTER INSERT trigger function | Supabase managed Postgres |

### No New Dependencies
Phase 4 adds zero new npm packages. All required libraries are already installed.

**Installation:** None required.

---

## Architecture Patterns

### System Architecture Diagram

```
telemetry_logs INSERT
        |
        v
[PostgreSQL AFTER INSERT trigger: check_telemetry_alerts()]
        |
        |-- voltage < 11.8V → INSERT alerts (BATTERY_WARNING)
        |-- voltage < 11.5V → INSERT alerts (BATTERY_CRITICAL)
        |-- temp > 100°C    → INSERT alerts (COOLANT_WARNING)
        |-- temp > 110°C    → INSERT alerts (COOLANT_CRITICAL)
        |-- dtc_codes[i]    → INSERT alerts (DTC_{code}) × N codes
        |
        | (dedup guard: skip if unacknowledged alert same code+vehicle within 1h)
        v
    [alerts table]
        |
        |--- Supabase Realtime (INSERT events) ──────────────────────────────>
        |                                                                      |
        |--- SELECT (last 7 days, paginated) ─── AlertService.alertResource   |
        |                                                                      v
        |--- UPDATE acknowledged=true ────────── acknowledge()           [AlertService]
                                                                              |
                                                   ┌──────────────────────────┤
                                                   |              |            |
                                               [ToastComponent] [AlertsPage] [HeaderComponent bell]
                                                                   |            |
                                                              [VehicleCard]  [FleetMap markers]
                                                              badge+severity  color+pulse
```

### Recommended Project Structure (new files only)

```
packages/supabase/migrations/
└── 014_alert_trigger_and_rls.sql  # New hardcoded-threshold trigger + alerts RLS

packages/dashboard/src/app/
├── core/
│   ├── models/
│   │   └── alert.model.ts         # Extend: add SupabaseAlert interface for DB rows
│   └── services/
│       └── alert.service.ts       # Refactor: add Supabase resource() + Realtime
└── features/
    └── alerts/
        ├── alerts.component.ts    # Rewrite: Supabase-backed, paginated, 7-day window
        ├── alerts.component.html  # Rewrite: mat-list with paginator
        └── alerts.component.css   # Update: dimmed state for acknowledged alerts
```

No new components needed beyond AlertsComponent rewrite. Bell icon is already in HeaderComponent. Vehicle card badge is already CSS-classed. Map marker alert state already handled via `alertCount`.

### Pattern 1: AlertService resource() + Realtime (mirrors VehicleService)

**What:** Load alerts from Supabase on init, subscribe to Realtime INSERT events, expose computed signals.
**When to use:** Any service that needs both initial data load and live updates.

```typescript
// Source: VehicleService pattern (vehicle.service.ts lines 48-96)
// AlertService equivalent:

readonly alertResource = resource({
  params: () => {
    const fleetIds = this.fleetService.fleets().map(f => f.id);
    return fleetIds.length > 0 ? { fleetIds } : undefined;
  },
  loader: async ({ params: { fleetIds } }) => {
    const since = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString();
    const { data, error } = await this.supabase.client
      .from('alerts')
      .select('*')
      .in('fleet_id', fleetIds)
      .gte('created_at', since)
      .order('created_at', { ascending: false })
      .limit(200);
    if (error) throw error;
    this._dbAlerts.set((data ?? []) as SupabaseAlert[]);
    return data;
  },
});
```

### Pattern 2: Supabase Realtime for alerts table

**What:** Subscribe to INSERT events on the alerts table, filtered by fleet_id.
**When to use:** Delivering new alerts to the dashboard without polling.

```typescript
// Source: [ASSUMED based on supabase-js v2 pattern, mirrors TelemetryService subscription]
// Single shared subscription in AlertService constructor or effect():

private subscribeToAlerts(fleetIds: string[]): void {
  this.supabase.client
    .channel('alerts-realtime')
    .on(
      'postgres_changes',
      {
        event: 'INSERT',
        schema: 'public',
        table: 'alerts',
        // Note: supabase-js v2 filter supports eq/neq/lt/lte/gt/gte/in for single column
        // For multi-fleet, subscribe without filter and filter in handler,
        // OR subscribe per fleet_id (one channel per fleet).
        // Simplest: no server-side filter, check fleet_id in handler.
      },
      (payload) => {
        const alert = payload.new as SupabaseAlert;
        if (fleetIds.includes(alert.fleet_id)) {
          this._dbAlerts.update(alerts => [alert, ...alerts]);
          // Trigger toast by pushing to in-memory active alerts
        }
      }
    )
    .subscribe();
}
```

**Important:** Supabase Realtime `filter` for `in` operator on `fleet_id` requires the format `fleet_id=in.(uuid1,uuid2)` — verify against actual supabase-js version. Safer approach for MVP: no server-side filter, check in handler. [ASSUMED: in-filter syntax — verify before implementing]

### Pattern 3: PostgreSQL AFTER INSERT Trigger (new migration)

**What:** Replace the existing `generate_alert_if_needed()` with hardcoded-threshold `check_telemetry_alerts()`.
**When to use:** Server-side alert detection independent of client connection state.

```sql
-- Source: [ASSUMED PL/pgSQL pattern, consistent with migration 003 style]
-- Migration 014: drop old trigger, create new function + trigger

CREATE OR REPLACE FUNCTION check_telemetry_alerts()
RETURNS TRIGGER AS $$
DECLARE
  v_fleet_id UUID;
BEGIN
  SELECT fleet_id INTO v_fleet_id FROM vehicles WHERE id = NEW.vehicle_id;
  IF v_fleet_id IS NULL THEN RETURN NEW; END IF;

  -- Battery: CRITICAL
  IF NEW.voltage IS NOT NULL AND NEW.voltage < 11.5 THEN
    IF NOT EXISTS (
      SELECT 1 FROM alerts
      WHERE vehicle_id = NEW.vehicle_id
        AND code = 'BATTERY_CRITICAL'
        AND acknowledged = false
        AND created_at > NOW() - INTERVAL '1 hour'
    ) THEN
      INSERT INTO alerts (vehicle_id, fleet_id, severity, code, message, lat, lng)
      VALUES (NEW.vehicle_id, v_fleet_id, 'critical', 'BATTERY_CRITICAL',
        'Battery critical: ' || ROUND(NEW.voltage::NUMERIC, 1) || 'V — vehicle may not start',
        NEW.lat, NEW.lng);
    END IF;
  -- Battery: WARNING (only if not already in critical territory)
  ELSIF NEW.voltage IS NOT NULL AND NEW.voltage < 11.8 THEN
    IF NOT EXISTS (
      SELECT 1 FROM alerts
      WHERE vehicle_id = NEW.vehicle_id
        AND code IN ('BATTERY_WARNING', 'BATTERY_CRITICAL')
        AND acknowledged = false
        AND created_at > NOW() - INTERVAL '1 hour'
    ) THEN
      INSERT INTO alerts (vehicle_id, fleet_id, severity, code, message, lat, lng)
      VALUES (NEW.vehicle_id, v_fleet_id, 'warning', 'BATTERY_WARNING',
        'Battery low: ' || ROUND(NEW.voltage::NUMERIC, 1) || 'V',
        NEW.lat, NEW.lng);
    END IF;
  END IF;

  -- Coolant: CRITICAL
  IF NEW.temp IS NOT NULL AND NEW.temp > 110 THEN
    IF NOT EXISTS (
      SELECT 1 FROM alerts
      WHERE vehicle_id = NEW.vehicle_id
        AND code = 'COOLANT_CRITICAL'
        AND acknowledged = false
        AND created_at > NOW() - INTERVAL '1 hour'
    ) THEN
      INSERT INTO alerts (vehicle_id, fleet_id, severity, code, message, lat, lng)
      VALUES (NEW.vehicle_id, v_fleet_id, 'critical', 'COOLANT_CRITICAL',
        'Engine overheating: ' || ROUND(NEW.temp::NUMERIC, 0) || '°C — stop and cool down',
        NEW.lat, NEW.lng);
    END IF;
  -- Coolant: WARNING
  ELSIF NEW.temp IS NOT NULL AND NEW.temp > 100 THEN
    IF NOT EXISTS (
      SELECT 1 FROM alerts
      WHERE vehicle_id = NEW.vehicle_id
        AND code IN ('COOLANT_WARNING', 'COOLANT_CRITICAL')
        AND acknowledged = false
        AND created_at > NOW() - INTERVAL '1 hour'
    ) THEN
      INSERT INTO alerts (vehicle_id, fleet_id, severity, code, message, lat, lng)
      VALUES (NEW.vehicle_id, v_fleet_id, 'warning', 'COOLANT_WARNING',
        'Engine running hot: ' || ROUND(NEW.temp::NUMERIC, 0) || '°C',
        NEW.lat, NEW.lng);
    END IF;
  END IF;

  -- DTC codes: one alert per distinct code
  IF NEW.dtc_codes IS NOT NULL AND array_length(NEW.dtc_codes, 1) > 0 THEN
    DECLARE
      dtc_code TEXT;
    BEGIN
      FOREACH dtc_code IN ARRAY NEW.dtc_codes LOOP
        IF NOT EXISTS (
          SELECT 1 FROM alerts
          WHERE vehicle_id = NEW.vehicle_id
            AND code = dtc_code
            AND acknowledged = false
            AND created_at > NOW() - INTERVAL '1 hour'
        ) THEN
          INSERT INTO alerts (vehicle_id, fleet_id, severity, code, message, dtc_codes, lat, lng)
          VALUES (NEW.vehicle_id, v_fleet_id, 'warning', dtc_code,
            'DTC fault code: ' || dtc_code,
            ARRAY[dtc_code], NEW.lat, NEW.lng);
        END IF;
      END LOOP;
    END;
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**Note on PL/pgSQL DECLARE inside IF block:** The FOREACH + nested DECLARE pattern shown is valid PL/pgSQL but the DECLARE must be in a sub-block `BEGIN...END`. Tested pattern: outer DECLARE is before the first BEGIN, inner DECLARE uses an explicit `DECLARE` block. [ASSUMED: verified consistent with PostgreSQL docs for anonymous block scoping]

### Pattern 4: Alert Acknowledgment

**What:** Direct UPDATE via supabase-js; no RPC needed for simple boolean flip.

```typescript
// Source: [ASSUMED pattern, consistent with supabase-js v2 UPDATE usage]
async acknowledgeAlert(alertId: number): Promise<void> {
  const { error } = await this.supabase.client
    .from('alerts')
    .update({
      acknowledged: true,
      acknowledged_at: new Date().toISOString(),
    })
    .eq('id', alertId);
  if (error) throw error;
  // Optimistic update: flip acknowledged flag in _dbAlerts signal
  this._dbAlerts.update(alerts =>
    alerts.map(a => a.id === alertId ? { ...a, acknowledged: true } : a)
  );
}
```

### Pattern 5: Pagination for /alerts page

**What:** Offset-based pagination (simpler than cursor for this use case — fixed 7-day window, no infinite scroll needed).
**When to use:** Fixed time-window list with known total count.

```typescript
// Page size: 25. Use .range(from, to) in Supabase query.
const from = page * PAGE_SIZE;
const to = from + PAGE_SIZE - 1;
const { data, count } = await this.supabase.client
  .from('alerts')
  .select('*', { count: 'exact' })
  .in('fleet_id', fleetIds)
  .gte('created_at', sevenDaysAgo)
  .order('created_at', { ascending: false })
  .range(from, to);
```

### Anti-Patterns to Avoid

- **Keeping in-memory alert generation:** The existing `AlertService.checkAndCreateAlerts()` generates alerts in-memory from telemetry batches. After Phase 4, the trigger owns detection. The in-memory path should be removed from `processBatch()` in VehicleService (or `checkAndCreateAlerts` becomes a no-op). Do not run both paths — double alerts will appear.
- **Subscribing to alerts table inside AlertsComponent:** Realtime subscription belongs in `AlertService` (singleton). Components derive from signals, not their own subscriptions.
- **SECURITY DEFINER on new trigger function:** The trigger inserts alerts with `fleet_id` from `vehicles` — it must be SECURITY DEFINER so it can read `vehicles` and insert `alerts` when called in the context of the parser's service role. Correct.
- **Forgetting the existing trigger:** Migration 003 created `trigger_generate_alert` calling `generate_alert_if_needed()`. Migration 014 must `DROP TRIGGER trigger_generate_alert ON telemetry_logs` before creating the new one, or use `CREATE OR REPLACE FUNCTION` + `DROP TRIGGER IF EXISTS` + `CREATE TRIGGER`.
- **alerts.id is BIGSERIAL (integer), not UUID:** The existing `acknowledge_alert()` function in migration 003 takes `p_alert_id UUID` — this is a schema mismatch. The alerts.id column is `BIGSERIAL PRIMARY KEY`. Direct UPDATE by integer id is correct; avoid the old RPC.
- **Realtime not enabled on alerts table:** Migration 003 comments indicate Realtime must be enabled via `ALTER PUBLICATION supabase_realtime ADD TABLE alerts`. This must be in Migration 014 or confirmed as already done.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Alert dedup | Custom Angular service dedup logic | PostgreSQL trigger NOT EXISTS check | Client-side dedup misses alerts fired when dashboard is closed |
| DTC translation | DB table lookup or API call | Existing `DtcTranslationService` + `DTC_DATABASE` | Already built; 35 P-codes in DB, generic fallback for unknowns |
| Realtime connection management | Custom WebSocket | `supabase.client.channel().on().subscribe()` | Already used in TelemetryService; same pattern |
| Pagination | Custom offset tracking | Angular Material `mat-paginator` + Supabase `.range()` | mat-paginator handles page events; Supabase range is simple |
| Alert severity badge colors | Custom CSS variables | Angular Material `mat-badge` with color binding | Consistent with project Material usage |
| Bell badge count | Separate Supabase query | `computed(() => dbAlerts().filter(a => !a.acknowledged).length)` | Derived from already-loaded data |

**Key insight:** The detection, storage, and lookup infrastructure all pre-exist. Phase 4 is primarily wiring — the new trigger replaces an old one, and Angular components switch from in-memory to Supabase-backed state.

---

## Common Pitfalls

### Pitfall 1: Existing Trigger Conflict
**What goes wrong:** Migration 014 creates `check_telemetry_alerts()` and a new trigger, but the old `trigger_generate_alert` (from migration 003) still fires `generate_alert_if_needed()`. Both run on every INSERT. The old function queries `telemetry_rules` (which has no rows), so it silently inserts nothing — but it adds overhead and confusion.
**Why it happens:** `CREATE OR REPLACE FUNCTION` replaces the function body but doesn't affect the trigger binding. The old trigger name still exists.
**How to avoid:** Migration 014 must include `DROP TRIGGER IF EXISTS trigger_generate_alert ON telemetry_logs;` before `CREATE TRIGGER`.
**Warning signs:** Two triggers visible in `SELECT * FROM information_schema.triggers WHERE event_object_table = 'telemetry_logs'`.

### Pitfall 2: In-Memory AlertService Double-Alerting
**What goes wrong:** VehicleService.processBatch() still calls `alertService.checkAndCreateAlerts()`, which generates in-memory alerts. Simultaneously, Realtime subscription pushes the same alert from the DB. The /alerts page shows DB alerts; toast shows in-memory alerts. Alert count doubles.
**Why it happens:** The existing AlertService was the source of truth for Phase 3. Phase 4 must transfer ownership.
**How to avoid:** Remove the `alertService.checkAndCreateAlerts(record, ...)` call from `VehicleService.processBatch()`. AlertService.activeAlerts becomes derived from DB alerts only.
**Warning signs:** Toast fires twice for same event; bell count is 2x actual unacknowledged count.

### Pitfall 3: RLS Missing on alerts Table
**What goes wrong:** Alerts table has RLS enabled (migration 002), but no SELECT or UPDATE policy exists for fleet members using `auth.jwt()->>'sub'`. All reads return 0 rows.
**Why it happens:** Migration 002 enables RLS on alerts but the original policies used `auth.uid()` (returns NULL for Auth0 users) or were never created.
**How to avoid:** Migration 014 must add SELECT policy (fleet members can read their fleet's alerts) and UPDATE policy (fleet members can acknowledge their fleet's alerts). Use the `auth.jwt()->>'sub'` pattern from migrations 010/012/013.
**Warning signs:** alertResource.value() returns [] despite alerts existing in DB; no Supabase error (RLS blocks silently).

### Pitfall 4: alerts.id Type Mismatch
**What goes wrong:** The existing `acknowledge_alert(p_alert_id UUID, ...)` RPC in migration 003 takes a UUID parameter, but `alerts.id` is `BIGSERIAL` (integer). Calling this RPC with an integer alert ID will fail.
**Why it happens:** Schema inconsistency between 001 and the function in 003.
**How to avoid:** Do NOT use the `acknowledge_alert()` RPC. Use direct `.update()` with `.eq('id', numericId)` via supabase-js. Migration 014 can optionally drop the stale RPC.
**Warning signs:** RPC call errors with "invalid input syntax for type uuid".

### Pitfall 5: DTC Severity in Trigger vs. Dashboard
**What goes wrong:** The trigger inserts all DTC alerts with `severity = 'warning'`. The existing `AlertService.checkDtcAlerts()` uses `CRITICAL_DTC_CODES` set and `DtcEntry.severity` to escalate some DTCs to `'critical'`. After Phase 4, the trigger is the detection source — it doesn't know the DtcEntry severity.
**Why it happens:** DTC severity metadata lives in the Angular `DTC_DATABASE` static file, not in PostgreSQL.
**How to avoid:** Two options — (1) accept all DTC alerts as `'warning'` severity from trigger (simpler, slight downgrade); (2) embed a hardcoded list of critical DTC codes in the trigger function. Decision is Claude's discretion (D-02/D-03 area). Recommended: option 1 for MVP — DTC critical escalation is a future enhancement. Document the delta.
**Warning signs:** P0300 (random misfire) shows as warning instead of critical in dashboard — acceptable for MVP.

### Pitfall 6: Realtime Publication Not Enabled for alerts
**What goes wrong:** Supabase Realtime subscription to alerts table receives no events even though inserts are happening. `channel.subscribe()` succeeds but payload never arrives.
**Why it happens:** The `alerts` table was not added to the `supabase_realtime` publication. Migration 003 only comments this out ("Run in Supabase SQL Editor").
**How to avoid:** Migration 014 must include: `ALTER PUBLICATION supabase_realtime ADD TABLE alerts;`
**Warning signs:** New alerts appear in DB (visible via Supabase dashboard), but no toast fires and bell count doesn't update.

### Pitfall 7: VehicleWithHealth.alertCount Still Reads In-Memory AlertService
**What goes wrong:** `vehiclesWithHealth` computed in VehicleService uses `this.alertService.activeAlerts().filter(a => a.vehicleId === v.id).length`. After refactor, if `activeAlerts` now reads from DB alerts, the field name used for join (`vehicleId` camelCase vs `vehicle_id` snake_case) must match the SupabaseAlert interface.
**Why it happens:** In-memory Alert model uses `vehicleId`; DB schema uses `vehicle_id`.
**How to avoid:** Either map DB rows to camelCase in AlertService loader, or update the filter in vehiclesWithHealth to use `vehicle_id`. Consistency is critical.

---

## Code Examples

### RLS Policies for alerts table

```sql
-- Source: Migration 010/013 pattern — auth.jwt()->>'sub' for Auth0

-- SELECT: fleet members and org owners can read their alerts
CREATE POLICY "Users can read own fleet alerts" ON alerts
FOR SELECT USING (
  EXISTS (
    SELECT 1 FROM fleet_members fm
    WHERE fm.fleet_id = alerts.fleet_id
      AND fm.user_id = (auth.jwt() ->> 'sub')
  )
  OR
  EXISTS (
    SELECT 1 FROM fleets f
    JOIN organizations o ON o.id = f.organization_id
    WHERE f.id = alerts.fleet_id
      AND o.owner_id = (auth.jwt() ->> 'sub')
  )
);

-- UPDATE: fleet members can acknowledge (set acknowledged=true)
CREATE POLICY "Users can acknowledge own fleet alerts" ON alerts
FOR UPDATE USING (
  EXISTS (
    SELECT 1 FROM fleet_members fm
    WHERE fm.fleet_id = alerts.fleet_id
      AND fm.user_id = (auth.jwt() ->> 'sub')
  )
  OR
  EXISTS (
    SELECT 1 FROM fleets f
    JOIN organizations o ON o.id = f.organization_id
    WHERE f.id = alerts.fleet_id
      AND o.owner_id = (auth.jwt() ->> 'sub')
  )
)
WITH CHECK (acknowledged = true);  -- Can only set acknowledged=true, not revert
```

### Map Marker Severity Update (extend FleetMapComponent)

```typescript
// Source: fleet-map.component.ts getMarkerIcon() — extend existing logic
// Current: colors['alert'] = '#eab308' (yellow for all alerts)
// Phase 4 extension: differentiate critical vs warning

private getMarkerColor(vehicle: VehicleWithHealth): string {
  const state = this.getVehicleState(vehicle);
  if (state !== 'alert') return state === 'running' ? '#84cc16' : '#5a4530';
  // Get highest severity from DB alerts for this vehicle
  const alerts = this.alertService.activeDbAlerts()
    .filter(a => a.vehicle_id === vehicle.id && !a.acknowledged);
  const hasCritical = alerts.some(a => a.severity === 'critical');
  return hasCritical ? '#ef4444' : '#eab308'; // red : yellow
}
```

### Bell Icon in HeaderComponent (already exists — verify template)

```typescript
// header.component.ts already has:
readonly criticalAlertCount = this.alertService.criticalAlertCount;
readonly activeAlertCount = this.alertService.activeAlertCount;
// After refactor: these signals must read from DB alerts, not in-memory
// New signal needed: unacknowledgedCount (for bell badge)
readonly unacknowledgedCount = this.alertService.unacknowledgedCount;
```

### SupabaseAlert Interface (extend alert.model.ts)

```typescript
// New interface for DB rows (snake_case from Supabase)
export interface SupabaseAlert {
  id: number;                    // BIGSERIAL
  vehicle_id: string;
  fleet_id: string;
  severity: AlertSeverity;
  code: string;
  message: string;
  dtc_codes: string[] | null;
  lat: number | null;
  lng: number | null;
  acknowledged: boolean;
  acknowledged_by: string | null;
  acknowledged_at: string | null;
  created_at: string;
  resolved_at: string | null;
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `generate_alert_if_needed()` uses telemetry_rules table | Replace with `check_telemetry_alerts()` hardcoded thresholds | Migration 014 | Alerts actually fire for MVP |
| AlertService generates alerts in-memory from telemetry | AlertService reads from Supabase alerts table + Realtime | Phase 4 | Alerts persist; visible on reload; multi-tab consistent |
| /alerts page shows in-memory activeAlerts | /alerts page queries Supabase 7-day window paginated | Phase 4 | History survives page refresh |
| VehicleWithHealth.alertCount from in-memory | alertCount from Supabase DB alerts | Phase 4 | Accurate count even on first load |
| Map marker alert state: single yellow color | Map marker: red (critical) / yellow (warning) differentiation | Phase 4 | D-06 compliance |

**Deprecated/outdated:**
- `AlertService.checkAndCreateAlerts()`: Call site in `VehicleService.processBatch()` must be removed; method itself can be kept as dead code or removed.
- `acknowledge_alert(p_alert_id UUID, p_user_id UUID)` RPC (migration 003): Wrong type for alert id; don't use. Replace with direct UPDATE.
- `CRITICAL_DTC_CODES` set in AlertService: No longer needed for detection (trigger handles it); retained for potential future severity escalation.

---

## Open Questions (RESOLVED)

1. **DTC severity in trigger (critical vs warning)**
   - What we know: DTC_DATABASE has severity per code; trigger can't access it
   - What's unclear: Does MVP need P0300 to show 'critical' severity, or is 'warning' for all DTCs acceptable?
   - RESOLVED: Default all DTC trigger alerts to 'warning'; add critical DTC list to trigger in a future enhancement. 04-01 Task 2 implements this.

2. **Realtime subscription scope: single channel vs per-fleet**
   - What we know: supabase-js v2 supports filter on single column; fleet_id is available
   - What's unclear: `in` filter syntax for multiple fleet IDs in Realtime — `fleet_id=in.(uuid1,uuid2)` — needs verification against installed supabase-js 2.39.0
   - RESOLVED: Subscribe without server-side filter; check fleet_id in handler. 04-02 Task 1 implements this.

3. **Acknowledged alerts in /alerts page UX**
   - What we know: D-05c says acknowledged alerts remain visible but visually dimmed
   - What's unclear: Should the default view show both acknowledged and unacknowledged, or default to unacknowledged-only with a toggle?
   - RESOLVED: Show all (7-day window), unacknowledged first (ORDER BY acknowledged ASC, created_at DESC), acknowledged rows dimmed via CSS opacity. 04-02 Task 2 implements this.

---

## Environment Availability

Step 2.6: SKIPPED (no external tools beyond Supabase and npm packages already installed; all dependencies confirmed present in package.json).

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Jest (dashboard: jest + jest.config.js) |
| Config file | `packages/dashboard/jest.config.js` |
| Quick run command | `npm test --prefix packages/dashboard -- --testPathPattern="alert"` |
| Full suite command | `npm test --prefix packages/dashboard` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DTC-01 | Trigger inserts alert row for each DTC code in telemetry_logs INSERT | Manual (DB trigger, no Jest) | supabase db test / psql smoke | ❌ Wave 0 |
| DTC-02 | DtcTranslationService.translate() returns description for known P-code | unit | `npm test --prefix packages/dashboard -- --testPathPattern="dtc-translation"` | ❌ Wave 0 |
| DTC-03 | AlertsComponent shows paginated list filtered to last 7 days | unit | `npm test --prefix packages/dashboard -- --testPathPattern="alerts.component"` | ❌ Wave 0 |
| BATT-01 | Trigger inserts BATTERY_WARNING when voltage < 11.8V | Manual (DB trigger) | supabase SQL smoke | ❌ Wave 0 |
| BATT-02 | AlertService.unacknowledgedCount updates when Realtime INSERT arrives | unit (mock Supabase) | `npm test --prefix packages/dashboard -- --testPathPattern="alert.service"` | ❌ Wave 0 |
| BATT-03 | VehicleCard shows battery voltage from latestTelemetry | unit | `npm test --prefix packages/dashboard -- --testPathPattern="vehicle-card"` | existing spec |
| COOL-01 | Trigger inserts COOLANT_WARNING when temp > 100°C | Manual (DB trigger) | supabase SQL smoke | ❌ Wave 0 |
| COOL-02 | Toast shows when new alert arrives via Realtime | unit (mock Realtime) | `npm test --prefix packages/dashboard -- --testPathPattern="alert.service"` | ❌ Wave 0 |
| COOL-03 | VehicleCard shows coolant temp from latestTelemetry | unit | existing vehicle-card spec | existing spec |

### Sampling Rate
- **Per task commit:** `npm test --prefix packages/dashboard -- --testPathPattern="alert" --passWithNoTests`
- **Per wave merge:** `npm test --prefix packages/dashboard`
- **Phase gate:** Full suite green before `/gsd-verify-work`

### Wave 0 Gaps
- [ ] `packages/dashboard/src/app/core/services/alert.service.spec.ts` — covers BATT-02, COOL-02 (Realtime mock, unacknowledgedCount)
- [ ] `packages/dashboard/src/app/features/alerts/alerts.component.spec.ts` — covers DTC-03 (paginated list, 7-day filter)
- [ ] `packages/dashboard/src/app/core/services/dtc-translation.service.spec.ts` — covers DTC-02

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | Auth handled by Phase 2 auth guard |
| V3 Session Management | no | Session handled by Phase 2 |
| V4 Access Control | yes | RLS on alerts table: fleet_members JOIN with auth.jwt()->>'sub' |
| V5 Input Validation | yes | Alert acknowledgment UPDATE: WITH CHECK (acknowledged = true) prevents revert |
| V6 Cryptography | no | No new crypto |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Cross-fleet alert read | Information Disclosure | RLS SELECT policy: alerts.fleet_id → fleet_members JOIN |
| Acknowledging another fleet's alert | Tampering | RLS UPDATE policy: same fleet_members JOIN |
| Alert spam via rapid telemetry inserts | Denial of Service | 1-hour dedup window in trigger (D-08) |
| Trigger reading vehicles without auth | Elevation of Privilege | Trigger function is SECURITY DEFINER — reads vehicles/inserts alerts as postgres superuser |
| Reverting acknowledged alert | Tampering | RLS UPDATE WITH CHECK (acknowledged = true) — can only set true, not false |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Supabase Realtime `in` filter for multi-fleet requires format `fleet_id=in.(uuid1,uuid2)` — exact syntax unverified for supabase-js 2.39.0 | Pattern 2 | Subscription fails silently; no real-time delivery. Mitigation: client-side filter |
| A2 | PL/pgSQL FOREACH with inner DECLARE sub-block compiles correctly in Supabase managed Postgres | Pattern 3 | Trigger fails to compile; no DTC alerts. Mitigation: use a separate helper function for DTC loop |
| A3 | Supabase Realtime is not yet enabled for the `alerts` table in the production project | Pitfall 6 | If already enabled, migration 014's ADD TABLE is a no-op (safe). If not enabled, critical for real-time delivery |
| A4 | AlertsComponent currently reads from in-memory AlertService.activeAlerts (not Supabase) | Architecture | Confirmed by reading alerts.component.ts line 36 — `[VERIFIED: codebase]` — not an assumption |
| A5 | DTC severity for trigger defaults to 'warning' for all codes (critical escalation not in trigger) | Pitfall 5 | Some critical DTCs show as warning — acceptable for MVP per recommendation |

---

## Sources

### Primary (HIGH confidence — verified from codebase)
- `packages/supabase/migrations/001_initial_schema.sql` — alerts table schema (BIGSERIAL id, severity/code/message/acknowledged columns, existing indexes)
- `packages/supabase/migrations/003_functions_triggers.sql` — existing trigger `trigger_generate_alert` on telemetry_logs; `acknowledge_alert()` RPC (type mismatch found)
- `packages/supabase/migrations/010_fix_rls_policies.sql`, `012_telemetry_rls_and_rpc.sql`, `013_fix_telemetry_rls_owner_check.sql` — auth.jwt()->>'sub' RLS pattern
- `packages/dashboard/src/app/core/services/alert.service.ts` — in-memory AlertService; checkAndCreateAlerts(), pushAlert(), all computed signals
- `packages/dashboard/src/app/core/services/vehicle.service.ts` — resource() pattern, processBatch(), alertService.checkAndCreateAlerts() call site
- `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts` — getMarkerIcon(), getVehicleState(), alertCount usage
- `packages/dashboard/src/app/layout/header/header.component.ts` — criticalAlertCount, activeAlertCount signals already wired
- `packages/dashboard/src/app/features/alerts/alerts.component.ts` — current in-memory implementation; needs full Supabase rewrite
- `packages/dashboard/src/app/shared/data/dtc-database.ts` — 35 P-codes in existing static DB; generic fallback for unknowns
- `packages/dashboard/src/app/core/services/dtc-translation.service.ts` — translate() and isKnownCode() methods

### Secondary (MEDIUM confidence)
- Angular Material mat-badge, mat-paginator docs — [ASSUMED: consistent with Angular Material 21 API, same as 17-20]
- Supabase Realtime postgres_changes subscription pattern — [ASSUMED: consistent with supabase-js v2 pattern used in TelemetryService]

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all verified from package.json and existing codebase
- Architecture: HIGH — trigger pattern and Angular service pattern fully verified from existing migrations and services
- Pitfalls: HIGH — all derived from concrete code reading (type mismatches, existing trigger conflicts found directly)
- Supabase Realtime filter syntax: LOW — not verified against supabase-js 2.39.0 changelog

**Research date:** 2026-05-14
**Valid until:** 2026-06-14 (Angular/Supabase stable; trigger SQL is timeless)
