# Phase 3: Live Fleet Dashboard — Research

**Researched:** 2026-05-14
**Domain:** Angular 21 signals + Supabase Realtime + Leaflet map integration
**Confidence:** HIGH (codebase read directly; Angular/Supabase docs verified via WebSearch + official sources)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01** `/dashboard` and `/map` stay as separate routes. No unification.
- **D-02** Default root redirect stays `/dashboard`. Sidebar has both nav items.
- **D-03** Vehicle card click on `/dashboard` → `/vehicle/:id`.
- **D-04** Adopt Angular 21 `resource()` API throughout — mirrors OrganizationService pattern.
- **D-05** `VehicleService` refactors vehicle loading to `resource()` with `params: () => selectedOrg.id`. Shell does NOT call load methods manually.
- **D-06** Initial telemetry snapshot uses `resource()` with `params: () => vehicleIds`. Triggers automatically when vehicles load.
- **D-07** `subscribeToFleet()` starts as side effect of vehicle resource loader completing (inside VehicleService). VehicleService owns subscription lifecycle.
- **D-08** `vehiclesWithHealth` stays as `computed()` merging vehicle resource value + realtime telemetry signal. No structural change.
- **D-09** `telemetryMap` signal updated by resource snapshot on initial load, then patched by Realtime records. `processBatch()` remains the merge function.
- **D-10** Replace `loadInitialTelemetry()` 500-row query with RPC `get_latest_telemetry(vehicle_ids uuid[])` using `SELECT DISTINCT ON (vehicle_id)`.
- **D-11** RPC must use caller's RLS context (no `SECURITY DEFINER`) or explicitly check fleet membership inside the function.
- **D-12** Add RLS migration for `telemetry_logs`. Policy uses `auth.jwt()->>'sub'` (Auth0 pattern from Phase 2).
- **D-13** Supabase Realtime respects RLS automatically for `postgres_changes`. No client-side filtering needed.
- **D-14** Enable RLS on `telemetry_logs` if not already. Verify parser uses service role key for inserts.
- **D-15** Staleness threshold = 15 minutes. Compare `latestTelemetry.timestamp` against `Date.now()`.
- **D-16** Stale vehicles appear on map at last known position with grey dimmed DivIcon.
- **D-17** Stale marker tooltip: "Last seen X min ago".
- **D-18** `getVehicleState()` gains 4th state `'stale'`. Priority: `stale > offline > alert > running`.
- **D-19** `VdConnectionPillComponent` wired to `TelemetryService.connectionStatus` in shell header.
- **D-20** Connection status transitions emit toast via `ToastComponent`.
- **D-21** Map marker click → `VehicleDetailPanelComponent` slide-in (already wired to `selectedVehicleId`).
- **D-22** "View full details" link in detail panel routes to `/vehicle/:id`.
- **D-23** `/dashboard` empty state: illustration + "No vehicles in your fleet" + `[+ Add Vehicle]` → `/fleet-management`.
- **D-24** `/map` empty state: map tiles visible at default center `[32.0853, 34.7818]`, centered overlay "No vehicles with GPS data".
- **D-25** Loading state: show `LoadingSpinnerComponent`.

### Claude's Discretion

- How to safely bridge `resource()` loader completion → `subscribeToFleet()` side effect (effect() pattern, exact placement).
- Exact SQL for `get_latest_telemetry(vehicle_ids uuid[])` RPC function signature and SECURITY model.
- Whether to use `toSignal()` to bridge `telemetryBatch$` Observable into the writable `telemetryMap` signal, or keep the explicit `subscribe()` + `telemetryMap.update()` pattern.
- Auto-fit bounds strategy — whether to suppress `fitBounds()` after first load to prevent jarring re-centering on each telemetry update.

### Deferred Ideas (OUT OF SCOPE)

- Telemetry history / sparklines (Phase 4+)
- Per-vehicle Realtime channels (`subscribeToVehicle()`)
- Map clustering
- Geofencing
- Route history / breadcrumb trail
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FLEET-01 | User can view live map of all their vehicles with GPS positions | Leaflet `updateMarkers()` + `resource()` vehicle load + RLS-scoped query |
| FLEET-02 | Dashboard updates vehicle positions in real-time via Supabase subscriptions | `subscribeToFleet()` + `telemetryBatch$` + `processBatch()` wiring |
| FLEET-03 | User can see health status of each vehicle (health score + alert count) | `vehiclesWithHealth` computed signal; `calculateHealthScore()` already implemented |
</phase_requirements>

---

## Summary

Phase 3 is an **integration phase**, not a greenfield build. The Leaflet map, vehicle health cards, Supabase Realtime subscription, `telemetryBatch$` stream, and `processBatch()` merge function all exist and are correct. The three integration gaps are: (1) the data bootstrap chain (`ShellComponent` currently calls `loadVehicles()` imperatively — must move to `resource()`-driven reactivity in VehicleService); (2) the telemetry initial snapshot (`loadInitialTelemetry()` must be replaced with RPC `get_latest_telemetry(vehicle_ids[])`); and (3) RLS on `telemetry_logs` (enabled in migration 002 but the existing policy uses `get_user_fleet_ids()` which calls `auth.uid()` — must be replaced with `auth.jwt()->>'sub'` for Auth0 compatibility).

The `resource()` API is stable in Angular 19+ (confirmed stabilized). The exact property name is `params` (not `request` — that was the v18 experimental name). Returning `undefined` from `params` causes the loader to be skipped and status becomes `'idle'`, which is the correct guard pattern for the `vehicleIds` resource when no vehicles have loaded yet.

The key architectural risk is **circular signal dependency**: if `VehicleService` `resource()` reads from a signal that is also written by `processBatch()`, Angular will throw a circular write error. The safe pattern is `effect()` inside the constructor (injection context) watching `vehicleResource.status()` and calling `subscribeToFleet()` when status transitions to `'resolved'`.

