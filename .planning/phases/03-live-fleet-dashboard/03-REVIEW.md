---
phase: 03-live-fleet-dashboard
reviewed: 2026-05-14T00:00:00Z
depth: standard
files_reviewed: 15
files_reviewed_list:
  - packages/supabase/migrations/012_telemetry_rls_and_rpc.sql
  - packages/dashboard/src/app/core/services/vehicle.service.ts
  - packages/dashboard/src/app/core/services/vehicle.service.spec.ts
  - packages/dashboard/src/app/core/services/alert.service.ts
  - packages/dashboard/src/app/layout/shell/shell.component.ts
  - packages/dashboard/src/app/layout/shell/shell.component.html
  - packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts
  - packages/dashboard/src/app/features/fleet-map/fleet-map.component.html
  - packages/dashboard/src/app/features/fleet-map/fleet-map.component.spec.ts
  - packages/dashboard/src/app/features/dashboard/dashboard.component.ts
  - packages/dashboard/src/app/features/dashboard/dashboard.component.html
  - packages/dashboard/src/app/features/dashboard/dashboard.component.spec.ts
  - packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-grid.component.ts
  - packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-grid.component.html
  - packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-card/vehicle-card.component.spec.ts
findings:
  critical: 1
  warning: 7
  info: 4
  total: 12
status: issues_found
---

# Phase 3: Code Review Report

**Reviewed:** 2026-05-14T00:00:00Z
**Depth:** standard
**Files Reviewed:** 15
**Status:** issues_found

## Summary

Phase 3 wires the live fleet dashboard: a corrected RLS policy + `get_latest_telemetry` RPC, a `resource()`-driven `VehicleService`, the alert engine, the Leaflet fleet map, and the dashboard/grid components.

One BLOCKER: the connection-restored toast logic in `ShellComponent` is dead code because of a state-transition mismatch with `TelemetryService`. Seven warnings cover effect-driven subscription leaks, an XSS-capable Leaflet popup, an unused effect in `VehicleService`, a stale-marker effect that never clears markers when the fleet empties, and several robustness gaps. The spec files are RED-stub scaffolding that do not actually exercise the implementation — flagged as warnings since they provide false coverage confidence.

## Critical Issues

### CR-01: Connection-restored toast never fires — state transition mismatch

**File:** `packages/dashboard/src/app/layout/shell/shell.component.ts:74`
**Issue:** The reconnect toast requires `this.previousConnectionStatus === 'disconnected'` at the moment status becomes `'connected'`. But `TelemetryService` transitions `disconnected → reconnecting → connected` (telemetry.service.ts:36 sets `'reconnecting'` before line 50 sets `'connected'`). When the effect sees `'connected'`, `previousConnectionStatus` is `'reconnecting'`, so the branch never executes. The "Live data restored." toast (D-20 requirement) is dead code. Additionally the `'disconnected'` branch fires the "Connection lost" toast on the `connected/reconnecting → disconnected` edge but never on `reconnecting` itself, so a fleet that goes straight to `reconnecting` shows no toast at all.
**Fix:**
```ts
effect(() => {
  const status = this.telemetryService.connectionStatus();
  const prev = this.previousConnectionStatus;
  if (prev !== null && prev !== status) {
    if (status === 'disconnected' || status === 'reconnecting') {
      this.alertService.pushAlert('Connection lost — reconnecting…', 'warning');
    } else if (status === 'connected' && prev !== 'connected') {
      this.alertService.pushAlert('Live data restored.', 'info');
    }
  }
  this.previousConnectionStatus = status;
});
```

## Warnings

### WR-01: effect() re-subscribes to telemetryBatch$ on every resolved transition without cleaning prior subscription

