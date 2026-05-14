# Phase 3: Live Fleet Dashboard — Pattern Map

**Mapped:** 2026-05-14
**Files analyzed:** 8 new/modified files
**Analogs found:** 8 / 8

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|---|---|---|---|---|
| `packages/dashboard/src/app/core/services/vehicle.service.ts` | service | CRUD + event-driven | self (refactor) | self |
| `packages/dashboard/src/app/core/services/telemetry.service.ts` | service | event-driven | self (no changes) | reference only |
| `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts` | component | event-driven | self (refactor) | self |
| `packages/dashboard/src/app/features/dashboard/dashboard.component.ts` | component | request-response | `dashboard.component.ts` (self) | self |
| `packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-grid.component.ts` | component | request-response | `vehicle-grid.component.ts` (self) | self |
| `packages/dashboard/src/app/layout/shell/shell.component.ts` | component | request-response | self (refactor) | self |
| `packages/supabase/migrations/012_telemetry_rls_and_rpc.sql` | migration | CRUD | `010_fix_rls_policies.sql` | exact |

---

## Pattern Assignments

### `packages/dashboard/src/app/core/services/vehicle.service.ts` (service, CRUD + event-driven)

**Change type:** Refactor — replace `loadVehicles()` + `loadInitialTelemetry()` imperative calls with `resource()` reactive pattern. Add `effect()` bridge to `subscribeToFleet()`.

**Analog for resource() pattern:** `packages/dashboard/src/app/core/services/organization.service.ts`

**Key observation:** OrganizationService does NOT yet use `resource()` — it uses manual `loadOrganizations()` with `isLoading`/`error` signals (lines 17–39). The `resource()` pattern is new to this codebase in Phase 3. Use RESEARCH.md examples as the primary reference for the API shape.

**Existing inject pattern to preserve** (vehicle.service.ts lines 1–24):
```typescript
import { Injectable, computed, inject, signal, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { SupabaseService } from './supabase.service';
import { TelemetryService } from './telemetry.service';
import { OrganizationService } from './organization.service';
import { FleetService } from './fleet.service';

@Injectable({ providedIn: 'root' })
export class VehicleService implements OnDestroy {
  private readonly supabase = inject(SupabaseService);
  private readonly telemetryService = inject(TelemetryService);
  private readonly organizationService = inject(OrganizationService);
  private readonly fleetService = inject(FleetService);
  private readonly destroy$ = new Subject<void>();
  private readonly reloadVehicles$ = new Subject<void>();
```

**Add to imports (new):**
```typescript
import { resource, effect } from '@angular/core';
```

**Core resource() pattern to ADD** (field initializer — injection context):
```typescript
// vehicleResource replaces loadVehicles() signal management
readonly vehicleResource = resource({
  params: () => {
    const org = this.organizationService.selectedOrganization();
    if (!org) return undefined;
    const fleetIds = this.fleetService.fleets()
      .filter(f => f.organization_id === org.id)
      .map(f => f.id);
    if (fleetIds.length === 0) return undefined;
    return { fleetIds };
  },
  loader: async ({ params: { fleetIds }, abortSignal }) => {
    const { data, error } = await this.supabase.client
      .from('vehicles')
      .select('*')
      .in('fleet_id', fleetIds)
      .eq('status', 'active')
      .order('make')
      .abortSignal(abortSignal);
    if (error) throw error;
    return (data ?? []) as Vehicle[];
  },
  defaultValue: [],
});

// Derived vehicleIds for telemetryResource guard
private readonly vehicleIds = computed(() =>
  this.vehicleResource.value()?.map(v => v.id) ?? []
);

// telemetryResource replaces loadInitialTelemetry()
readonly telemetryResource = resource({
  params: () => {
    const ids = this.vehicleIds();
    return ids.length > 0 ? ids : undefined;  // undefined → loader skips, status = 'idle'
  },
  loader: async ({ params: vehicleIds }) => {
    const { data, error } = await this.supabase.client
      .rpc('get_latest_telemetry', { vehicle_ids: vehicleIds });
    if (error) throw error;
    const grouped = new Map<string, TelemetryRecord[]>();
    for (const record of (data ?? [])) {
      grouped.set(record.vehicle_id, [this.mapDbRecord(record)]);
    }
    this.telemetryMap.set(grouped);
    return data;
  },
});
```

