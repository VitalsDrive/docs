---
phase: 03-live-fleet-dashboard
plan: "02"
subsystem: dashboard
tags: [fleet-map, stale-state, loading-states, connection-pill, toast, angular-signals]
dependency_graph:
  requires: [03-01]
  provides: [stale-marker, fitBounds-guard, connection-pill, pushAlert, dashboard-empty-state]
  affects: [packages/dashboard]
tech_stack:
  added: []
  patterns: [4-state getVehicleState, initialFitDone guard, bindTooltip, effect() connection toast, pushAlert sentinel]
key_files:
  created: []
  modified:
    - packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts
    - packages/dashboard/src/app/features/fleet-map/fleet-map.component.html
    - packages/dashboard/src/app/features/dashboard/dashboard.component.ts
    - packages/dashboard/src/app/features/dashboard/dashboard.component.html
    - packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-grid.component.ts
    - packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-grid.component.html
    - packages/dashboard/src/app/layout/shell/shell.component.ts
    - packages/dashboard/src/app/layout/shell/shell.component.html
    - packages/dashboard/src/app/core/services/alert.service.ts
decisions:
  - "Stale check fires before inactive/alert checks in getVehicleState() — stale always wins"
  - "initialFitDone class field guards fitBounds() to first load only"
  - "applyStaleTooltip() extracted as private method to avoid duplication between create/update paths"
  - "vd-connection-pill placed in shell template (not inside HeaderComponent) — HeaderComponent not in plan scope"
  - "Reconnect toast auto-dismisses at 8000ms (ALERT_AUTO_DISMISS.info) not 3000ms (UI-SPEC) — ALERT_AUTO_DISMISS not modified"
  - "ShellComponent org/fleet load chain kept in constructor (not ngOnInit) — resource() still needs org+fleet signals seeded"
metrics:
  duration: "~30 minutes"
  completed: "2026-05-14"
  tasks_completed: 3
  files_changed: 9
---

# Phase 03 Plan 02: UI Integration — Stale State, Loading/Empty States, Connection Pill Summary

Wire all UI integration points: map stale state, dashboard loading/empty states, shell connection pill, and connection toast.

## What Was Built

### Task 1 — FleetMapComponent (commit 532248b)

**getVehicleState()** — 4-state with stale priority:
- Return type: `'stale' | 'offline' | 'running' | 'alert'`
- Stale threshold: `15 * 60 * 1000` ms from `Date.now()`
- Priority order: stale → offline (inactive) → alert → running → offline

**getMarkerIcon()** — stale branch + color fixes:
- Added `stale: '#5a4530'` to colors map
- `opacity: 0.5` applied via inline style on stale markers
- Alert color updated from `#ef4444` to `#eab308` (UI-SPEC wins over PATTERNS.md)
- Running color updated from `#22c55e` to `#84cc16` (UI-SPEC token `--vd-healthy`)
- Pulse animation only on `running` and `alert` states (not stale)

**initialFitDone guard** — `private initialFitDone = false` class field wraps the entire fitBounds block; sets to `true` after first fire.

**applyStaleTooltip()** — private method called after icon set for both new and existing markers:
- Stale: `marker.bindTooltip('Last seen N min ago', { permanent: false })`
- Non-stale: `marker.unbindTooltip()`

**Template** — loading overlay (LoadingSpinnerComponent) + empty state overlay ("No vehicles with GPS data") both positioned absolute over always-rendered Leaflet map div. Legend updated to match new colors.

**Tests:** 3/3 fleet-map.component spec tests pass (GREEN).

---

### Task 2 — DashboardComponent + VehicleGridComponent (commit 1ae47f5)

**DashboardComponent additions:**
- `readonly isLoading = this.vehicleService.vehicleResource.isLoading`
- `readonly vehiclesWithHealth = this.vehicleService.vehiclesWithHealth`
- `readonly hasError = computed(() => this.vehicleService.vehicleResource.status() === 'error')`
- `LoadingSpinnerComponent` + `RouterLink` added to imports[]

**Dashboard template** — three-branch `@if/@else if/@else`:
1. `@if (isLoading())` → `<app-loading-spinner message="Loading fleet..." />`
2. `@else if (vehiclesWithHealth().length === 0)` → empty state with "No vehicles in your fleet" heading, body text, `+ Add Vehicle` button `routerLink="/fleet-management"`
3. `@else` → `<app-vehicle-grid />`

**VehicleGridComponent** — swapped signal sources:
- `isLoading` → `vehicleService.vehicleResource.isLoading`
- `hasError` → `computed(() => vehicleService.vehicleResource.status() === 'error')`
- `errorMessage` → `vehicleService.vehicleResource.error`
- Removed direct `vehicleService.isLoading` and `vehicleService.error` references

**Vehicle card click** — already wired in grid via `(clicked)="onVehicleSelected($event)"` → `router.navigate(['/vehicle', vehicleId])`. No change needed.

**Tests:** 4/4 dashboard + vehicle-card spec stubs pass (GREEN).

---

### Task 3 — AlertService + ShellComponent (commit cadfeea)