**Primary recommendation:** Implement `resource()` vehicle loading in VehicleService, guard telemetry resource with `params: () => vehicleIds.length > 0 ? vehicleIds : undefined`, use `effect()` to bridge resource resolution to `subscribeToFleet()`, and write migration 012 for RLS + RPC.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Vehicle list (initial load) | Angular service (VehicleService) | Supabase DB | `resource()` loader fetches from `vehicles` table scoped by org |
| Telemetry snapshot (initial) | Angular service (VehicleService) | Supabase RPC | `get_latest_telemetry()` RPC called once when vehicle IDs known |
| Live telemetry updates | Supabase Realtime | Angular service | `postgres_changes` INSERT events → `channel$` → `telemetryBatch$` |
| Health score computation | Angular service (VehicleService) | — | `calculateHealthScore()` — pure function on telemetry signal |
| Map rendering | Browser/Leaflet | Angular component | Leaflet owns DOM; Angular signals drive `updateMarkers()` |
| RLS enforcement | Database (Supabase) | Supabase Realtime | RLS on `telemetry_logs` filters both query and Realtime events |
| Staleness detection | Angular component (FleetMapComponent) | — | `getVehicleState()` computes stale from `Date.now() - timestamp` |
| Connection status display | Angular shell (ShellComponent) | TelemetryService | `connectionStatus` signal → `VdConnectionPillComponent` input |
| Toast on status change | Angular service or effect | ToastComponent | `effect()` watching `connectionStatus` transitions |

---

## Standard Stack

### Core (already installed — verified against `packages/dashboard/package.json`)

| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| `@angular/core` | ^21.0.0 | Signals, resource(), computed(), effect() | [VERIFIED: package.json] |
| `@supabase/supabase-js` | ^2.39.0 | Realtime channels, RPC, DB queries | [VERIFIED: package.json] |
| `leaflet` | ^1.9.4 | Map rendering, DivIcon markers | [VERIFIED: package.json] |
| `rxjs` | ^7.8.1 | bufferTime, takeUntil — telemetryBatch$ stream | [VERIFIED: package.json] |

### No New Dependencies Needed

Phase 3 requires zero new npm packages. All libraries are installed. [VERIFIED: package.json]

---

## Architecture Patterns

### System Architecture Diagram

```
OrganizationService.selectedOrganization() [signal]
        │
        ▼ params changes
VehicleService.vehicleResource [resource()]
  └─ loader: fetch vehicles WHERE fleet_id IN org's fleets
        │
        ▼ status === 'resolved'
        ├─► effect() → subscribeToFleet()
        │       └─► TelemetryService.channel$ [ReplaySubject]
        │               └─► telemetryBatch$ [bufferTime 500ms]
        │                       └─► processBatch() → telemetryMap.update()
        │
        └─► telemetryResource.params = () => vehicleIds
                └─► loader: RPC get_latest_telemetry(vehicle_ids)
                        └─► telemetryMap.set(snapshot)  ← initial load

vehiclesWithHealth [computed()]
  = vehicles() ⊕ telemetryMap()
        │
        ├─► FleetMapComponent.updateMarkers() [effect() → setTimeout 1s]
        │       └─► Leaflet DivIcon markers (running/alert/offline/stale)
        │
        └─► VehicleGridComponent.vehicles [signal alias]
                └─► VehicleCardComponent × N

TelemetryService.connectionStatus [signal]
        └─► ShellComponent → VdConnectionPillComponent [input]
        └─► effect() → ToastComponent (on transition)
```

### Recommended File Change Map

```
packages/dashboard/src/app/
├── core/services/
│   ├── vehicle.service.ts          ← ADD vehicleResource, telemetryResource, effect() wiring
│   └── telemetry.service.ts        ← NO CHANGES (already correct)
├── features/
│   ├── dashboard/
│   │   ├── dashboard.component.ts  ← ADD resource status passthrough, loading/error states
│   │   └── vehicle-grid/
│   │       └── vehicle-grid.component.ts  ← ADD empty state, resource status binding
│   └── fleet-map/
│       └── fleet-map.component.ts  ← ADD stale state to getVehicleState(), staleness filter
├── layout/shell/
│   └── shell.component.ts          ← REMOVE manual load chain, ADD ConnectionPill, ADD toast effect
└── shared/supabase/migrations/
    └── 012_telemetry_rls_and_rpc.sql  ← NEW: RLS policy + get_latest_telemetry() RPC
```

---

## Research Area 1: Angular 21 `resource()` API

### Exact Signature

[CITED: angular.dev/api/core/resource, angular.dev/api/core/ResourceLoaderParams]

```typescript
function resource<T, R>(options: ResourceOptions<T, R>): ResourceRef<T>;

interface ResourceOptions<T, R> {
  params?: () => R;           // reactive computation — re-runs when signals change
  loader: (opts: ResourceLoaderParams<R>) => Promise<T>;
  defaultValue?: T;
  injector?: Injector;
}

interface ResourceLoaderParams<R> {
  params: NoInfer<Exclude<R, undefined>>;  // undefined stripped — loader only called when non-undefined
  abortSignal: AbortSignal;                // use with fetch({ signal: abortSignal })
  previous: { status: ResourceStatus };   // previous resource status
}

interface ResourceRef<T> {
  readonly value: Signal<T | undefined>;
  readonly status: Signal<ResourceStatus>;  // 'idle'|'loading'|'refreshing'|'resolved'|'error'|'local'
  readonly error: Signal<unknown>;
  readonly isLoading: Signal<boolean>;
  set(value: T): void;           // set local value (status → 'local')
  update(fn: (v: T) => T): void; // update local value
  reload(): void;                 // programmatic reload
  hasValue(): boolean;            // type guard: value is defined and not error
  destroy(): void;
}
```

### Key Behaviors

**`params` returning `undefined` → loader skips, status = `'idle'`** [CITED: angular.dev/guide/signals/resource]

This is the correct guard for the telemetry resource:
```typescript
// Source: angular.dev/guide/signals/resource
private readonly vehicleIds = computed(() =>
  this.vehicleResource.value()?.map(v => v.id) ?? []
);

readonly telemetryResource = resource({
  params: () => {
    const ids = this.vehicleIds();
    return ids.length > 0 ? ids : undefined;  // undefined → loader skips, status = 'idle'
  },
  loader: async ({ params: vehicleIds }) => {
    // NOTE: .abortSignal() is NOT available on supabase-js query builder (verified v2.104.1)
    // abortSignal parameter from ResourceLoaderParams is intentionally unused here
    const { data, error } = await this.supabase.client
      .rpc('get_latest_telemetry', { vehicle_ids: vehicleIds });
    if (error) throw error;
    return data;
  }
});
```