**effect() bridge pattern to ADD in constructor** (D-07):
```typescript
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

**Existing processBatch() to PRESERVE unchanged** (vehicle.service.ts lines 228–256):
```typescript
private processBatch(batch: TelemetryRecord[]): void {
  const latestPerVehicle = new Map<string, TelemetryRecord>();
  for (const record of batch) {
    const existing = latestPerVehicle.get(record.vehicle_id);
    if (!existing || new Date(record.timestamp) > new Date(existing.timestamp)) {
      latestPerVehicle.set(record.vehicle_id, record);
    }
  }
  this.telemetryMap.update((map) => {
    const newMap = new Map(map);
    latestPerVehicle.forEach((record, vehicleId) => {
      const existing = newMap.get(vehicleId) ?? [];
      newMap.set(vehicleId, [record, ...existing].slice(0, MAX_TELEMETRY_HISTORY));
      const vehicle = this.vehicles().find((v) => v.id === vehicleId);
      this.alertService.checkAndCreateAlerts(record, vehicle ? getVehicleDisplayName(vehicle) : undefined);
    });
    return newMap;
  });
}
```

**vehiclesWithHealth computed to UPDATE** — change `this.vehicles()` to `this.vehicleResource.value() ?? []` (vehicle.service.ts lines 40–54):
```typescript
readonly vehiclesWithHealth = computed<VehicleWithHealth[]>(() => {
  const map = this.telemetryMap();
  return (this.vehicleResource.value() ?? []).map((v) => {
    const history = map.get(v.id) ?? [];
    const latest = history[0];
    return {
      ...v,
      latestTelemetry: latest,
      healthScore: this.calculateHealthScore(latest),
      alertCount: this.alertService.activeAlerts().filter(a => a.vehicleId === v.id).length,
    };
  });
});
```

**DELETE:** `loadVehicles()` (lines 77–126), `loadInitialTelemetry()` (lines 198–218), `isLoading` signal (line 31), `vehicles` signal (line 28) — replaced by `vehicleResource`.

**KEEP:** `calculateHealthScore()`, `mapDbRecord()`, `processBatch()`, `telemetryMap`, `selectedVehicleId`, `ngOnDestroy()`, all CRUD methods (createVehicle, updateVehicle, deleteVehicle).

---

### `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts` (component, event-driven)

**Change type:** Refactor — add `stale` state to `getVehicleState()`, fix `fitBounds()` to run once, add stale tooltip.

**Existing imports pattern to PRESERVE** (lines 1–15):
```typescript
import { AfterViewInit, ChangeDetectionStrategy, Component, OnDestroy,
         computed, effect, inject, signal } from '@angular/core';
import * as L from 'leaflet';
import { VehicleService } from '../../core/services/vehicle.service';
import { VehicleWithHealth, getVehicleDisplayName } from '../../core/models/vehicle.model';
import { VehicleDetailPanelComponent } from './vehicle-detail-panel/vehicle-detail-panel.component';
```

**getVehicleState() — REPLACE** (lines 180–185):
```typescript
// OLD (3 states):
private getVehicleState(vehicle: VehicleWithHealth): 'offline' | 'running' | 'alert' {
  if (vehicle.status === 'inactive') return 'offline';
  if (vehicle.alertCount > 0) return 'alert';
  if (vehicle.latestTelemetry?.engine_on) return 'running';
  return 'offline';
}

// NEW (4 states — stale checked first per D-18):
private getVehicleState(vehicle: VehicleWithHealth): 'stale' | 'offline' | 'running' | 'alert' {
  const ts = vehicle.latestTelemetry?.timestamp;
  if (ts && Date.now() - new Date(ts).getTime() > 15 * 60 * 1000) return 'stale';
  if (vehicle.status === 'inactive') return 'offline';
  if (vehicle.alertCount > 0) return 'alert';
  if (vehicle.latestTelemetry?.engine_on) return 'running';
  return 'offline';
}
```

**getMarkerIcon() — ADD stale branch** (extends lines 153–178):
```typescript
// Add stale to colors map and apply opacity:
const colors: Record<string, string> = {
  offline: '#6b7280',
  running: '#22c55e',
  alert:   '#ef4444',
  stale:   '#6b7280',   // same grey as offline
};
const opacity = state === 'stale' ? '0.5' : '1';
const pulse = state === 'running' || state === 'alert' ? 'pulse' : '';

