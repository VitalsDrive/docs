# Phase 3: Live Fleet Dashboard — Discussion Log

**Date:** 2026-05-14
**Participants:** User + Claude Code

---

## Codebase Scout Findings

Before presenting gray areas, the existing Phase 3 codebase was scouted. Key finding: substantial implementation already exists:

- `FleetMapComponent` — Leaflet map with DivIcon markers, auto-fit bounds, `getVehicleState()`, throttled updates
- `TelemetryService` — single fleet channel, 500ms RxJS batching, `connectionStatus` signal
- `VehicleService` — `vehiclesWithHealth` computed, `calculateHealthScore()` (fully implemented), `processBatch()`
- Routes: `/dashboard` and `/map` already registered as separate routes
- Shared UI: `ConnectionPillComponent`, `LoadingSpinnerComponent`, `ToastComponent` all available

Phase 3 is an **integration and wiring** phase — not a greenfield build.

---

## Gray Areas Discussed

### 1. Dashboard Layout
**Question:** Should `/dashboard` and `/map` stay as separate routes, or be merged into one unified view?

**Options presented:** Keep separate | Unified split view | Map-primary

**Decision:** Keep separate routes — `/dashboard` (health card grid) + `/map` (full Leaflet map). Vehicle card click → `/vehicle/:id`.

---

### 2. Data Bootstrap
**Question:** Where should `loadVehicles()` + `loadInitialTelemetry()` be triggered?

**User direction:** Hybrid approach — use Angular's `resource()` API throughout, mirroring org data loading. Vehicles are relatively static so load reactively via `resource()` when org changes. Telemetry is dynamic — `resource()` for initial snapshot + Realtime for live push. VehicleService owns the subscription lifecycle.

**Decision:** `resource()` in VehicleService with params derived from org selection. Shell does NOT call any load methods. `subscribeToFleet()` starts as a side effect of vehicle resource loader completing.

**Realtime lifecycle question:** VehicleService owns it (existing `ngOnDestroy` cleanup preserved).

---

### 3. Telemetry RLS
**Question:** The Realtime channel receives ALL telemetry_logs globally. How to restrict?

**Options presented:** RLS policy on `telemetry_logs` | Client-side filter only

**Decision:** RLS migration on `telemetry_logs` via `vehicles → fleet_members` JOIN with `auth.jwt()->>'sub'`. Supabase Realtime respects RLS automatically. Parser uses service role key (bypasses RLS for writes) — must verify before applying.

---

### 4. GPS Staleness
**Question:** How should the map handle vehicles with no recent GPS fix?

**Options presented:** 15-min threshold (grey marker) | 30-min (remove from map) | Defer to Phase 4

**Decision:** 15-minute threshold. Stale vehicles show at last known position with grey dimmed DivIcon. Tooltip shows "Last seen X min ago". New `'stale'` state added to `getVehicleState()`.

---

### 5. Connection State UI
**Question:** Where to show Realtime connection status?

**Decision (hybrid):** Persistent pill in shell header (`ConnectionPillComponent`) + toast/snackbar on status transitions.

---

### 6. Map Marker Click
**Question:** Marker click → slide-in panel or navigate to detail page?

**Decision:** Keep existing `VehicleDetailPanelComponent` slide-in behavior (already built and wired). No change needed.

---

### 7. Empty States
**Question:** What to show when fleet has no vehicles?

**Decision:**
- `/dashboard` no vehicles → illustration + "No vehicles yet" + `[+ Add Vehicle]` CTA → `/fleet-management`
- `/map` no GPS data → map tiles visible at default center (Tel Aviv), overlay: "No vehicles with GPS data"
- Loading state → `LoadingSpinnerComponent`

---

### 8. Initial Telemetry Load Strategy
**Question:** Current `loadInitialTelemetry()` fetches top 500 rows — restructure?

**Decision:** Replace with Supabase RPC `get_latest_telemetry(vehicle_ids uuid[])` using `DISTINCT ON (vehicle_id)` — exactly 1 record per vehicle, scales to any fleet size.

---

## Claude's Discretion

- `processBatch()` merge logic — no changes needed, user confirmed health score algorithm is correct
- Default map center (Tel Aviv `[32.0853, 34.7818]`) — already set, keep as-is
- `bufferTime(500)` batching interval — keep at 500ms, sufficient for fleet-scale updates

---

## Deferred Ideas

- Map clustering for large fleets — deferred (not needed for MVP fleet size)
- Per-vehicle Realtime channels in Phase 3 — deferred (use fleet-wide channel only; per-vehicle already implemented in TelemetryService for future use)
- Route/history breadcrumb trail on map — deferred