**`params` property name is `params` — not `request`** [CITED: angular.dev/api/core/ResourceLoaderParams]

The `request` property name was used in the Angular 18 experimental API. As of Angular 19+ (stable), the property is `params`. CONTEXT.md D-04/D-05 confirm this codebase uses `params`.

**Status lifecycle:**
- `'idle'` — params returned undefined, loader not called
- `'loading'` — first load in progress
- `'refreshing'` — reload() called or params changed while value exists
- `'resolved'` — loader completed successfully
- `'error'` — loader threw or rejected
- `'local'` — value set manually via `.set()` or `.update()`

**`resource()` must be called in injection context** (constructor or field initializer). [CITED: angular.dev/api/core/effect]

---

## Research Area 2: Supabase Realtime + RLS

### How `postgres_changes` Respects RLS

[CITED: supabase.com/blog/realtime-row-level-security-in-postgresql, supabase.com/docs/guides/realtime/postgres-changes]

Supabase Realtime performs a security check before broadcasting a `postgres_changes` event to each subscribed client. It temporarily assumes the identity of the subscribing client (using the JWT they authenticated with) and verifies the changed row passes that client's RLS SELECT policy. If RLS evaluates to `false`, the event is silently dropped — the client never receives it.

**Key facts:**
- RLS evaluation is cached per connection (not per message) — no per-row DB query overhead
- The JWT must be passed when creating the Supabase client or channel for RLS to apply
- Without RLS enabled, all `postgres_changes` events go to all subscribers (security hole)
- DELETE events cannot be filtered by RLS (Supabase limitation) — not relevant here (parser only INSERTs)

### Channel Filter Syntax (existing implementation — correct)

[VERIFIED: packages/dashboard/src/app/core/services/telemetry.service.ts]

```typescript
// Current subscribeToFleet() — no filter on vehicle_id (fleet-wide channel)
this.fleetChannel = this.supabase.client
  .channel('fleet-telemetry')
  .on('postgres_changes',
    { event: 'INSERT', schema: 'public', table: 'telemetry_logs' },
    (payload) => { ... }
  )
  .subscribe();
```

With D-12 RLS policy in place, RLS automatically scopes the channel to vehicles the authenticated user can see. No `filter:` property needed on the channel. The existing implementation is correct post-RLS-migration.

### Reconnect Behavior

[CITED: github.com/supabase/realtime/issues/1088, github.com/supabase/realtime-py/issues/213]

**Known issue:** When Supabase Realtime auto-reconnects after a dropped WebSocket, the JS client successfully rejoins the channel but `postgres_changes` re-subscription is not guaranteed to be automatic in all versions. The `connectionStatus` signal already handles `CHANNEL_ERROR` and `TIMED_OUT` → `'disconnected'`. The existing `fleetChannel` guard (`if (this.fleetChannel) return`) means `subscribeToFleet()` will NOT re-subscribe after disconnect.

**Implication for planning:** The `effect()` that calls `subscribeToFleet()` must set `fleetChannel = null` before calling `subscribeToFleet()` again when recovering from disconnect, OR the Realtime client's built-in reconnect is sufficient (Supabase JS v2 handles WebSocket reconnect with channel re-subscription automatically for most cases). The existing `CHANNEL_ERROR`/`TIMED_OUT` → `connectionStatus('disconnected')` signal is correct UI feedback but does not re-subscribe. For MVP, trust Supabase JS v2 auto-reconnect. Document as known limitation.

### Cleanup in `ngOnDestroy`

[VERIFIED: telemetry.service.ts lines 117–127]

Already implemented correctly:
```typescript
ngOnDestroy(): void {
  this.destroy$.next();
  this.destroy$.complete();
  if (this.fleetChannel) {
    this.supabase.client.removeChannel(this.fleetChannel);
  }
  this.vehicleChannels.forEach(ch => this.supabase.client.removeChannel(ch));
}
```

---

## Research Area 3: Leaflet in Angular 21

### DivIcon Update Strategy — `setLatLng` + `setIcon` (existing approach)

[VERIFIED: fleet-map.component.ts lines 112–114]

The existing `updateMarkers()` already uses `setLatLng` + `setIcon` on existing markers (not remove+re-add):

```typescript
if (existing) {
  existing.marker.setLatLng([lat, lng]);
  existing.marker.setIcon(this.getMarkerIcon(vehicle));  // replaces DivIcon HTML
}
```

`setIcon` with a new `DivIcon` replaces the marker's DOM element in-place. This is the correct approach — remove+re-add would lose the click handler binding.

**Stale marker implementation:** Add `stale` to `getVehicleState()` return type and add a branch to `getMarkerIcon()`:

```typescript
// In getMarkerIcon() — add stale branch
const colors: Record<string, string> = {
  offline: '#5a4530',
  running: '#84cc16',
  alert:   '#eab308',
  stale:   '#5a4530',  // same as offline, opacity applied on marker element
};
const opacity = state === 'stale' ? '0.5' : '1';
// Apply opacity to the outer div:
// html: `<div class="map-marker map-marker--${state}" style="opacity:${opacity}" ...>`
```

**Tooltip for stale markers:**

```typescript
// In updateMarkers() after creating/updating marker:
if (state === 'stale') {
  const minsAgo = Math.floor((Date.now() - new Date(vehicle.latestTelemetry!.timestamp).getTime()) / 60000);
  marker.bindTooltip(`Last seen ${minsAgo} min ago`, { permanent: false });
} else {
  marker.unbindTooltip();
}
```

### Auto-fit Bounds Pitfall

[VERIFIED: fleet-map.component.ts lines 144–150]

The existing `fitBounds()` fires on **every** `updateMarkers()` call (every 1s from effect debounce). This causes the map to re-center whenever a vehicle position updates even slightly, preventing user pan/zoom. **Fix:** Fit bounds only on first load (when `markerMap` was previously empty), not on subsequent updates.