return L.divIcon({
  html: `
    <div class="map-marker map-marker--${state} ${pulse}"
         style="opacity:${opacity}" aria-hidden="true">
      <svg width="32" height="32" viewBox="0 0 32 32" fill="none">
        <circle cx="16" cy="16" r="12" fill="${color}" opacity="0.2"/>
        <circle cx="16" cy="16" r="7" fill="${color}"/>
        <circle cx="16" cy="16" r="3" fill="white"/>
      </svg>
    </div>
  `,
  className: '',
  iconSize: [32, 32],
  iconAnchor: [16, 16],
  popupAnchor: [0, -16],
});
```

**fitBounds() fix — ADD `initialFitDone` flag** (class field + updateMarkers change at lines 144–150):
```typescript
// Add class field:
private initialFitDone = false;

// Replace unconditional fitBounds block (lines 144–150) with:
if (validVehicles.length > 0 && !this.initialFitDone) {
  const bounds = L.latLngBounds(
    validVehicles.map(v => [v.latestTelemetry!.latitude!, v.latestTelemetry!.longitude!]),
  );
  this.map.fitBounds(bounds.pad(0.1));
  this.initialFitDone = true;
}
```

**Stale tooltip — ADD inside updateMarkers() marker update block** (after line 114):
```typescript
// After existing.marker.setIcon() or new marker creation:
const state = this.getVehicleState(vehicle);
if (state === 'stale') {
  const minsAgo = Math.floor(
    (Date.now() - new Date(vehicle.latestTelemetry!.timestamp).getTime()) / 60000
  );
  marker.bindTooltip(`Last seen ${minsAgo} min ago`, { permanent: false });
} else {
  marker.unbindTooltip();
}
```

**Empty state — ADD to template** (no TS change needed — handled in HTML template):
Map tiles visible at default center `[32.0853, 34.7818]` (already set in `initMap()` line 83). Add centered overlay in template when `vehicles().length === 0 && !vehicleService.vehicleResource.isLoading()`.

---

### `packages/dashboard/src/app/layout/shell/shell.component.ts` (component, request-response)

**Change type:** Remove manual load chain from `ngOnInit`, add `VdConnectionPillComponent` + connection toast `effect()`.

**Existing import block** (lines 1–19) — ADD:
```typescript
import { effect } from '@angular/core';
import { TelemetryService } from '../../core/services/telemetry.service';
import { VdConnectionPillComponent } from '../../shared/ui/connection-pill/connection-pill.component';
// ConnectionState type also available from connection-pill.component.ts line 3
```

**Remove from imports[]:** No removals — keep existing. Add `VdConnectionPillComponent`.

**DELETE ngOnInit entirely** (lines 54–62):
```typescript
// DELETE THIS — resource() in VehicleService auto-reacts to org changes:
ngOnInit(): void {
  this.organizationService.loadOrganizations().then(() => {
    this.fleetService.loadFleets().then(() => {
      this.vehicleService.loadVehicles().then(() => {
        this.vehicleService.loadInitialTelemetry();
      });
    });
  });
}
```

**ADD in constructor** (after breakpointObserver subscription, lines 43–51):
```typescript
// Inject TelemetryService
private readonly telemetryService = inject(TelemetryService);

// Expose connectionStatus for template
readonly connectionStatus = this.telemetryService.connectionStatus;

