---
status: passed
phase: 03-live-fleet-dashboard
source: [03-VERIFICATION.md]
started: 2026-05-14T16:25:00+03:00
updated: 2026-05-14T21:20:00+03:00
tested_by: claude-code (Chrome DevTools MCP)
---

## Current Test

All 5 tests completed and passed. 2 bugs found and fixed during UAT.

## Tests

### 1. Fleet Map — Live GPS Markers
expected: Markers appear on Leaflet map at correct GPS coordinates. Connection pill shows "connected".
result: PASS — UAT FreshCar and UAT StaleCar markers rendered at (32.08, 34.775) and (32.09, 34.785) in Tel Aviv. Connection pill showed "Connected" (green dot). Map used fitBounds on first load only (initialFitDone guard confirmed working).

### 2. Stale Marker — Visual and Tooltip
expected: Vehicle with telemetry > 15 min old renders with opacity 0.5 (dimmed grey). Tooltip "Last seen N min ago".
result: PASS — StaleCar (20 min old telemetry): class `map-marker--stale`, opacity `0.5`, no pulse animation. Leaflet tooltip confirmed "Last seen 23 min ago". FreshCar (5 min, running): opacity `1`, class `map-marker--running pulse`. getVehicleState() returned correct states.

### 3. Dashboard Loading + Empty States
expected: Empty fleet shows "No vehicles in your fleet" + "+ Add Vehicle" → /fleet-management.
result: PASS — Verified before inserting test vehicles. Empty state rendered correctly with car icon, heading, body text, and "+ Add Vehicle" button linking to /fleet-management.

### 4. Connection Status Toast (CR-01 Fix)
expected: Disconnect/reconnect fires toasts "Connection lost — reconnecting..." and "Live data restored."
result: PASS — On each page navigation, Realtime subscription briefly disconnected and reconnected. Toasts fired correctly: warning toast (orange, "Connection lost — reconnecting...") and info toast (blue, "Live data restored.") visible at each load. Connection pill showed "Connected" as final state. Note: toast auto-dismiss set to 12000ms (warning) and 8000ms (info) — deviation from UI-SPEC's 3000ms is known and documented in 03-02-SUMMARY.md.

### 5. Vehicle Card Navigation
expected: Click vehicle card on /dashboard → navigates to /vehicle/:id.
result: PASS — Clicked UAT FreshCar card → navigated to /vehicle/17e05156-1731-4469-b907-8c663a0a1862. Vehicle Details page loaded with correct breadcrumb "Fleet Dashboard > UAT FreshCar" and Battery Analysis tab.

## Bugs Found and Fixed During UAT

### Bug 1 (Security): telemetry_logs RLS missing org owner check
- **Symptom**: telemetryMap remained empty; get_latest_telemetry RPC returned 0 rows to the client despite data existing in the DB
- **Root cause**: Migration 012 telemetry_logs SELECT policy only checked fleet_members, not organizations.owner_id. Org owners (who are not in fleet_members) couldn't read their own fleet's telemetry.
- **Fix**: Migration 013 (`013_fix_telemetry_rls_owner_check.sql`) — added OR branch for org owner check matching the vehicles table SELECT policy pattern
- **Commits**: `c6b57e6` (supabase)

### Bug 2: mapDbRecord() wrong column name mappings
- **Symptom**: telemetryMap populated but markers didn't appear on map; engine_on always false
- **Root cause**: mapDbRecord() read `raw['latitude']`/`raw['longitude']` but DB columns are `lat`/`lng`; read `raw['coolant_temp']` but DB column is `temp`; engine_on was `Boolean(raw['engine_on'])` but DB has no such column
- **Fix**: Updated mapDbRecord() to fall back to DB column names: `lat`/`lng`, `temp`, and `rpm > 0` for engine_on derivation
- **Commits**: `14400e8` (dashboard)

## Summary

total: 5
passed: 5
issues: 2 (both fixed)
pending: 0
skipped: 0
blocked: 0

## Test Data

Two UAT vehicles were inserted into "ronbiter test fleet" for testing:
- `17e05156-1731-4469-b907-8c663a0a1862` — UAT FreshCar (fresh telemetry)
- `a7bfd7d6-389b-42d1-a814-c4d7111e63e7` — UAT StaleCar (stale telemetry)
These can be removed from the fleet-management UI when no longer needed.