**File:** `packages/dashboard/src/app/core/services/vehicle.service.ts:134-142`
**Issue:** The constructor `effect()` runs every time `vehicleResource.status()` becomes `'resolved'` (e.g. after `reload()` or any param change that re-resolves). Each run calls `.subscribe(...)` again. The comment claims `reloadVehicles$.next()` cancels the prior subscription, but `reloadVehicles$.next()` is called *before* the new `.subscribe()` in the same effect body — it tears down the subscription created in the *previous* effect run, which is correct only if the effect re-runs. However `subscribeToFleet()` is also called every time, and `TelemetryService` may stack channels. More importantly, if the effect re-runs while status stays `'resolved'` (signal read inside still triggers), nothing changes — but on genuine reloads you get N `subscribeToFleet()` calls. Verify `subscribeToFleet()` is idempotent; if not, this leaks realtime channels.
**Fix:** Guard with a "already subscribed" flag, or move the subscription out of the effect into `ngOnInit`/constructor once, and let `reloadVehicles$` only gate batch processing. Confirm `TelemetryService.subscribeToFleet()` is idempotent.

### WR-02: XSS via vehicle display name in Leaflet popup HTML

**File:** `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts:208-221`
**Issue:** `buildPopupContent()` interpolates `getVehicleDisplayName(vehicle)` directly into an HTML string passed to `marker.bindPopup()`. Vehicle make/model/nickname are user-editable fields (Phase 2 CRUD). A nickname like `<img src=x onerror=alert(1)>` executes in the popup. Same pattern in `getMarkerIcon()` `html` (line 184) — though that interpolates only `state`/`color`/`opacity` which are internally derived, so lower risk. The popup is the live injection point.
**Fix:** Escape interpolated values, or build the popup DOM with `document.createElement` and `textContent`. Minimum: an `escapeHtml()` helper applied to `getVehicleDisplayName(vehicle)`, `temp`, and `volt`.

### WR-03: Unused `resource` import / `error` + `isLoading` deprecated signals never updated

**File:** `packages/dashboard/src/app/core/services/vehicle.service.ts:38-41`
**Issue:** `isLoading` and `error` are writable signals marked deprecated, but nothing in this file (or, per the resource-driven design, anywhere) sets them anymore. Components that still read `vehicleService.isLoading()` / `.error()` will silently get stale `false` / `null` forever — a latent bug if any consumer remains. Either remove them and fix consumers, or wire them to mirror `vehicleResource`.
**Fix:** `grep` for `.isLoading` / `.error` usages on `VehicleService`; delete the dead signals or back them with `computed(() => this.vehicleResource.isLoading())`.

### WR-04: Fleet-map stale/marker effect never clears markers when fleet becomes empty

**File:** `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts:65-71`
**Issue:** The effect only calls `updateMarkers(vehicles)` when `vehicles.length > 0`. If the fleet transitions to empty (org switch, all vehicles deactivated), `updateMarkers` is never invoked, so the "remove markers no longer present" loop (lines 141-146) never runs and stale markers stay on the map. The empty-state overlay (`fleet-map.component.html:35`) renders on top, but the markers persist underneath and reappear if the overlay condition flickers.
**Fix:** Drop the `vehicles.length > 0` guard, or add an `else` branch that clears `markerMap`. `updateMarkers([])` already handles the empty case correctly.

### WR-05: `initMap()` runs `updateMarkers` before throttled effect — possible double initial render and `initialFitDone` race

**File:** `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts:96, 65-71`
**Issue:** `initMap()` (in `ngAfterViewInit`) calls `updateMarkers(this.vehicles())` synchronously. The constructor `effect()` also schedules `updateMarkers` 1s after any `vehicles()` change. If telemetry arrives between effect registration and `ngAfterViewInit`, `this.map` is still null so the effect's `updateMarkers` is skipped, but the effect's `setTimeout` was still scheduled and will fire later. Mostly benign, but `initialFitDone` can be set by the synchronous call before any real GPS data exists if a vehicle briefly has coords, permanently disabling auto-fit. Low-severity but worth a guard.
**Fix:** Only set `initialFitDone` once markers for the *resolved* fleet are placed; consider gating the initial `updateMarkers` call on `vehicleResource.status() === 'resolved'`.

### WR-06: `predictVoltageIn2Hours` extrapolates 120 readings ahead from as few as 5 samples