// Toast on connection status transitions (D-20)
// Note: ToastComponent is driven by AlertService; for connection toasts,
// call alertService.addAlert() on transition inside the effect.
private previousConnectionStatus: string | null = null;
// In constructor:
effect(() => {
  const status = this.telemetryService.connectionStatus();
  if (this.previousConnectionStatus !== null && this.previousConnectionStatus !== status) {
    // Emit connection change toast via AlertService or a dedicated method
    // Exact integration depends on AlertService API (check alert.service.ts)
  }
  this.previousConnectionStatus = status;
});
```

**Template addition — ConnectionPill** (in shell header, pattern from connection-pill.component.ts lines 59–64):
```html
<!-- VdConnectionPillComponent inputs: state (ConnectionState), label (string|null) -->
<vd-connection-pill [state]="connectionStatus()" />
```

**Component class changes:** Remove `implements OnInit`, add `implements OnDestroy` if not already present (effect auto-cleans via DestroyRef).

---

### `packages/dashboard/src/app/features/dashboard/dashboard.component.ts` (component, request-response)

**Change type:** Add resource status passthrough for loading/empty states (D-23, D-25).

**Existing pattern to EXTEND** (lines 1–24 — entire file, small):
```typescript
import { ChangeDetectionStrategy, Component, inject } from '@angular/core';
import { VehicleService } from '../../core/services/vehicle.service';
// ...
export class DashboardComponent {
  private readonly vehicleService = inject(VehicleService);
  readonly vehicleCount = this.vehicleService.onlineVehicleCount;
  // ...
}
```

**ADD signals for template-driven states:**
```typescript
// Expose resource signals for template
readonly isLoading = this.vehicleService.vehicleResource.isLoading;
readonly vehiclesWithHealth = this.vehicleService.vehiclesWithHealth;
readonly resourceStatus = this.vehicleService.vehicleResource.status;
```

**Template empty state pattern** (D-23) — HTML only, no TS change:
```html
@if (isLoading()) {
  <app-loading-spinner message="Loading fleet..." [fullPage]="true" />
} @else if (vehiclesWithHealth().length === 0) {
  <!-- Empty state: illustration + message + [+ Add Vehicle] button → /fleet-management -->
} @else {
  <app-vehicle-grid />
}
```

**LoadingSpinnerComponent inputs** (from loading-spinner.component.ts lines 47–50):
- `message` input: string, default `'Loading...'`
- `fullPage` input: boolean, default `false`

---

### `packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-grid.component.ts` (component, request-response)

**Change type:** Swap `isLoading` from old `vehicleService.isLoading` signal to `vehicleResource.isLoading`, add empty state binding.

**Existing pattern** (lines 19–26 — no structural change):
```typescript
readonly vehicles = this.vehicleService.vehiclesWithHealth;
readonly isLoading = this.vehicleService.isLoading;      // ← CHANGE TO vehicleResource.isLoading
readonly error = this.vehicleService.error;               // ← CHANGE TO vehicleResource.error
```

**Updated bindings:**
```typescript
readonly isLoading = this.vehicleService.vehicleResource.isLoading;
readonly hasError = computed(() => this.vehicleService.vehicleResource.status() === 'error');
readonly errorMessage = this.vehicleService.vehicleResource.error;
```

Add `computed` to imports. No other structural changes.

---

### `packages/supabase/migrations/012_telemetry_rls_and_rpc.sql` (migration, CRUD)

**Analog:** `packages/supabase/migrations/010_fix_rls_policies.sql`

**Exact structure to copy** (010 lines 7–54):
```sql
BEGIN;

-- ============================================================
-- [Comment block explaining what this migration fixes]
-- ============================================================

DROP POLICY IF EXISTS "policy name" ON table_name;

CREATE POLICY "policy name" ON table_name
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM ...
            WHERE ... AND user_id = (auth.jwt()->>'sub')
        )
    );

COMMIT;
```

**auth.jwt()->>'sub' pattern** (010 line 27 — fleet_members, line 47 — vehicles):
```sql
-- fleet_members pattern (line 27):
FOR SELECT USING (user_id = (auth.jwt()->>'sub'));