**AlertService.pushAlert()** added after `clearAllAlerts()`:
- Public method: `pushAlert(message: string, severity: AlertSeverity, type: AlertType = 'connection'): void`
- Uses `vehicleId: 'fleet'` as non-vehicle sentinel
- Increments module-level `alertIdCounter` (bypasses private `generateId()`)
- Caps alert array at 200 entries (same cap as `createAlert`)

**ShellComponent changes:**
- `implements OnInit` removed; class is now plain `ShellComponent`
- `TelemetryService` and `AlertService` injected
- `readonly connectionStatus = this.telemetryService.connectionStatus` exposed
- `private previousConnectionStatus: string | null = null` for transition tracking
- `effect()` in constructor watches `connectionStatus()`:
  - `connected → disconnected`: `pushAlert('Connection lost — reconnecting…', 'warning')` (12000ms auto-dismiss)
  - `disconnected → connected`: `pushAlert('Live data restored.', 'info')` (8000ms auto-dismiss)
- `VdConnectionPillComponent` added to imports[] and template

**Shell template:** `<vd-connection-pill [state]="connectionStatus()" />` added in `mat-sidenav-content` area above alert-banner.

**Build:** `npm run build` — 0 TypeScript errors.

---

## Test Results

| Suite | Tests | Status |
|-------|-------|--------|
| fleet-map.component.spec.ts | 3 | PASS |
| dashboard.component.spec.ts | 2 | PASS |
| vehicle-card.component.spec.ts | 2 | PASS |
| vehicle.service.spec.ts | 8 (7 pass, 1 intentional RED) | Expected |
| **Total** | **25** | **24 pass, 1 expected RED** |

The 1 RED test (`vehicleResource: loader queries vehicles WHERE fleet_id IN org fleet IDs AND status = active`) is the same intentional RED from Wave 0 — mock resource doesn't auto-run the loader in jest context. Documented in 03-01-SUMMARY.md.

## Build Status

`cd packages/dashboard && npm run build` — PASSED (0 TypeScript errors)

## Deviations from Plan

### Auto-fixed Issues

None — plan executed as written.

### Design Deviations (documented)

**1. Reconnect toast auto-dismiss: 8000ms vs 3000ms**
- **Specified:** UI-SPEC calls for 3000ms reconnect toast dismiss
- **Implemented:** `ALERT_AUTO_DISMISS.info = 8000ms` — no custom dismiss timeout added
- **Reason:** `ALERT_AUTO_DISMISS` is a const record shared across the system; modifying it for one use case was out of plan scope
- **Impact:** "Live data restored." toast stays 8s instead of 3s — functionally acceptable

**2. VdConnectionPillComponent in shell template (not inside HeaderComponent)**
- **Specified:** UI-SPEC shows conn-pill in shell header
- **Implemented:** Added to shell.component.html as absolute-positioned overlay near header
- **Reason:** HeaderComponent is out of plan scope; shell template wraps the header and is safe to modify
- **Impact:** Pill is visually in header area — functionally correct

**3. org/fleet load chain kept in ShellComponent constructor**
- **Plan said:** "Remove ngOnInit manual load chain" — `loadVehicles()` and `loadInitialTelemetry()` already removed in 03-01
- **Implemented:** `loadOrganizations() → loadFleets()` kept (now in constructor instead of ngOnInit) — these seed signals that `vehicleResource.params()` reads
- **Reason:** resource() still needs org+fleet signals populated; removing these would break vehicle loading

## Known Stubs

None — all data paths are wired to real resource signals.

## Threat Flags

None — no new network endpoints, auth paths, or trust boundaries introduced. See threat model in plan for accepted risks T-03-05, T-03-06, T-03-07.

## Self-Check

- [x] fleet-map.component.ts contains `private initialFitDone = false`
- [x] fleet-map.component.ts contains `this.initialFitDone = true`
- [x] fleet-map.component.ts contains `15 * 60 * 1000`
- [x] fleet-map.component.ts contains `bindTooltip`
- [x] fleet-map.component.html contains `No vehicles with GPS data`
- [x] fleet-map.component.html contains `app-loading-spinner`
- [x] dashboard.component.ts contains `vehicleResource.isLoading`
- [x] dashboard.component.html contains `@if (isLoading())`
- [x] dashboard.component.html contains `No vehicles in your fleet`
- [x] dashboard.component.html contains `routerLink="/fleet-management"`
- [x] dashboard.component.html contains `+ Add Vehicle`
- [x] vehicle-grid.component.ts does NOT contain `vehicleService.isLoading` or `vehicleService.error` directly
- [x] alert.service.ts contains `pushAlert(`
- [x] alert.service.ts pushAlert() uses `vehicleId: 'fleet'`
- [x] shell.component.ts does NOT contain `loadVehicles` or `loadInitialTelemetry`
- [x] shell.component.ts contains `telemetryService = inject(TelemetryService)`
- [x] shell.component.ts contains `connectionStatus = this.telemetryService.connectionStatus`
- [x] shell.component.ts contains `effect(() =>`
- [x] shell.component.ts contains `previousConnectionStatus`
- [x] shell.component.ts contains `alertService.pushAlert(`
- [x] shell.component.html contains `vd-connection-pill`
- [x] shell.component.html contains `[state]="connectionStatus()"`
- [x] Build passes (0 errors)
- [x] Commits: 532248b (task 1), 1ae47f5 (task 2), cadfeea (task 3)

## Self-Check: PASSED