**File:** `packages/dashboard/src/app/core/services/alert.service.ts:270-289`
**Issue:** With `history.length >= 5`, the linear regression extrapolates to `n + 120`. Five noisy voltage samples produce a wildly unstable slope; extrapolating 120 steps amplifies noise into a "battery may not start tonight" critical-adjacent warning. Also the comment assumes readings are ~1/min, but `updateVoltageHistory` appends one entry per telemetry batch with no time basis — batch cadence is not guaranteed 1/min, so "2 hours" is fictional. False-positive alert risk.
**Fix:** Require a larger minimum sample count (e.g. 15-20), and/or compute slope per real elapsed time using telemetry timestamps instead of array index. At minimum, cap extrapolation distance relative to sample count.

### WR-07: Spec files are RED stubs that assert the *absence* of the feature — no real coverage

**File:** `packages/dashboard/src/app/core/services/vehicle.service.spec.ts:227-247`, `dashboard.component.spec.ts:47-91`, `vehicle-card.component.spec.ts:50-86`, `fleet-map.component.spec.ts:52-57`
**Issue:** These specs assert things like `expect(telemetry.subscribeToFleet).not.toHaveBeenCalled()` and `expect(hasVehicleResource).toBe(false)` against hand-rolled mocks — they were "Wave 0 RED stubs" and were never converted to GREEN. They will *pass* against the shipped implementation while testing nothing about it (`fleet-map.component.spec.ts` tests a *copy* of `getVehicleState` inlined in the spec, not the component's method, so the real implementation is entirely untested). This is false coverage: CI is green but the resource() chain, effect bridge, and stale logic have zero real assertions.
**Fix:** Convert to TestBed-based tests that exercise the actual classes, or delete the stubs so coverage reports are honest. The skipped test at `vehicle.service.spec.ts:202` explicitly acknowledges this gap.

## Info

### IN-01: `mapDbRecord` only applied to RPC seed path, not realtime batch path

**File:** `packages/dashboard/src/app/core/services/vehicle.service.ts:91-93, 281-306`
**Issue:** `telemetryResource` loader maps RPC rows through `mapDbRecord()` (normalizing `coolant_temp`/`voltage` with `Number(... ?? 0)`), but `processBatch()` stores realtime records raw. If realtime records ever have `null` voltage/coolant, `calculateHealthScore` does arithmetic on `null` (`null < 12.4` is `true` in JS → unexpected score deduction). Inconsistent normalization between the two ingestion paths.
**Fix:** Run realtime batch records through `mapDbRecord()` too, or assert the realtime payload shape matches `TelemetryRecord`.

### IN-02: Inline styles scattered across templates

**File:** `packages/dashboard/src/app/features/fleet-map/fleet-map.component.html:19-55`, `dashboard.component.html:33-78`, `shell.component.html:19-24`
**Issue:** Large blocks of inline `style="..."` (positioning, colors, spacing) instead of CSS classes. Harder to maintain, no theming consistency, and bypasses the `--vd-*` token discipline used elsewhere. Not a bug but a quality drift.
**Fix:** Move to component CSS files using existing design tokens.

### IN-03: `selectedVehicleId` duplicated between `VehicleService` and `FleetMapComponent`

**File:** `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts:54`
**Issue:** `VehicleService` exposes `selectedVehicleId` + `selectVehicle()` + `selectedVehicle`, but `FleetMapComponent` defines its own local `selectedVehicleId` signal and `selectedVehicle` computed. Two sources of truth for "which vehicle is selected" — clicking a marker updates only the local one; the service's selection is never set. Confusing for any future cross-component selection sync.
**Fix:** Use `vehicleService.selectVehicle()` / `vehicleService.selectedVehicle` consistently, or document why the map keeps a separate local selection.

### IN-04: Magic numbers for thresholds scattered without named constants

**File:** `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts:201`, `alert.service.ts:119,130,163,202`
**Issue:** `15 * 60 * 1000` (stale threshold), `12.4`/`12.0`/`12.6` (voltage thresholds), `100`/`105` (coolant thresholds), `1000` (marker throttle) are inline literals. The spec file even defines `STALE_THRESHOLD_MINUTES = 15` separately, so the constant exists conceptually but isn't shared. Drift risk if thresholds change in one place but not another.
**Fix:** Extract named constants (`STALE_THRESHOLD_MS`, `VOLTAGE_LOW`, `VOLTAGE_CRITICAL`, `COOLANT_WARN`, `COOLANT_CRITICAL`) in a shared location.

---

_Reviewed: 2026-05-14T00:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
