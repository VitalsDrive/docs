---
phase: 04-alert-system
reviewed: 2026-05-15T00:05:00Z
depth: standard
files_reviewed: 13
files_reviewed_list:
  - packages/supabase/migrations/014_alert_trigger_and_rls.sql
  - packages/dashboard/src/app/core/models/alert.model.ts
  - packages/dashboard/src/app/core/services/dtc-translation.service.spec.ts
  - packages/dashboard/src/app/core/services/alert.service.ts
  - packages/dashboard/src/app/core/services/alert.service.spec.ts
  - packages/dashboard/src/app/core/services/vehicle.service.ts
  - packages/dashboard/src/app/features/alerts/alerts.component.ts
  - packages/dashboard/src/app/features/alerts/alerts.component.html
  - packages/dashboard/src/app/features/alerts/alerts.component.css
  - packages/dashboard/src/app/features/alerts/alerts.component.spec.ts
  - packages/dashboard/src/app/layout/header/header.component.ts
  - packages/dashboard/src/app/layout/header/header.component.html
  - packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts
findings:
  critical: 4
  warning: 6
  info: 3
  total: 13
status: issues_found
---

# Phase 4: Code Review Report

**Reviewed:** 2026-05-15T00:05:00Z
**Depth:** standard
**Files Reviewed:** 13
**Status:** issues_found

## Summary

Phase 4 adds a PostgreSQL trigger-based alert detection system (migration 014), an `AlertService` refactor to `resource()` + Supabase Realtime, an `/alerts` page, a header bell badge, and severity-aware fleet map markers. The architecture is sound and the Auth0 JWT pattern (`auth.jwt()->>'sub'`) is applied consistently. However, four critical defects were found that will cause incorrect runtime behavior: a Realtime channel that never actually subscribes when `fleetIds` changes after initial setup, a trigger-level dedup logic gap that allows `BATTERY_WARNING` to fire even when `BATTERY_CRITICAL` is already active, a test suite that silently bypasses Supabase mock chaining, and a missing `acknowledged_at` optimistic update creating a stale signal state. Six warnings address edge cases and maintainability concerns.

---

## Critical Issues

### CR-01: Realtime subscription captures stale `fleetIds` closure — new fleets never receive alerts

**File:** `packages/dashboard/src/app/core/services/alert.service.ts:96-123`

**Issue:** The `effect()` in the constructor fires once when `alertResource` first resolves, captures `fleetIds` at that moment, sets `alertSubscribed = true`, and never re-runs. If the user switches organization or new fleets are added after initial load, `subscribeToAlerts()` is never called again with updated fleet IDs. The in-memory filter at line 115 (`fleetIds.includes(alert.fleet_id)`) then silently drops Realtime inserts for the new fleets — no error, no indicator. The user sees no real-time alerts for newly-joined fleets until a full page reload.

**Fix:** Remove the `alertSubscribed` guard and instead track the last-subscribed fleet set, or re-subscribe on fleet change by unsubscribing the old channel first:

```typescript
effect(() => {
  const fleetIds = this.fleetService.fleets().map((f) => f.id);
  if (fleetIds.length === 0) return;
  if (this.alertResource.status() !== 'resolved') return;
  // Unsubscribe old channel, re-subscribe with fresh fleet list
  if (this.alertChannel) {
    this.supabase.client.removeChannel(this.alertChannel);
    this.alertChannel = null;
  }
  this.subscribeToAlerts(fleetIds);
});
```

Also remove the `if (this.alertChannel) return;` guard in `subscribeToAlerts()` since the caller now ensures cleanup first.

---

### CR-02: SQL trigger dedup gap — `BATTERY_WARNING` fires even when `BATTERY_CRITICAL` is already active

**File:** `packages/supabase/migrations/014_alert_trigger_and_rls.sql:78-98`

**Issue:** When `voltage < 11.5`, a `BATTERY_CRITICAL` alert is inserted. On the very next telemetry row (still `< 11.5`), the critical dedup fires correctly. But if voltage recovers to `11.5 <= v < 11.8` on the next row, the `BATTERY_WARNING` branch executes. Its dedup query checks:

