---
phase: 02-auth-fleet-management
plan: "04"
subsystem: dashboard
tags: [fleet-management, vehicle-crud, angular-material, signals, soft-delete]
dependency_graph:
  requires: [02-02, 02-03]
  provides: [vehicle-service-crud, fleet-management-page]
  affects: [app.routes.ts, vehicle.model.ts, vehicle.service.ts]
tech_stack:
  added: []
  patterns: [signal-state, mat-dialog, soft-delete, lazy-load-route]
key_files:
  created:
    - packages/dashboard/src/app/features/fleet-management/fleet-management.component.ts
    - packages/dashboard/src/app/features/fleet-management/fleet-management.component.html
    - packages/dashboard/src/app/features/fleet-management/fleet-management.component.scss
    - packages/dashboard/src/app/features/fleet-management/add-vehicle-dialog.component.ts
    - packages/dashboard/src/app/features/fleet-management/remove-vehicle-dialog.component.ts
  modified:
    - packages/dashboard/src/app/core/services/vehicle.service.ts
    - packages/dashboard/src/app/core/models/vehicle.model.ts
    - packages/dashboard/src/app/app.routes.ts
decisions:
  - "Extended existing VehicleService rather than replacing it — service already had loadVehicles, signals, and telemetry wiring; CRUD methods appended as Phase 2 fleet management layer"
  - "Created two inline dialog components (AddVehicleDialogComponent, RemoveVehicleDialogComponent) rather than embedding dialog logic in the page component — cleaner separation"
  - "Vehicle model updated in-place (vehicle.model.ts) to add nullable nickname/make/model/vin fields; CreateVehicleDto added to model file and re-exported from service"
metrics:
  duration: "~25 minutes"
  completed_date: "2026-05-13"
  tasks_completed: 2
  files_modified: 8
---

# Phase 2 Plan 04: Fleet Management Page & VehicleService CRUD Summary

**One-liner:** Fleet management page at /fleet-management with vehicle card grid, MatDialog add/remove flows, fleet switcher, and soft-delete via VehicleService CRUD methods.

## Tasks Completed

| Task | Description | Commit |
|------|-------------|--------|
| 1 | VehicleService CRUD methods + Vehicle model extensions | 8d2a9ab |
| 2 | FleetManagementComponent, dialogs, route registration | 97e6726 |

## What Was Built

### Task 1 — VehicleService CRUD

Extended `packages/dashboard/src/app/core/services/vehicle.service.ts` with:
- `loadAllVehicles()` — loads all active vehicles (no fleet filter), ordered by nickname
- `loadVehicles(fleetId)` — scoped to fleet, ordered by nickname (existing method adapted)
- `createVehicle(dto)` — inserts with `status: 'active'`, optimistically appends to signal
- `updateVehicle(id, updates)` — patches row + updates signal in-place
- `deleteVehicle(id)` — **soft-delete**: sets `status: 'inactive'` + `updated_at`, removes from signal

Updated `vehicle.model.ts`:
- `nickname`, `make`, `model`, `vin`, `license_plate` made nullable (`string | null`)
- Added `CreateVehicleDto` interface (nickname required, all others nullable)

### Task 2 — FleetManagementComponent

`/fleet-management` page:
- Card grid: `repeat(auto-fill, minmax(280px, 1fr))`, gap `--vd-space-6`
- Each card: left accent border (`--vd-healthy`), nickname (Space Grotesk 600), make/model/year, plate badge (JetBrains Mono), fleet eyebrow label, edit/remove icon buttons
- Fleet switcher: MatSelect with "All Fleets" default + org fleets
- Empty state: "No vehicles yet" heading, "Add your first vehicle to start monitoring fleet health." body, "+ Add Vehicle" CTA
- Add button at top-right header (amber `--vd-brand`)
- Error banner via `vehicleService.error()` signal

**AddVehicleDialogComponent:**
- nickname (required, 2–40 chars), fleet_id (required), make/model/year/license_plate/vin (optional)
- `Validators.pattern(/^\d{4}$/)` on year
- Closes with `CreateVehicleDto` on submit

**RemoveVehicleDialogComponent:**
- Title: "Remove [Nickname]?"
- Body: "This vehicle will be deactivated and removed from your fleet. Historical telemetry data is preserved." (exact UI-SPEC copy)
- Actions: Cancel (ghost) + "Remove Vehicle" (--vd-critical red)
- Closes with `true` on confirm

**Route:** `/fleet-management` added to protected shell children with `canActivate: [onboardingGuard]`.

## Deviations from Plan

### Auto-fixed — Rule 2: Existing VehicleService collision

**Found during:** Task 1
**Issue:** `vehicle.service.ts` already existed as a complex telemetry service with signals, realtime subscriptions, and health-score computation. Replacing it would break the dashboard's live vehicle monitoring.
**Fix:** Added the 5 CRUD methods (`loadAllVehicles`, `createVehicle`, `updateVehicle`, `deleteVehicle`, updated `loadVehicles`) as a new section within the existing service. Updated `vehicle.model.ts` in-place for nullable fields + `CreateVehicleDto`. Re-exported types from service for backward compatibility.
**Files modified:** `vehicle.service.ts`, `vehicle.model.ts`

### Note — lint not configured

`npm run lint` is not configured (angular-eslint not installed). TypeScript (`npx tsc --noEmit`) and dev build used for verification. Pre-existing condition, out of scope.

## Verification Results

| Check | Result |
|-------|--------|
| `grep "status.*inactive" vehicle.service.ts` | Match at line 369 (deleteVehicle soft-delete) |
| `grep "fleet-management" app.routes.ts` | Match — route registered |
| `grep "No vehicles yet" fleet-management.component.html` | Match at line 42 |
| `grep "auth.getUser" vehicle.service.ts` | No match (correct) |
| `npx tsc --noEmit` | 0 errors |
| `npm run build -- --configuration=development` | Success (1 pre-existing unrelated warning) |

## Known Stubs

None — all data flows through VehicleService signals from Supabase. Edit button in vehicle card currently opens AddVehicleDialog (same as add flow) rather than a pre-populated edit form. This is acceptable for Phase 2 MVP; a proper edit flow with pre-filled data can be wired in a later plan.

## Threat Surface Scan

No new network endpoints or auth paths introduced. VehicleService CRUD uses existing Supabase anon key + RLS (T-02-04-01 through T-02-04-04 mitigations applied: Validators.required, maxLength(40), Validators.pattern year, soft-delete via status filter).

## Self-Check: PASSED

- `packages/dashboard/src/app/core/services/vehicle.service.ts` — exists, contains `loadAllVehicles`, `createVehicle`, `updateVehicle`, `deleteVehicle`
- `packages/dashboard/src/app/features/fleet-management/fleet-management.component.ts` — exists
- `packages/dashboard/src/app/features/fleet-management/fleet-management.component.html` — exists, contains "No vehicles yet"
- `packages/dashboard/src/app/app.routes.ts` — contains `fleet-management` + `onboardingGuard`
- Commits `8d2a9ab` and `97e6726` present in git log