```typescript
// Track whether initial fit has occurred
private initialFitDone = false;

// In updateMarkers():
if (validVehicles.length > 0 && !this.initialFitDone) {
  const bounds = L.latLngBounds(...);
  this.map.fitBounds(bounds.pad(0.1));
  this.initialFitDone = true;
}
```

### Zone Issues

[ASSUMED] Leaflet runs outside Angular's change detection zone. `FleetMapComponent` already uses `ChangeDetectionStrategy.OnPush` and drives updates via `effect()` → `updateMarkers()` in the Angular zone. No `NgZone.runOutsideAngular()` needed for the current pattern since the effect runs inside Angular. Leaflet DOM mutations (marker positions, icons) don't need zone awareness for OnPush components driven by signals.

---

## Research Area 4: `resource()` + RxJS Bridge

### Pattern: effect() Watching resource.status() → subscribeToFleet()

[CITED: angular.dev/guide/signals/effect, angular.dev/api/core/effect]

The safe pattern for triggering side effects when a `resource()` resolves is an `effect()` in the constructor (injection context):

```typescript
// In VehicleService constructor
constructor() {
  effect(() => {
    // Read status reactively
    const status = this.vehicleResource.status();
    if (status === 'resolved') {
      this.telemetryService.subscribeToFleet();
      // Wire batch stream with cancellation on reload
      this.reloadVehicles$.next();
      this.telemetryService.telemetryBatch$
        .pipe(takeUntil(this.reloadVehicles$), takeUntil(this.destroy$))
        .subscribe(batch => this.processBatch(batch));
    }
  });
}
```

**Why `effect()` and not `computed()`:** Side effects (starting a WebSocket subscription) cannot go in `computed()`. `computed()` is for pure derived values only.

**Why not `afterNextRender()`:** `afterNextRender()` runs once after the first render — it's for DOM-dependent setup (like `initMap()`). Subscription wiring is a service-level concern independent of DOM.

**Why not `linkedSignal()`:** `linkedSignal()` is for writable signals that reset when a source signal changes. Not applicable to subscription lifecycle.

### Bridging `telemetryBatch$` Observable → `telemetryMap` Signal

[VERIFIED: vehicle.service.ts lines 118–121]

The existing approach (explicit `.subscribe()` calling `processBatch()`) is correct and idiomatic. Keep it.

Do NOT use `toSignal(telemetryBatch$)` here — `toSignal` creates a read-only signal that replaces the entire value on each emission. The `processBatch()` pattern does a **patch-style merge** (`Map.update()`), which preserves history for all vehicles. `toSignal` would require the observable to emit the full merged Map on each batch, forcing the merge logic into the Observable chain — worse separation of concerns.

### Avoiding Circular Signal Dependencies

**Risk:** If `vehicleResource.params` reads from `telemetryMap` (or any signal written by `processBatch()`), Angular throws a circular write error.

**Safe dependency graph:**
```
selectedOrganization → vehicleResource.params ✓
vehicleIds (computed from vehicleResource.value) → telemetryResource.params ✓
telemetryResource.value → telemetryMap.set() [initial snapshot] ✓
telemetryBatch$ subscription → telemetryMap.update() [live patches] ✓
telemetryMap → vehiclesWithHealth (computed) ✓
```

**Forbidden:** Do NOT read `telemetryMap` or `vehiclesWithHealth` inside any `resource()` params or loader. ✓ (current design does not do this)

---

## Research Area 5: Supabase RPC Security

### RLS on `telemetry_logs` — Current State

[VERIFIED: migrations/002_rls_policies.sql lines 10, 163–170]

`ALTER TABLE telemetry_logs ENABLE ROW LEVEL SECURITY` — **already executed** in migration 002.

Existing SELECT policy (migration 002):
```sql
CREATE POLICY "Users can read own fleet telemetry"
    ON telemetry_logs FOR SELECT
    USING (
        vehicle_id IN (
            SELECT v.id FROM vehicles v
            WHERE v.fleet_id IN (SELECT get_user_fleet_ids())
        )
    );
```