```sql
AND code IN ('BATTERY_WARNING', 'BATTERY_CRITICAL')
```

This looks correct — but the critical alert from the prior row IS unacknowledged and within 1 hour, so warning is suppressed. **This is actually correct behavior.** However there is a real gap in the opposite direction: if a `BATTERY_WARNING` was inserted, and voltage then drops below `11.5`, the `BATTERY_CRITICAL` branch only checks `code = 'BATTERY_CRITICAL'` (line 62) — it does NOT suppress if a `BATTERY_WARNING` for the same vehicle is already active. This means both `BATTERY_WARNING` and `BATTERY_CRITICAL` are simultaneously active and unacknowledged for the same vehicle, which the dashboard does not handle gracefully (two rows shown, confusing UX). The same structural problem exists for `COOLANT_WARNING` → `COOLANT_CRITICAL` escalation (line 109 only checks `code = 'COOLANT_CRITICAL'`).

**Fix:** Add the lower-severity code to the critical dedup guard:

```sql
-- BATTERY_CRITICAL dedup (line 59-65)
AND code IN ('BATTERY_WARNING', 'BATTERY_CRITICAL')

-- COOLANT_CRITICAL dedup (line 109-115)
AND code IN ('COOLANT_WARNING', 'COOLANT_CRITICAL')
```

---

### CR-03: `acknowledgeAlert` optimistic update does not set `acknowledged_at` — signal state diverges from DB

**File:** `packages/dashboard/src/app/core/services/alert.service.ts:134-136`

**Issue:** The Supabase `UPDATE` at line 130 sets both `acknowledged: true` and `acknowledged_at: new Date().toISOString()`. The optimistic signal update at line 134-136 only flips `acknowledged: true` but leaves `acknowledged_at: null` on the in-memory object:

```typescript
alerts.map((a) => (a.id === alertId ? { ...a, acknowledged: true } : a))
```

`SupabaseAlert.acknowledged_at` remains `null` in the signal. Any component that renders `acknowledged_at` (e.g., a future audit trail or tooltip) will show `null` until `alertResource` reloads. More critically, if `alertResource.reload()` is not called after acknowledge, the signal is permanently stale for this field. The alerts page currently does not trigger a reload after acknowledge.

**Fix:**

```typescript
this._dbAlerts.update((alerts) =>
  alerts.map((a) =>
    a.id === alertId
      ? { ...a, acknowledged: true, acknowledged_at: new Date().toISOString() }
      : a,
  ),
);
```

---

### CR-04: Test stub `acknowledgeAlert` calls `eqMock` directly instead of chaining from `update` — test does not verify real Supabase call chain

**File:** `packages/dashboard/src/app/core/services/alert.service.spec.ts:78-88`

**Issue:** The stub's `acknowledgeAlert` (lines 78-88) calls `updateMock(...)` and `eqMock(...)` as independent function calls — not as a chain `fromMock('alerts').update(...).eq(...)`. The real `AlertService.acknowledgeAlert` chains `.from().update().eq()`. The test at line 175 asserts `updateMock` and `eqMock` were called, but because the stub wires them independently, `eqMock` is never actually called on the result of `updateMock`. The assertion on line 185 (`expect(svc.eqMock).toHaveBeenCalledWith('id', 42)`) passes because `eqMock` is called directly at line 83 (`eqMock('id', alertId)`), not because chaining works. This means the test provides zero confidence that the real `.update().eq()` chain is correct.

**Fix:** The stub's `acknowledgeAlert` should mirror the real implementation exactly:

```typescript
async function acknowledgeAlert(alertId: number): Promise<void> {
  const { error } = await fromMock('alerts')
    .update({ acknowledged: true, acknowledged_at: new Date().toISOString() })
    .eq('id', alertId);
  if (error) throw error;
  _dbAlerts.update((a) => a.map((x) => (x.id === alertId ? { ...x, acknowledged: true } : x)));
}
```

