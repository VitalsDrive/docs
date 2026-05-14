# Phase 3: Live Fleet Dashboard - Context

**Gathered:** 2026-05-14
**Status:** Ready for planning

<domain>
## Phase Boundary

Wire the existing Leaflet map, vehicle health cards, and Supabase Realtime infrastructure into a fully functional real-time fleet dashboard. Both `/dashboard` (vehicle grid) and `/map` (Leaflet map) routes must display live data with proper security, staleness handling, and empty states. No new UI components need to be built from scratch — this phase is integration and correctness.

</domain>

<decisions>
## Implementation Decisions

### Dashboard Layout
- **D-01:** Keep `/dashboard` and `/map` as separate routes. No unification. `/dashboard` = vehicle health card grid. `/map` = full Leaflet map with marker detail panel.
- **D-02:** Default root redirect stays as `/dashboard`. Sidebar navigation includes both [Dashboard] and [Map] items.
- **D-03:** Vehicle card click on `/dashboard` navigates to `/vehicle/:id` (full detail page).

### Data Bootstrap & Reactivity
- **D-04:** Adopt Angular 21 `resource()` API throughout — mirrors OrganizationService pattern already in the codebase.
- **D-05:** `VehicleService` refactors vehicle loading to `resource()` with `params: () => selectedOrg.id`. Auto-reloads on org change. Shell does NOT call any load methods manually.
- **D-06:** Initial telemetry snapshot uses `resource()` with `params: () => vehicleIds` (derived from vehicle resource). Triggers automatically when vehicles load.
- **D-07:** `subscribeToFleet()` starts as a side effect of the vehicle resource loader completing (inside VehicleService). VehicleService owns the subscription lifecycle — `ngOnDestroy` already handles cleanup.
- **D-08:** `vehiclesWithHealth` stays as `computed()` merging vehicle resource value + realtime telemetry signal. No structural change to the computed pattern.
- **D-09:** VehicleService's `telemetryMap` signal is updated by the resource snapshot on initial load, then patched by Realtime records as they arrive. `processBatch()` remains the merge function.