-- vehicles pattern (lines 39–52) — JOIN through org hierarchy:
FOR SELECT USING (
    EXISTS (
        SELECT 1 FROM organizations o
        JOIN fleets f ON f.organization_id = o.id
        WHERE f.id = vehicles.fleet_id
          AND o.owner_id = (auth.jwt()->>'sub')
    )
    OR
    fleet_id IN (
        SELECT fm.fleet_id FROM fleet_members fm
        WHERE fm.user_id = (auth.jwt()->>'sub')
    )
);
```

**Full migration 012 pattern** (copy structure from 010, apply D-12/D-10/D-11):
```sql
-- VitalsDrive Phase 3 Telemetry RLS Fix + RPC
-- Migration: 012_telemetry_rls_and_rpc.sql
-- D-12: Replace auth.uid()-based telemetry_logs policy with auth.jwt()->>'sub'
-- D-10: Add get_latest_telemetry(vehicle_ids uuid[]) RPC for initial snapshot
-- D-11: RPC uses SECURITY INVOKER (caller's RLS context — no bypass)

BEGIN;

-- Step 1: Drop old policy (uses auth.uid() — broken for Auth0 users)
DROP POLICY IF EXISTS "Users can read own fleet telemetry" ON telemetry_logs;

-- Step 2: New policy matching vehicles pattern from 010
CREATE POLICY "Users can read own fleet telemetry" ON telemetry_logs
FOR SELECT USING (
  EXISTS (
    SELECT 1 FROM vehicles v
    JOIN fleet_members fm ON fm.fleet_id = v.fleet_id
    WHERE v.id = telemetry_logs.vehicle_id
      AND fm.user_id = (auth.jwt()->>'sub')
  )
);

-- Step 3: RPC for initial telemetry snapshot (one record per vehicle)
-- SECURITY INVOKER = runs as calling user → RLS applies automatically (D-11)
-- RETURNS SETOF telemetry_logs = all columns match mapDbRecord() expectations
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

**Critical:** Do NOT reuse `get_latest_telemetry_per_vehicle(p_fleet_id UUID)` from migration 003 — different signature and returns `lat`/`lng`/`temp` column names instead of `latitude`/`longitude`/`coolant_temp` expected by `mapDbRecord()`.

---

## Shared Patterns

### Signal injection pattern
**Source:** `vehicle.service.ts` lines 1–8, `fleet-map.component.ts` lines 1–10
**Apply to:** All modified services and components
```typescript
// Always use inject() — no constructor parameter injection
private readonly serviceName = inject(ServiceClass);
```

### ChangeDetectionStrategy.OnPush
**Source:** `fleet-map.component.ts` line 41, `shell.component.ts` line 24, `dashboard.component.ts` line 11
**Apply to:** All components — already set everywhere, do not remove.

### Supabase query error handling
**Source:** `vehicle.service.ts` lines 108–125, `organization.service.ts` lines 22–38
```typescript
const { data, error } = await this.supabase.client.from('table').select('*');
if (error) throw error;  // resource() catches thrown errors → status = 'error'
return data ?? [];
```

### RxJS destroy pattern
**Source:** `vehicle.service.ts` lines 23–24, `telemetry.service.ts` lines 11–12
**Apply to:** VehicleService (already has both subjects — preserve):
```typescript
private readonly destroy$ = new Subject<void>();
private readonly reloadVehicles$ = new Subject<void>();
// takeUntil(this.reloadVehicles$) cancels per-load subscription
// takeUntil(this.destroy$) cancels on service destroy
```

### effect() in constructor (injection context)
**Source:** `fleet-map.component.ts` lines 61–69 (existing effect pattern)
```typescript
constructor() {
  effect(() => {
    const value = this.someSignal();
    // side effects here — no allowSignalWrites needed (Angular 19+)
  });
}
```

### auth.jwt()->>'sub' in RLS
**Source:** `010_fix_rls_policies.sql` lines 27, 47–51
**Apply to:** Migration 012 — every WHERE clause filtering by user identity.

### ConnectionPill component API
**Source:** `connection-pill.component.ts` lines 59–64
```typescript
// Selector: vd-connection-pill
// Inputs: state: ConnectionState ('connected'|'reconnecting'|'disconnected'), label: string|null
// Usage: <vd-connection-pill [state]="connectionStatus()" />
```

### LoadingSpinner component API
**Source:** `loading-spinner.component.ts` lines 47–50
```typescript
// Selector: app-loading-spinner
// Inputs: message: string (default 'Loading...'), fullPage: boolean (default false)
// Usage: <app-loading-spinner message="Loading fleet..." [fullPage]="true" />
```

---

## No Analog Found

No files are truly greenfield — all patterns exist in the codebase. The `resource()` API is new to this codebase; RESEARCH.md §Code Examples serves as the canonical reference since there is no existing `resource()` usage to copy from.

| Pattern | Source | Reason |
|---|---|---|
| `resource()` API (vehicleResource, telemetryResource) | RESEARCH.md §Code Examples | First use of `resource()` in this codebase |
| Toast on connection status change | Needs AlertService inspection | ToastComponent is alert-driven; connection toast integration depends on AlertService.addAlert() API shape |

---

## Metadata

**Analog search scope:** `packages/dashboard/src/`, `packages/supabase/migrations/`
**Files read:** 11
**Pattern extraction date:** 2026-05-14