Then make `updateMock` return `{ eq: eqMock }` and `eqMock` return `Promise.resolve({ error: null })` so the chain is exercised end-to-end.

---

## Warnings

### WR-01: `DtcTranslationService` instantiated with `new` instead of `inject()` — bypasses Angular DI

**File:** `packages/dashboard/src/app/core/services/alert.service.ts:19`

**Issue:** `private readonly dtcService = new DtcTranslationService();` bypasses Angular's dependency injection. If `DtcTranslationService` is ever changed to `inject()` other services internally, this will throw at runtime. It also prevents mocking in tests via the DI system.

**Fix:** Use `inject(DtcTranslationService)` consistent with the rest of the service's injection pattern.

---

### WR-02: `checkAndCreateAlerts` declared as dead code but still exported and callable — creates maintenance trap

**File:** `packages/dashboard/src/app/core/services/alert.service.ts:152-157`

**Issue:** The method is documented as dead code (comment: "NOTE: This method is retained as dead code after plan 04-02"). It is a public method that duplicates threshold logic in the SQL trigger, with different thresholds (coolant critical at 105°C vs trigger's 110°C, battery critical at 12.0V vs trigger's 11.5V). Any caller that accidentally invokes it will create in-memory alerts with incorrect thresholds. The voltageHistory map is still populated by `updateVoltageHistory()` even though nothing uses it for DB-backed alerts.

**Fix:** Mark the method and its private helpers (`checkCoolantAlert`, `checkBatteryAlert`, `checkDtcAlerts`, `createAlert`, `updateVoltageHistory`, `predictVoltageIn2Hours`, `voltageHistory`) with `@deprecated` JSDoc and schedule removal in the next phase. Add a runtime guard or remove now if no callers exist.

---

### WR-03: `calculateHealthScore` in `VehicleService` accesses `telemetry.voltage` and `telemetry.coolant_temp` without null guard — crashes on partial telemetry

**File:** `packages/dashboard/src/app/core/services/vehicle.service.ts:319-323`

**Issue:** `mapDbRecord()` sets `voltage: Number(raw['voltage'] ?? 0)` and `coolant_temp: Number(raw['coolant_temp'] ?? raw['temp'] ?? 0)`, so they default to `0`. However `calculateHealthScore` is also called with `latest = undefined` (guarded at line 317). If `voltage` is `0` (device not yet reporting), the health score deducts 50 points (`< 12.4` → -20, `< 12.0` → -30) incorrectly, showing vehicles with no telemetry as critically unhealthy. Similarly `dtc_codes.length` at line 323 would throw if `dtc_codes` were `undefined`, though `mapDbRecord` always sets it to `[]`.

**Fix:** Add a guard for zero/missing voltage in the health score:

```typescript
if (telemetry.voltage > 0 && telemetry.voltage < 12.4) score -= 20;
if (telemetry.voltage > 0 && telemetry.voltage < 12.0) score -= 30;
```

---

### WR-04: `alerts.component.html` shows raw `vehicle_id` UUID instead of vehicle name

**File:** `packages/dashboard/src/app/features/alerts/alerts.component.html:84`

**Issue:** `{{ alert.vehicle_id }}` renders the raw UUID (e.g., `3f2c1a4b-...`) to the user. `SupabaseAlert` has no `vehicleName` field. `VehicleService.vehiclesWithHealth()` is available but not injected into `AlertsComponent`. A UUID is not user-readable for fleet operators.

**Fix:** Inject `VehicleService` and create a computed lookup helper:

```typescript
private readonly vehicleService = inject(VehicleService);

getVehicleName(vehicleId: string): string {
  return this.vehicleService.vehiclesWithHealth()
    .find(v => v.id === vehicleId)
    ? getVehicleDisplayName(this.vehicleService.vehiclesWithHealth().find(v => v.id === vehicleId)!)
    : vehicleId;
}
```

Or add a `vehicleName` field via a Supabase join in the `alertResource` query.

---

### WR-05: `acknowledge()` in `AlertsComponent` swallows errors — user gets no feedback on failure