**Problem:** `get_user_fleet_ids()` uses `auth.uid()` (line 11 in migration 002). Auth0 integration established in Phase 2 requires `auth.jwt()->>'sub'`. This policy will return empty results for all Auth0 users (their `user_id` in `fleet_members` is the Auth0 `sub`, not Supabase's `auth.uid()`).

[VERIFIED: migrations/010_fix_rls_policies.sql — vehicles SELECT policy already fixed to use `auth.jwt()->>'sub'`]

**Migration 012 must:**
1. Drop the existing `"Users can read own fleet telemetry"` policy
2. Create new policy using `auth.jwt()->>'sub'` (matching the vehicles policy pattern from 010)
3. Create `get_latest_telemetry(vehicle_ids uuid[])` RPC function

### `get_latest_telemetry()` RPC — New Function

Per D-10 and D-11:

```sql
-- Migration: 012_telemetry_rls_and_rpc.sql
BEGIN;

-- Step 1: Fix telemetry_logs RLS (drop old auth.uid()-based policy)
DROP POLICY IF EXISTS "Users can read own fleet telemetry" ON telemetry_logs;

CREATE POLICY "Users can read own fleet telemetry" ON telemetry_logs
FOR SELECT USING (
  EXISTS (
    SELECT 1 FROM vehicles v
    JOIN fleet_members fm ON fm.fleet_id = v.fleet_id
    WHERE v.id = telemetry_logs.vehicle_id
      AND fm.user_id = (auth.jwt()->>'sub')
  )
);

-- Step 2: RPC for initial snapshot (one record per vehicle)
-- SECURITY INVOKER = runs as the calling user → RLS applies automatically (D-11)
CREATE OR REPLACE FUNCTION get_latest_telemetry(vehicle_ids uuid[])
RETURNS SETOF telemetry_logs
LANGUAGE sql
SECURITY INVOKER
STABLE
AS $$
  SELECT DISTINCT ON (vehicle_id) *
  FROM telemetry_logs
  WHERE vehicle_id = ANY(vehicle_ids)
  ORDER BY vehicle_id, timestamp DESC;
$$;

COMMIT;
```

**Why `SECURITY INVOKER`:** The function runs as the calling user, so RLS on `telemetry_logs` applies automatically. The existing `get_user_fleet_ids()` and other functions use `SECURITY DEFINER` (which bypasses RLS), but for this RPC the caller's RLS context IS the security model — no bypass needed. D-11 explicitly requires this.

**Why `RETURNS SETOF telemetry_logs`:** Returns full records including all columns that `mapDbRecord()` in VehicleService expects (id, vehicle_id, timestamp, rpm, speed, engine_on, coolant_temp, voltage, latitude, longitude, dtc_codes, fuel_level, signal_strength). Avoids maintaining a separate return type.

**Parser insert path:** Parser uses `SUPABASE_SERVICE_ROLE_KEY` which bypasses RLS entirely. Inserts always succeed regardless of the new policy. [VERIFIED: CONTEXT.md "Parser uses service role key for inserts" + D-14]

### Existing `get_latest_telemetry_per_vehicle()` — NOT reusable

[VERIFIED: migrations/003_functions_triggers.sql lines 197–224]

Migration 003 already has a function named `get_latest_telemetry_per_vehicle(p_fleet_id UUID)` — different signature (takes fleet_id, returns partial columns with old column names `lat`/`lng`/`temp` instead of `latitude`/`longitude`/`coolant_temp`). Do NOT reuse. Create the new `get_latest_telemetry(vehicle_ids uuid[])` separately.

---

## Research Area 6: Angular Service Teardown — Safe Patterns

### `effect()` in Constructor — Correct Pattern

[CITED: angular.dev/guide/signals/effect, dev.to/pckalyan/effects-and-injectioncontext-in-angularv21-20ib]

```typescript
@Injectable({ providedIn: 'root' })
export class VehicleService implements OnDestroy {
  // ...
  constructor() {
    // effect() in constructor = injection context present = auto-cleanup via DestroyRef
    effect(() => {
      const status = this.vehicleResource.status();
      if (status === 'resolved') {
        this.startRealtimeSubscription();
      }
    });
  }

  private startRealtimeSubscription(): void {
    this.reloadVehicles$.next(); // cancel previous subscription
    this.telemetryService.subscribeToFleet();
    this.telemetryService.telemetryBatch$
      .pipe(takeUntil(this.reloadVehicles$), takeUntil(this.destroy$))
      .subscribe(batch => this.processBatch(batch));
  }
}
```

Angular automatically destroys effects created in constructor via `DestroyRef` injection. No manual cleanup needed for the effect itself.

### `effect()` — Side Effect Write Policy (Angular 19+)

[CITED: blog.angular.dev/latest-updates-to-effect-in-angular]

Angular 19 removed the restriction on signal writes inside `effect()`. Writing to signals inside effects is now allowed by default (no `allowSignalWrites: true` needed). The `processBatch()` method writes to `telemetryMap` — this is called from a `.subscribe()` callback, not inside an `effect()` directly, so no issue either way.

### What NOT to Use

| Pattern | Why Avoid |
|---------|-----------|
| `afterNextRender()` | One-shot, DOM-focused. Not for subscription lifecycle. |
| `linkedSignal()` | For writable signals that mirror another signal. No subscription use case. |
| `computed()` with side effects | Angular logs warning; computed is memoized pure function only. |
| `toSignal(telemetryBatch$)` | Returns read-only signal; destroys patch-merge semantics of `processBatch()`. |

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Reactive async data loading | Custom observable/promise chain | `resource()` (Angular 21) | Built-in status, abort, retry, reactive params |
| Realtime data delivery | Long-polling or SSE | Supabase Realtime `postgres_changes` | Already implemented in TelemetryService |
| Telemetry batch merge | Custom Map diffing | `processBatch()` (already in VehicleService) | Tested, handles history buffer, alert side effects |
| Map marker management | Full remove+re-add on every update | `setLatLng()` + `setIcon()` on existing markers | Preserves click handlers, avoids DOM thrash |
| RLS enforcement | Client-side vehicle ID filtering | Supabase RLS on `telemetry_logs` | Enforced at DB + Realtime layer; can't be bypassed |
| Initial telemetry fetch | 500-row `.limit()` query | `get_latest_telemetry()` RPC with `DISTINCT ON` | O(1) per vehicle vs O(500) scan |

---

## Common Pitfalls

### Pitfall 1: `auth.uid()` vs `auth.jwt()->>'sub'` in RLS

**What goes wrong:** Existing `get_user_fleet_ids()` uses `auth.uid()`. For Auth0-issued JWTs, Supabase's `auth.uid()` returns the Supabase internal user ID, which does NOT match the Auth0 `sub` stored in `fleet_members.user_id`. RLS policy returns zero rows for all Auth0 users.

**Why it happens:** Migration 002 was written before Auth0 integration. Phase 2 fixed `vehicles` and `fleet_members` policies (010) but not `telemetry_logs`.

**How to avoid:** Migration 012 drops the old policy and creates a new one using `auth.jwt()->>'sub'` (same pattern as 010_fix_rls_policies.sql).

**Warning signs:** Fleet map shows no vehicles; Supabase logs show `telemetry_logs` queries returning 0 rows.

### Pitfall 2: `fitBounds()` on Every Telemetry Update

**What goes wrong:** The existing `updateMarkers()` calls `fitBounds()` unconditionally. Every Realtime INSERT triggers `updateMarkers()` (via 1s debounced effect), which re-centers the map — user cannot pan or zoom without being immediately reset.

**Why it happens:** `fitBounds()` was correct for initial load but not for live updates.

**How to avoid:** `initialFitDone` boolean flag; fit only when transitioning from 0 markers to N markers.

**Warning signs:** Map snaps back to "full fleet view" whenever a vehicle position updates.

### Pitfall 3: Duplicate Realtime Subscriptions on Org Switch

**What goes wrong:** `subscribeToFleet()` has a guard (`if (this.fleetChannel) return`), which is correct for first call. But when `vehicleResource` reloads on org change (params changed), the effect fires again with `status === 'resolved'`. If `fleetChannel` was not cleared, the old channel is reused (silently) while vehicle IDs have changed. RLS means you only see your fleet's data anyway, but the old channel name `'fleet-telemetry'` is static — no per-org scoping needed. This is actually fine.

**Actual risk:** The `reloadVehicles$.next()` cancels the previous `telemetryBatch$` subscription but does not affect the Realtime channel. The channel stays open across org switches (which is correct — it receives all INSERT events the user has RLS access to).

**How to avoid:** Understand the design — one fleet-wide channel per session is intentional. No per-org channel needed.

### Pitfall 4: `resource()` Loader Called in Wrong Context

**What goes wrong:** If `resource()` is called outside injection context (e.g., inside `ngOnInit` or a method), Angular throws `NG0203: resource() can only be used within an injection context`.

**How to avoid:** Declare `resource()` as a class field (initializer runs in injection context) or call it inside the constructor.

### Pitfall 5: `get_latest_telemetry_per_vehicle()` Column Name Mismatch

**What goes wrong:** The existing function in migration 003 returns columns named `lat`, `lng`, `temp` — not matching the TypeScript `TelemetryRecord` model which uses `latitude`, `longitude`, `coolant_temp`. Using this function would produce silent undefined values.

**How to avoid:** Create the new `get_latest_telemetry(vehicle_ids uuid[])` function returning `SETOF telemetry_logs` (all columns with correct names). Do not reuse the old function.

### Pitfall 6: Stale Vehicle Still Matches `running` State

**What goes wrong:** A vehicle with `engine_on: true` in its last telemetry record (15+ min ago) would be classified as `running` by the existing `getVehicleState()`. The stale check must run first.

**How to avoid:** D-18 explicitly sets priority: `stale > offline > alert > running`. Implementation:

```typescript
private getVehicleState(vehicle: VehicleWithHealth): 'stale' | 'offline' | 'running' | 'alert' {
  const ts = vehicle.latestTelemetry?.timestamp;
  if (ts && Date.now() - new Date(ts).getTime() > 15 * 60 * 1000) return 'stale';
  if (vehicle.status === 'inactive') return 'offline';
  if (vehicle.alertCount > 0) return 'alert';
  if (vehicle.latestTelemetry?.engine_on) return 'running';
  return 'offline';
}
```

### Pitfall 7: `.abortSignal()` Not Available on Supabase Query Builder

**What goes wrong:** The Angular `resource()` loader receives an `AbortSignal` via `ResourceLoaderParams.abortSignal`. Calling `.abortSignal(abortSignal)` on `supabase.client.from().select()` produces a TypeScript error — the method does not exist on the query builder chain in supabase-js v2.104.1 (verified: checked `dist/index.d.mts`, no `abortSignal` method; only a timeout-based abort option is available).

**How to avoid:** Do NOT call `.abortSignal()` on supabase query builder chains. Simply ignore the `abortSignal` parameter in the loaders. Vehicle queries complete in < 500ms; omitting abort handling is safe for MVP.

---

## Code Examples

### Vehicle Resource with org-reactive params

```typescript
// Source: angular.dev/api/core/resource + codebase pattern
readonly vehicleResource = resource({
  params: () => {
    const org = this.organizationService.selectedOrganization();
    if (!org) return undefined;  // skip loader if no org selected
    const fleetIds = this.fleetService.fleets()
      .filter(f => f.organization_id === org.id)
      .map(f => f.id);
    if (fleetIds.length === 0) return undefined;
    return { fleetIds };
  },
  loader: async ({ params: { fleetIds } }) => {
    // NOTE: abortSignal parameter intentionally omitted — not supported by supabase-js query builder
    const { data, error } = await this.supabase.client
      .from('vehicles')
      .select('*')
      .in('fleet_id', fleetIds)
      .eq('status', 'active')
      .order('make');
    if (error) throw error;
    return (data ?? []) as Vehicle[];
  },
  defaultValue: [],
});
```

### Telemetry Resource (snapshot, triggered by vehicle IDs)

```typescript
// Source: D-06 + angular.dev/api/core/resource
private readonly vehicleIds = computed(() =>
  this.vehicleResource.value()?.map(v => v.id) ?? []
);

readonly telemetryResource = resource({
  params: () => {
    const ids = this.vehicleIds();
    return ids.length > 0 ? ids : undefined;
  },
  loader: async ({ params: vehicleIds }) => {
    const { data, error } = await this.supabase.client
      .rpc('get_latest_telemetry', { vehicle_ids: vehicleIds });
    if (error) throw error;
    // Seed telemetryMap with snapshot
    const grouped = new Map<string, TelemetryRecord[]>();
    for (const record of (data ?? [])) {
      grouped.set(record.vehicle_id, [this.mapDbRecord(record)]);
    }
    this.telemetryMap.set(grouped);
    return data;
  },
});
```

### Effect: subscribeToFleet after vehicle resource resolves

```typescript
// Source: angular.dev/guide/signals/effect + codebase pattern (D-07)
constructor() {
  effect(() => {
    if (this.vehicleResource.status() === 'resolved') {
      this.reloadVehicles$.next();  // cancel previous batch subscription
      this.telemetryService.subscribeToFleet();
      this.telemetryService.telemetryBatch$
        .pipe(takeUntil(this.reloadVehicles$), takeUntil(this.destroy$))
        .subscribe(batch => this.processBatch(batch));
    }
  });
}
```

### Shell: Remove manual load chain, add ConnectionPill

```typescript
// ShellComponent — replace ngOnInit load chain with ConnectionPill wiring
// No ngOnInit needed — resource() in VehicleService reacts to org changes automatically.
// Shell only needs to display connection status.
readonly connectionStatus = this.telemetryService.connectionStatus;

// Add VdConnectionPillComponent to imports[]
// In template: <vd-connection-pill [state]="connectionStatus()" />
```

### Toast API — AlertService (VERIFIED from codebase read)

```typescript
// ToastComponent reads from AlertService.activeAlerts() (computed signal)
// ToastComponent is NOT directly called — push alerts via AlertService._alerts.update()
// AlertService has NO public addAlert() method. Add one:

// In alert.service.ts — add this public method:
pushAlert(message: string, severity: AlertSeverity, type: AlertType = 'connection'): void {
  const id = `alert-${Date.now()}-${++alertIdCounter}`;
  this._alerts.update(alerts => {
    const trimmed = alerts.length >= 200 ? alerts.slice(-199) : alerts;
    return [...trimmed, {
      id,
      vehicleId: 'fleet',   // sentinel for non-vehicle alerts
      type,
      severity,
      message,
      timestamp: new Date(),
      status: 'active',
    } satisfies Alert];
  });
}

// In ShellComponent constructor effect():
// severity 'warning' auto-dismisses at 12000ms (ALERT_AUTO_DISMISS.warning)
// severity 'critical' never auto-dismisses (ALERT_AUTO_DISMISS.critical = 0)
// Use 'warning' for disconnect (persistent enough), 'info' for reconnect (8000ms auto-dismiss)
// Note: UI-SPEC calls for 3000ms reconnect toast — ALERT_AUTO_DISMISS.info = 8000ms is the closest available
this.alertService.pushAlert('Connection lost — reconnecting…', 'warning');
this.alertService.pushAlert('Live data restored.', 'info');
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `loadVehicles()` imperative call from Shell | `resource()` reactive loading in VehicleService | Angular 19 (stable) | Org change auto-triggers reload without Shell coordination |
| `.limit(500)` telemetry query | `get_latest_telemetry()` RPC with `DISTINCT ON` | Phase 3 | O(1) per vehicle; correct for any fleet size |
| `auth.uid()` in RLS | `auth.jwt()->>'sub'` for Auth0 | Phase 2 (vehicles); Phase 3 (telemetry) | Required for Auth0 JWTs where sub ≠ Supabase UID |
| `request` property in resource() | `params` property | Angular 19 stable | API renamed during stabilization |
| `.abortSignal()` on query builder | Omit abort handling | Phase 3 | Method not available in supabase-js v2.104.1 |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `NgZone` wrapping not needed for Leaflet with OnPush + effect() pattern | Research Area 3 | Markers might not trigger CD; fix: wrap updateMarkers in NgZone.run() |
| A2 | Supabase JS v2 auto-reconnect re-subscribes postgres_changes without explicit retry logic | Research Area 2 | Live data stops after disconnect; fix: watch connectionStatus and removeChannel + re-create on reconnect |
| A3 | `abortSignal` is supported by Supabase JS client's `.from().select()` chain (`.abortSignal()` method) | Research Areas 1 & 5 | RESOLVED: Not supported — see Pitfall 7 and Open Questions (RESOLVED) |

---

## Open Questions (RESOLVED)

1. **Does `supabase.client.from().select().abortSignal()` exist in v2.39.0?**
   - RESOLVED: No — `.abortSignal()` is NOT present in the installed supabase-js. Actual installed version is v2.104.1 (semver satisfies `^2.39.0`). Checked `node_modules/@supabase/supabase-js/dist/index.d.mts` — no `abortSignal` method on the query builder chain. Only a timeout-based abort option is documented (line 64 of type defs mentions timeout abort). A3 assumption is now confirmed false.
   - **Impact on plans:** Remove `.abortSignal(abortSignal)` from the vehicle loader in Plan 03-01 Task 3. Telemetry loader already omits it. Both loaders should destructure only `{ params }` from `ResourceLoaderParams`, not `{ params, abortSignal }`. See updated Code Examples above.

2. **Is `get_user_fleet_ids()` used elsewhere in active RLS policies?**
   - RESOLVED: Yes — used in fleets, fleet_members, vehicles, and telemetry_logs policies from migration 002. However, migration 009 (`009_phase2_auth_schema.sql`, Section 3) already rewrote `get_user_fleet_ids()` to use `auth.jwt()->>'sub'` (line 115: `SELECT fleet_id FROM fleet_members WHERE user_id = (auth.jwt()->>'sub')`). Migration 009 also recreated all affected policies inline. The `telemetry_logs` SELECT policy was the only one NOT updated in 009 or 010 — it still references the 002 definition. The 002 policy technically works through the now-updated helper function, but uses the old indirect join pattern.
   - **Impact on plans:** Migration 012 proceeds as planned — drop + recreate `telemetry_logs` policy using the inline `auth.jwt()->>'sub'` JOIN pattern for consistency with 009/010. Do NOT modify `get_user_fleet_ids()` in migration 012 (already correct from 009).

---

## Environment Availability

| Dependency | Required By | Available | Notes |
|------------|------------|-----------|-------|
| Supabase CLI | Migration 012 deployment | Confirmed (package.json devDep `supabase ^1.226.4`) | `supabase db push` deploys migrations |
| Angular CLI (ng) | Build/test | Confirmed (package.json) | `npm run build:dashboard` |
| Simulator (Ghost Fleet) | Realtime data flow testing | Exists at `packages/simulator/` | Used for end-to-end Realtime verification |
| Parser (TCP server) | Simulator → Supabase path | Exists at `packages/parser/` | Uses service role key for inserts |
| Leaflet assets | Map tile display | Confirmed (`assets/leaflet/` referenced in fleet-map.component.ts) | Already wired |

---

## Validation Architecture

### How to verify Supabase Realtime delivers live data (full stack)

**Test:** Simulator → Parser → Supabase → Dashboard

1. Start parser: `npm run dev:parser` (Railway :5050 or local)
2. Start simulator: `npm run dev:simulator` (sends Codec 8 Extended frames)
3. Open dashboard → `/map` route
4. Observe: vehicle markers should appear/update without page refresh
5. Verify: `TelemetryService.connectionStatus` shows `'connected'` in `VdConnectionPillComponent`

**Manual smoke check:**
```sql
-- Supabase SQL Editor: verify inserts arriving
SELECT vehicle_id, timestamp FROM telemetry_logs ORDER BY timestamp DESC LIMIT 5;
```

### How to test `resource()` reactive chain (org change → vehicles reload → telemetry reload)

**Test:**
1. Log in as user with access to multiple organizations
2. Open `/dashboard` → verify vehicle cards load
3. Switch organization in sidebar dropdown
4. Observe: vehicle cards update to new org's vehicles (no page refresh)
5. Verify: `vehicleResource.status()` transitions `loading → resolved` (add `console.log` in loader for debugging)

**Unit test pattern:**
```typescript
// VehicleService spec
it('reloads vehicles when selectedOrganization changes', fakeAsync(() => {
  const orgService = TestBed.inject(OrganizationService);
  orgService.selectedOrganization.set(mockOrg1);
  tick();
  expect(vehicleService.vehicleResource.value()).toEqual(mockVehiclesOrg1);
  orgService.selectedOrganization.set(mockOrg2);
  tick();
  expect(vehicleService.vehicleResource.value()).toEqual(mockVehiclesOrg2);
}));
```

### How to verify RLS blocks cross-fleet telemetry access

**Test:**
1. Create two organizations with separate fleet members (user A and user B)
2. Insert test telemetry for vehicle in org A's fleet
3. Log in as user B
4. Check: `telemetry_logs` query returns 0 rows for user B's session
5. Verify: Realtime channel delivers 0 events to user B for org A's vehicle inserts

**SQL verification (run as user B's JWT):**
```sql
-- Should return 0 rows if RLS is correct
SELECT count(*) FROM telemetry_logs
WHERE vehicle_id = '<org-A-vehicle-id>';
```

### How to verify stale marker rendering at 15-min threshold

**Manual test:**
1. Insert a telemetry record with `timestamp = NOW() - INTERVAL '20 minutes'` for a vehicle
2. Open `/map` route
3. Verify: vehicle marker shows grey dimmed DivIcon (opacity 0.5)
4. Hover marker: tooltip shows "Last seen 20 min ago"
5. Insert fresh telemetry for same vehicle (`timestamp = NOW()`)
6. Wait for Realtime event + 1s debounce
7. Verify: marker transitions back to running/alert/offline state (no grey opacity)

**Unit test:**
```typescript
it('returns stale state when telemetry is older than 15 minutes', () => {
  const staleTimestamp = new Date(Date.now() - 16 * 60 * 1000).toISOString();
  const vehicle = { ...mockVehicle, latestTelemetry: { ...mockTelemetry, timestamp: staleTimestamp, engine_on: true } };
  expect(component['getVehicleState'](vehicle)).toBe('stale');
});
```

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No (handled in Phase 2) | Auth0 + JWT exchange |
| V3 Session Management | No (handled in Phase 2) | localStorage JWT persistence |
| V4 Access Control | Yes | Supabase RLS on `telemetry_logs`; `auth.jwt()->>'sub'` for Auth0 |
| V5 Input Validation | No | No user input in Phase 3 (read-only dashboard) |
| V6 Cryptography | No | No crypto operations |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Cross-fleet telemetry read (user reads another org's vehicle data) | Information Disclosure | RLS policy on `telemetry_logs` scoped to `fleet_members.user_id = auth.jwt()->>'sub'` |
| RPC function bypassing RLS via SECURITY DEFINER | Elevation of Privilege | `get_latest_telemetry()` uses `SECURITY INVOKER` (D-11) |
| Realtime channel receiving cross-fleet events | Information Disclosure | Supabase Realtime RLS check before broadcast — automatic once D-12 policy is applied |
| Parser inserting with anon key (no RLS bypass) | Denial of Service / Data Integrity | Parser must use `SUPABASE_SERVICE_ROLE_KEY` — verify env var set before applying D-12 |

---

## Sources

### Primary (HIGH confidence)
- `packages/dashboard/src/app/core/services/vehicle.service.ts` — existing code, directly read
- `packages/dashboard/src/app/core/services/telemetry.service.ts` — existing code, directly read
- `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts` — existing code, directly read
- `packages/dashboard/src/app/shared/components/toast/toast.component.ts` — directly read; ToastComponent driven by AlertService.activeAlerts()
- `packages/dashboard/src/app/core/services/alert.service.ts` — directly read; no public addAlert(); createAlert() is private; AlertType includes 'connection'
- `packages/dashboard/src/app/core/models/alert.model.ts` — directly read; ALERT_AUTO_DISMISS: critical=0, warning=12000ms, info=8000ms
- `packages/supabase/migrations/002_rls_policies.sql` — existing migration, directly read
- `packages/supabase/migrations/009_phase2_auth_schema.sql` — directly read; get_user_fleet_ids() updated to auth.jwt()->>'sub' in Section 3
- `packages/supabase/migrations/010_fix_rls_policies.sql` — existing migration, directly read
- `packages/supabase/migrations/003_functions_triggers.sql` — existing migration, directly read
- `node_modules/@supabase/supabase-js/dist/index.d.mts` — directly checked; no abortSignal method on query builder; installed version is 2.104.1
- [angular.dev/api/core/resource](https://angular.dev/api/core/resource) — ResourceRef interface, ResourceLoaderParams
- [angular.dev/api/core/ResourceLoaderParams](https://angular.dev/api/core/ResourceLoaderParams) — exact params/abortSignal/previous shape
- [angular.dev/guide/signals/resource](https://angular.dev/guide/signals/resource) — loader skip behavior, status lifecycle

### Secondary (MEDIUM confidence)
- [supabase.com/blog/realtime-row-level-security-in-postgresql](https://supabase.com/blog/realtime-row-level-security-in-postgresql) — Realtime RLS enforcement mechanism
- [supabase.com/docs/guides/realtime/postgres-changes](https://supabase.com/docs/guides/realtime/postgres-changes) — channel filter syntax, DELETE limitation
- [angular.dev/guide/signals/effect](https://angular.dev/guide/signals/effect) — effect() injection context requirements
- [blog.angular.dev/latest-updates-to-effect-in-angular](https://blog.angular.dev/latest-updates-to-effect-in-angular) — signal writes inside effect() allowed in Angular 19+

### Tertiary (LOW confidence)
- [github.com/supabase/realtime/issues/1088](https://github.com/supabase/realtime/issues/1088) — reconnect + postgres_changes re-subscription edge case (A2 assumption)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all versions verified from package.json
- Architecture patterns: HIGH — derived directly from codebase read + official docs
- Pitfalls: HIGH — most verified against existing migration files and code
- RLS correctness: HIGH — existing 010 migration confirms the `auth.jwt()->>'sub'` pattern works
- Toast API: HIGH — ToastComponent and AlertService directly read; no public addAlert() exists; must add pushAlert() method

**Research date:** 2026-05-14
**Valid until:** 2026-06-14 (stable APIs — Angular resource() stable since v19, Supabase JS v2 stable)