### Telemetry Initial Load
- **D-10:** Replace `loadInitialTelemetry()` 500-row `.limit()` query with a Supabase RPC function `get_latest_telemetry(vehicle_ids uuid[])` using `SELECT DISTINCT ON (vehicle_id) * FROM telemetry_logs WHERE vehicle_id = ANY($1) ORDER BY vehicle_id, timestamp DESC`. Fetches exactly 1 record per vehicle — scales to any fleet size.
- **D-11:** The RPC function must have `SECURITY DEFINER` removed (use caller's RLS context) or explicitly check fleet membership inside the function body.

### Security — Telemetry RLS
- **D-12:** Add Supabase RLS migration for `telemetry_logs`. Policy: `SELECT` allowed only when the vehicle belongs to a fleet the current user is a member of.

  ```sql
  CREATE POLICY telemetry_fleet_member ON telemetry_logs
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM vehicles v
      JOIN fleet_members fm ON fm.fleet_id = v.fleet_id
      WHERE v.id = telemetry_logs.vehicle_id
        AND fm.user_id = (auth.jwt()->>'sub')
    )
  );
  ```

- **D-13:** Supabase Realtime respects RLS automatically for `postgres_changes` subscriptions. Once D-12 is in place, `subscribeToFleet()` will only deliver records the user can SELECT — no client-side filtering needed.
- **D-14:** Enable RLS on `telemetry_logs` if not already enabled (`ALTER TABLE telemetry_logs ENABLE ROW LEVEL SECURITY`). Verify parser's insert path uses service role key (bypasses RLS for writes).

### GPS Staleness
- **D-15:** Staleness threshold = **15 minutes**. Compare `latestTelemetry.timestamp` against `Date.now()`.
- **D-16:** Stale vehicles still appear on the map at their last known position using a grey dimmed `DivIcon` marker.
- **D-17:** Stale marker tooltip/title shows "Last seen X min ago" (not the vehicle name alone).
- **D-18:** `getVehicleState()` in `FleetMapComponent` gains a fourth state: `'stale'`. State priority: `stale` > `offline` > `alert` > `running`. A stale vehicle cannot be `running` or `alert` regardless of other signals.

### Connection State UI
- **D-19:** `ConnectionPillComponent` (already in `shared/ui/connection-pill`) is wired to `TelemetryService.connectionStatus` signal and rendered in the shell header — always visible.
- **D-20:** On status transitions (`connected → disconnected`, `disconnected → connected`), emit a toast/snackbar in addition to the pill state change. Use `ToastComponent` (already in `shared/components/toast`).

### Map Marker Interaction
- **D-21:** Map marker click → `VehicleDetailPanelComponent` slide-in (already built and wired to `selectedVehicleId` signal). No behavior change needed here.
- **D-22:** The existing panel already has a "View full details" link pattern. Ensure it routes to `/vehicle/:id`.

### Empty States
- **D-23:** `/dashboard` with no vehicles: illustration + "No vehicles in your fleet" + `[+ Add Vehicle]` button routing to `/fleet-management`.
- **D-24:** `/map` with no vehicles having GPS data: map tiles visible at default center (Tel Aviv: `[32.0853, 34.7818]`), centered overlay: "No vehicles with GPS data". Default center already set in `FleetMapComponent.initMap()`.
- **D-25:** Loading state (resource pending): show `LoadingSpinnerComponent` (already in `shared/components`).

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase Scope
- `.planning/ROADMAP.md` §Phase 3 — goal, requirements [FLEET-01, FLEET-02, FLEET-03], success criteria
- `.planning/PROJECT.md` — core constraints (Supabase Realtime not polling, Angular 21 signals)

### Existing Map Implementation
- `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts` — existing Leaflet map: `DivIcon` markers, `getVehicleState()`, `updateMarkers()`, `selectedVehicleId` signal, auto-fit bounds. This file is modified in Phase 3 (GPS staleness).
- `packages/dashboard/src/app/features/fleet-map/vehicle-detail-panel/vehicle-detail-panel.component.ts` — slide-in panel, already wired to `selectedVehicleId`.

### Data Layer
- `packages/dashboard/src/app/core/services/vehicle.service.ts` — `vehiclesWithHealth` computed, `calculateHealthScore()` (fully implemented, no changes), `processBatch()`, `loadVehicles()` (to be refactored to `resource()`).
- `packages/dashboard/src/app/core/services/telemetry.service.ts` — `subscribeToFleet()`, `subscribeToVehicle()`, `connectionStatus` signal, `telemetryBatch$` RxJS stream.
- `packages/dashboard/src/app/core/services/supabase.service.ts` — Supabase client injection.
- `packages/dashboard/src/app/core/services/organization.service.ts` — pattern reference: how `resource()` is used for org loading. **Read this to understand the resource() pattern before implementing D-05/D-06.**

### Dashboard & Routes
- `packages/dashboard/src/app/features/dashboard/dashboard.component.ts` — needs bootstrap wiring and empty state.
- `packages/dashboard/src/app/app.routes.ts` — `/dashboard` and `/map` route definitions.
- `packages/dashboard/src/app/layout/shell/shell.component.ts` — auth-protected layout wrapper; review for connection pill placement.

### Shared UI (Reuse)
- `packages/dashboard/src/app/shared/ui/connection-pill/connection-pill.component.ts` — wire to `connectionStatus`.
- `packages/dashboard/src/app/shared/components/loading-spinner/loading-spinner.component.ts` — use for resource pending state.
- `packages/dashboard/src/app/shared/components/toast/toast.component.ts` — use for connection status change toasts.

### Vehicle Grid
- `packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-grid.component.ts` — vehicle card grid, needs empty state.
- `packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-card/vehicle-card.component.ts` — already wired with health gauge, battery, DTC; card click emits `clicked` output.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `FleetMapComponent` — Leaflet map fully built. DivIcon markers with `offline/running/alert` states. `effect()` watches `vehicles()` signal and throttle-updates markers (1s debounce). Needs `stale` state added to `getVehicleState()` and `updateMarkers()` staleness filter update.
- `VehicleService.calculateHealthScore()` — fully implemented. Voltage + coolant + DTC thresholds. **Do not modify.**
- `TelemetryService.subscribeToFleet()` — single fleet-wide channel, 500ms buffer batching via RxJS `bufferTime`. Guard prevents duplicate channel creation. `ngOnDestroy` already removes channels.
- `ConnectionPillComponent` — exists in `shared/ui`, accepts status input. Wire to `TelemetryService.connectionStatus`.
- `VehicleDetailPanelComponent` — fully built and wired to `FleetMapComponent.selectedVehicleId`.
- `LoadingSpinnerComponent` — exists in `shared/components`.
- `ToastComponent` — exists in `shared/components`.

### Established Patterns
- `resource()` API: uses `params` property (not `request`). Loader receives `{ params, abortSignal }`. See `OrganizationService` for canonical usage in this codebase.
- Angular signals: `signal()`, `computed()`, `effect()` — used throughout. No NgRx.
- `auth.jwt()->>'sub'` (not `auth.uid()`) for Auth0 user identity in RLS policies — established in Phase 2 migration.
- Soft-delete: vehicles use `status: 'inactive'`, never hard-deleted. Filter `status = 'active'` in vehicle queries.
- Supabase client injected via `SupabaseService.client`.

### Integration Points
- **VehicleService ↔ TelemetryService:** `loadVehicles()` calls `subscribeToFleet()` and pipes `telemetryBatch$` into `processBatch()`. This wiring must survive the refactor to `resource()`.
- **VehicleService ↔ OrganizationService:** Org selection drives vehicle loading. `resource()` params must derive from `OrganizationService.selectedOrganization()`.
- **VehicleService ↔ FleetService:** Fleet IDs are used to scope vehicle queries when no specific fleetId provided. This scoping logic must be preserved in the resource loader.
- **ShellComponent ↔ ConnectionPill:** Shell header renders the connection pill. Confirm Shell imports `ConnectionPillComponent` in Phase 3.

</code_context>

<specifics>
## Specific Ideas

- User wants Angular 21's latest reactivity (`resource()`, `linkedSignal()`, `computed()`) — mirroring the org data loading pattern.
- "Hybrid" telemetry approach: `resource()` for snapshot, Realtime for live push. Not a pure `resource()` replacement.
- Health score algorithm is already correct — no changes to thresholds or formula.
- Simulator (Ghost Fleet) remains the primary testing tool for Realtime data flow. Ensure simulator writes via parser → Supabase → Realtime channel path is verified during planning.
- Parser uses service role key for inserts — must bypass RLS. Confirm `SUPABASE_SERVICE_ROLE_KEY` is set in parser environment before applying D-12.

</specifics>

<deferred>
## Deferred Ideas

- **Telemetry history / sparklines** — vehicle detail page (Phase 4+). The 100-record `telemetryMap` history buffer is retained in VehicleService for future use but not displayed in Phase 3.
- **Per-vehicle Realtime channels** — `subscribeToVehicle()` already implemented in TelemetryService but not used in Phase 3. Reserved for vehicle detail page (future phase).
- **Map clustering** — Leaflet marker clustering for large fleets. Deferred until fleet size warrants it.
- **Geofencing** — explicitly out of scope (PROJECT.md).
- **Route history / breadcrumb trail** — show where vehicle has been. Deferred.

</deferred>

---

*Phase: 3-Live Fleet Dashboard*
*Context gathered: 2026-05-14*