**File:** `packages/dashboard/src/app/features/alerts/alerts.component.ts:81-83`

**Issue:** `acknowledge(alert)` calls `this.alertService.acknowledgeAlert(alert.id)` which throws on Supabase error. The `async` method has no `try/catch`. If the RLS policy denies the update (e.g., user is not a fleet member), or if the network fails, the error is swallowed silently — the optimistic signal flip already occurred (CR-03 is separate), and the user sees no error toast.

**Fix:**

```typescript
async acknowledge(alert: SupabaseAlert): Promise<void> {
  try {
    await this.alertService.acknowledgeAlert(alert.id);
  } catch (err) {
    console.error('Failed to acknowledge alert', err);
    // Show error toast or snackbar
  }
}
```

---

### WR-06: `getVehicleState()` returns `'offline'` before checking `alertCount` when engine is off but alerts are active — alert markers not shown

**File:** `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts:208-215`

**Issue:** State priority in `getVehicleState()`:
1. stale (>15 min)
2. offline (`status === 'inactive'`)
3. alert (`alertCount > 0`)
4. running (`engine_on`)
5. offline (fallback)

A vehicle that is `active` status, engine off, and has active alerts falls through to step 3 correctly. But a vehicle that is parked with `engine_on = false` and `alertCount > 0` correctly shows as `'alert'`. The actual bug: `status === 'inactive'` check at step 2 short-circuits before `alertCount` check. An inactive vehicle with unacknowledged alerts (e.g., issued before it was deactivated) will show as `'offline'` (brown marker) and the alert severity coloring will never apply — the `getMarkerColor` call for `'offline'` returns `'#5a4530'` without checking alerts. Fleet operators cannot visually identify alerts on inactive vehicles.

**Fix:** Swap the priority order — check `alertCount` before `status === 'inactive'`:

```typescript
private getVehicleState(vehicle: VehicleWithHealth): 'stale' | 'offline' | 'running' | 'alert' {
  const ts = vehicle.latestTelemetry?.timestamp;
  if (ts && Date.now() - new Date(ts).getTime() > 15 * 60 * 1000) return 'stale';
  if (vehicle.alertCount > 0) return 'alert';      // moved up
  if (vehicle.status === 'inactive') return 'offline';
  if (vehicle.latestTelemetry?.engine_on) return 'running';
  return 'offline';
}
```

---

## Info

### IN-01: `alertResource` query fetches up to 200 rows with no server-side pagination — page 8+ always empty

**File:** `packages/dashboard/src/app/core/services/alert.service.ts:81`

**Issue:** `.range(0, 199)` caps the server response at 200 rows. `AlertsComponent` paginates client-side (25 per page = 8 pages max). For fleets with >200 alerts in 7 days this silently truncates older alerts. The `count: 'exact'` is requested but never surfaced to the component for display.

**Fix:** Either implement server-side pagination driven by `currentPage` signal, or raise the range cap and document the limit. Minimum: display a "showing first 200 alerts" notice when truncated.

---

### IN-02: `filter-chip-count` CSS class defined but never used in template

**File:** `packages/dashboard/src/app/features/alerts/alerts.component.css:54-66`

**Issue:** `.filter-chip-count` styles exist for a badge/count bubble on filter chips, but the HTML template does not render counts on the chips. Dead CSS.

**Fix:** Either add count badges to filter chips (useful UX) or remove the dead CSS.

---

### IN-03: `alerts.component.spec.ts` file listed in scope but not present — no component-level tests

**File:** `packages/dashboard/src/app/features/alerts/alerts.component.spec.ts`

**Issue:** The file was listed in the review scope but does not exist on disk. The alerts component has zero unit or integration tests: no coverage for filter logic, pagination, acknowledge button interaction, loading/error state rendering, or empty state conditions.

**Fix:** Create `alerts.component.spec.ts` with at minimum: filter computed signal tests, `onPageChange` resets page, `acknowledge` calls `alertService.acknowledgeAlert`, and loading/error/empty template state checks using Angular `TestBed`.

---

_Reviewed: 2026-05-15T00:05:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
