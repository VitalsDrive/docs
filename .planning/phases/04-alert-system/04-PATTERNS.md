# Phase 4: Alert System - Pattern Map

**Mapped:** 2026-05-15
**Files analyzed:** 7 new/modified files
**Analogs found:** 7 / 7

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `packages/supabase/migrations/014_alert_trigger_and_rls.sql` | migration | event-driven | `packages/supabase/migrations/003_functions_triggers.sql` + `010_fix_rls_policies.sql` | role-match |
| `packages/dashboard/src/app/core/models/alert.model.ts` | model | — | self (extend existing) | exact |
| `packages/dashboard/src/app/core/services/alert.service.ts` | service | CRUD + event-driven | `packages/dashboard/src/app/core/services/vehicle.service.ts` | exact |
| `packages/dashboard/src/app/features/alerts/alerts.component.ts` | component | CRUD | self (rewrite existing) + `vehicle.service.ts` resource() | exact |
| `packages/dashboard/src/app/features/alerts/alerts.component.html` | template | CRUD | `packages/dashboard/src/app/features/alerts/alerts.component.html` (rewrite) | exact |
| `packages/dashboard/src/app/layout/header/header.component.ts` | component | event-driven | self (extend existing) | exact |
| `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts` | component | event-driven | self (extend existing) | exact |

---

## Pattern Assignments

---

### `packages/supabase/migrations/014_alert_trigger_and_rls.sql` (migration, event-driven)

**Analogs:**
- `packages/supabase/migrations/003_functions_triggers.sql` — trigger function structure + DECLARE/BEGIN/RETURN pattern
- `packages/supabase/migrations/010_fix_rls_policies.sql` — RLS policy pattern with `auth.jwt()->>'sub'`

**Migration wrapper pattern** (`010_fix_rls_policies.sql` lines 7, 54):
```sql
BEGIN;

-- ... all DDL statements ...

COMMIT;
```

**Existing trigger to drop** (`003_functions_triggers.sql` lines 42-60):
```sql
-- The old trigger function uses telemetry_rules (has no rows for MVP).
-- Migration 014 must drop this trigger before creating the new one:
DROP TRIGGER IF EXISTS trigger_generate_alert ON telemetry_logs;
DROP FUNCTION IF EXISTS generate_alert_if_needed();
```

**Trigger function skeleton** (`003_functions_triggers.sql` lines 42-55):
```sql
CREATE OR REPLACE FUNCTION check_telemetry_alerts()
RETURNS TRIGGER AS $$
DECLARE
    v_fleet_id UUID;
BEGIN
    SELECT fleet_id INTO v_fleet_id FROM vehicles WHERE id = NEW.vehicle_id;
    IF v_fleet_id IS NULL THEN
        RETURN NEW;
    END IF;
    -- ... threshold checks ...
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**AFTER INSERT trigger binding** (`003_functions_triggers.sql` lines 30-32):
```sql
CREATE TRIGGER check_telemetry_alerts_trigger
    AFTER INSERT ON telemetry_logs
    FOR EACH ROW EXECUTE FUNCTION check_telemetry_alerts();
```

**RLS SELECT policy pattern** (`010_fix_rls_policies.sql` lines 37-52):
```sql
CREATE POLICY "Users can read own fleet alerts" ON alerts
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM organizations o
            JOIN fleets f ON f.organization_id = o.id
            WHERE f.id = alerts.fleet_id
              AND o.owner_id = (auth.jwt()->>'sub')
        )
        OR
        fleet_id IN (
            SELECT fm.fleet_id FROM fleet_members fm
            WHERE fm.user_id = (auth.jwt()->>'sub')
        )
    );
```

**RLS UPDATE policy — WITH CHECK** (RESEARCH.md pattern, auth pattern from `010`):
```sql
CREATE POLICY "Users can acknowledge own fleet alerts" ON alerts
    FOR UPDATE USING (
        EXISTS (
            SELECT 1 FROM fleet_members fm
            WHERE fm.fleet_id = alerts.fleet_id
              AND fm.user_id = (auth.jwt()->>'sub')
        )
        OR
        EXISTS (
            SELECT 1 FROM fleets f
            JOIN organizations o ON o.id = f.organization_id
            WHERE f.id = alerts.fleet_id
              AND o.owner_id = (auth.jwt()->>'sub')
        )
    )
    WITH CHECK (acknowledged = true);
```

**Realtime publication** (must be in migration — RESEARCH.md Pitfall 6):
```sql
ALTER PUBLICATION supabase_realtime ADD TABLE alerts;
```

**Critical notes for implementer:**
- `alerts.id` is `BIGSERIAL` (integer), NOT UUID — dedup guard uses integer comparison
- DTC FOREACH loop requires a sub-block `DECLARE ... BEGIN ... END` inside the outer function body
- SECURITY DEFINER required so trigger can read `vehicles` and insert `alerts` as postgres role

---

### `packages/dashboard/src/app/core/models/alert.model.ts` (model, extend existing)

**Analog:** self — existing file at lines 1-22

**Existing types to keep** (lines 1-22):
```typescript
export type AlertType = 'dtc' | 'battery' | 'coolant' | 'connection';
export type AlertSeverity = 'info' | 'warning' | 'critical';
export type AlertStatus = 'active' | 'acknowledged' | 'dismissed' | 'resolved';

export interface Alert { /* keep as-is for ToastComponent compatibility */ }
export const ALERT_AUTO_DISMISS: Record<AlertSeverity, number> = { /* keep */ };
```

**New interface to add** (append to file):
```typescript
/** Row shape returned by Supabase from the alerts table (snake_case). */
export interface SupabaseAlert {
  id: number;                    // BIGSERIAL — integer, NOT UUID
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

### `packages/dashboard/src/app/core/services/alert.service.ts` (service, CRUD + event-driven)

**Analog:** `packages/dashboard/src/app/core/services/vehicle.service.ts` (full file)

**Imports pattern** (`vehicle.service.ts` lines 1-11):
```typescript
import { Injectable, computed, inject, signal, OnDestroy, resource, effect } from '@angular/core';
import { SupabaseService } from './supabase.service';
import { FleetService } from './fleet.service';
import type { RealtimeChannel } from '@supabase/supabase-js';
// alert.service.ts adds:
import { SupabaseAlert, Alert, AlertSeverity, AlertType, AlertStatus, ALERT_AUTO_DISMISS } from '../models/alert.model';
```

**resource() loader pattern** (`vehicle.service.ts` lines 48-70):
```typescript
readonly alertResource = resource({
  params: () => {
    const fleetIds = this.fleetService.fleets().map(f => f.id);
    return fleetIds.length > 0 ? { fleetIds } : undefined;
  },
  loader: async ({ params: { fleetIds } }: { params: { fleetIds: string[] } }) => {
    const since = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString();
    const { data, error } = await this.supabase.client
      .from('alerts')
      .select('*', { count: 'exact' })
      .in('fleet_id', fleetIds)
      .gte('created_at', since)
      .order('acknowledged', { ascending: true })
      .order('created_at', { ascending: false })
      .range(0, 199);
    if (error) throw error;
    this._dbAlerts.set((data ?? []) as SupabaseAlert[]);
    return data;
  },
  defaultValue: [],
});
```

**Realtime subscription pattern** (`telemetry.service.ts` lines 33-57 — exact channel pattern):
```typescript
// In AlertService constructor, start after alertResource resolves (mirror VehicleService effect):
private alertChannel: RealtimeChannel | null = null;

private subscribeToAlerts(fleetIds: string[]): void {
  if (this.alertChannel) return;
  this.alertChannel = this.supabase.client
    .channel('alerts-realtime')
    .on(
      'postgres_changes',
      { event: 'INSERT', schema: 'public', table: 'alerts' },
      (payload) => {
        const alert = payload.new as SupabaseAlert;
        if (fleetIds.includes(alert.fleet_id)) {
          this._dbAlerts.update(alerts => [alert, ...alerts]);
          // Also push to in-memory _alerts for ToastComponent compatibility:
          this.pushAlertFromDb(alert);
        }
      },
    )
    .subscribe();
}
```

**effect() bridge to start subscription** (`vehicle.service.ts` lines 143-152):
```typescript
constructor() {
  effect(() => {
    if (this.alertResource.status() === 'resolved' && !this.alertSubscribed) {
      this.alertSubscribed = true;
      const fleetIds = this.fleetService.fleets().map(f => f.id);
      this.subscribeToAlerts(fleetIds);
    }
  });
}
```

**Computed signals pattern** (`alert.service.ts` lines 26-48 — keep existing, add new):
```typescript
// Keep existing computed slices for ToastComponent + HeaderComponent compatibility:
readonly activeAlerts = computed(() => this._alerts().filter(a => a.status === 'active'));
readonly activeAlertCount = computed(() => this.activeAlerts().length);
readonly criticalAlertCount = computed(() => this.criticalAlerts().length);

// New: DB-backed signals
private readonly _dbAlerts = signal<SupabaseAlert[]>([]);
readonly dbAlerts = this._dbAlerts.asReadonly();

readonly unacknowledgedCount = computed(() =>
  this._dbAlerts().filter(a => !a.acknowledged).length
);

readonly activeDbAlerts = computed(() =>
  this._dbAlerts().filter(a => !a.acknowledged)
);
```

**acknowledgeAlert() — direct UPDATE** (replaces in-memory flip, mirrors `vehicle.service.ts` updateVehicle pattern lines 254-269):
```typescript
async acknowledgeAlert(alertId: number): Promise<void> {
  const { error } = await this.supabase.client
    .from('alerts')
    .update({ acknowledged: true, acknowledged_at: new Date().toISOString() })
    .eq('id', alertId);
  if (error) throw error;
  // Optimistic update on _dbAlerts signal:
  this._dbAlerts.update(alerts =>
    alerts.map(a => a.id === alertId ? { ...a, acknowledged: true } : a)
  );
}
```

**ngOnDestroy pattern** (`telemetry.service.ts` lines 117-127):
```typescript
ngOnDestroy(): void {
  this.destroy$.next();
  this.destroy$.complete();
  if (this.alertChannel) {
    this.supabase.client.removeChannel(this.alertChannel);
  }
}
```

**VehicleService call site to remove** (`vehicle.service.ts` line 311):
```typescript
// DELETE this line from VehicleService.processBatch():
this.alertService.checkAndCreateAlerts(record, vehicle ? getVehicleDisplayName(vehicle) : undefined);
```

**vehiclesWithHealth alertCount fix** (`vehicle.service.ts` line 109):
```typescript
// Change from (uses in-memory camelCase vehicleId):
alertCount: this.alertService.activeAlerts().filter(a => a.vehicleId === v.id).length,
// To (uses DB snake_case vehicle_id):
alertCount: this.alertService.activeDbAlerts().filter(a => a.vehicle_id === v.id).length,
```

---

### `packages/dashboard/src/app/features/alerts/alerts.component.ts` (component, CRUD)

**Analog:** existing `alerts.component.ts` (full rewrite) + `vehicle.service.ts` resource/pagination pattern

**Imports pattern** (existing `alerts.component.ts` lines 1-14, extend):
```typescript
import {
  ChangeDetectionStrategy, Component, computed, inject, signal,
} from '@angular/core';
import { DatePipe } from '@angular/common';
import { RouterLink } from '@angular/router';
import { MatButtonModule } from '@angular/material/button';
import { MatIconModule } from '@angular/material/icon';
import { MatChipsModule } from '@angular/material/chips';
import { MatPaginatorModule, PageEvent } from '@angular/material/paginator';
import { MatBadgeModule } from '@angular/material/badge';
import { AlertService } from '../../core/services/alert.service';
import { SupabaseAlert } from '../../core/models/alert.model';
```

**Component decorator pattern** (existing `alerts.component.ts` lines 18-29):
```typescript
@Component({
  selector: 'app-alerts',
  templateUrl: './alerts.component.html',
  styleUrl: './alerts.component.css',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [DatePipe, RouterLink, MatButtonModule, MatIconModule,
            MatChipsModule, MatPaginatorModule, MatBadgeModule],
})
```

**Pagination state + paged slice** (offset pagination per RESEARCH.md Pattern 5):
```typescript
export class AlertsComponent {
  private readonly alertService = inject(AlertService);

  readonly PAGE_SIZE = 25;
  readonly currentPage = signal(0);

  readonly allAlerts = this.alertService.dbAlerts;

  readonly totalCount = computed(() => this.allAlerts().length);

  readonly pagedAlerts = computed(() => {
    const from = this.currentPage() * this.PAGE_SIZE;
    return this.allAlerts().slice(from, from + this.PAGE_SIZE);
  });

  onPageChange(event: PageEvent): void {
    this.currentPage.set(event.pageIndex);
  }

  async acknowledge(alert: SupabaseAlert): Promise<void> {
    await this.alertService.acknowledgeAlert(alert.id);
  }

  getSeverityIcon(severity: string): string {
    // keep existing implementation from alerts.component.ts lines 66-72
  }
}
```

---

### `packages/dashboard/src/app/features/alerts/alerts.component.html` + `.css` (template/styles, rewrite)

**Analog:** existing `alerts.component.html` + Angular Material mat-list/mat-paginator pattern

**Template structure to implement:**
```html
<!-- mat-list rows, one per pagedAlerts() item -->
<!-- Each row: severity icon | vehicle name | type | message | timestamp | acknowledged badge | Acknowledge button -->
<!-- acknowledged rows: add CSS class 'alert-row--acknowledged' for dimmed opacity -->
<!-- mat-paginator at bottom bound to totalCount() and PAGE_SIZE -->
```

**CSS acknowledged dimming pattern** (new rule for `alerts.component.css`):
```css
.alert-row--acknowledged {
  opacity: 0.45;
}
```

---

### `packages/dashboard/src/app/layout/header/header.component.ts` (component, extend)

**Analog:** self — existing file lines 1-47

**Existing injection pattern** (lines 1-47 — keep all, add one signal):
```typescript
// Existing (keep):
readonly criticalAlertCount = this.alertService.criticalAlertCount;
readonly activeAlertCount = this.alertService.activeAlertCount;

// Add:
readonly unacknowledgedCount = this.alertService.unacknowledgedCount;
```

**MatBadgeModule already imported** (line 11) — no import changes needed.

**Bell icon template binding** (new in header.component.html):
```html
<!-- bind [matBadge] to unacknowledgedCount(); hide badge when 0 -->
<button mat-icon-button routerLink="/alerts"
        [matBadge]="unacknowledgedCount()"
        [matBadgeHidden]="unacknowledgedCount() === 0"
        matBadgeColor="warn">
  <mat-icon>notifications</mat-icon>
</button>
```

---

### `packages/dashboard/src/app/features/fleet-map/fleet-map.component.ts` (component, extend)

**Analog:** self — existing file lines 1-240

**Existing alertService injection** (line 47 — already injected):
```typescript
private readonly alertService = inject(AlertService);
```

**getMarkerIcon() extension point** (lines 174-201):
```typescript
// Current single alert color (line 179):
alert: '#eab308',

// Replace getMarkerIcon() to use severity-aware color:
private getMarkerColor(vehicle: VehicleWithHealth): string {
  const state = this.getVehicleState(vehicle);
  if (state !== 'alert') return state === 'running' ? '#84cc16' : '#5a4530';
  const vehicleAlerts = this.alertService.activeDbAlerts()
    .filter(a => a.vehicle_id === vehicle.id);
  const hasCritical = vehicleAlerts.some(a => a.severity === 'critical');
  return hasCritical ? '#ef4444' : '#eab308'; // red : yellow
}
```

**pulse CSS class already applied** (line 184):
```typescript
// Existing — pulse already fires when state === 'alert'. No change needed.
const pulse = (state === 'running' || state === 'alert') ? 'pulse' : '';
```

**getVehicleState() — no change needed** (lines 203-210):
```typescript
// alertCount check at line 207 drives 'alert' state.
// After VehicleService fix (alertCount from activeDbAlerts), this works automatically.
if (vehicle.alertCount > 0) return 'alert';
```

---

## Shared Patterns

### Supabase Realtime Channel Pattern
**Source:** `packages/dashboard/src/app/core/services/telemetry.service.ts` lines 33-57
**Apply to:** `alert.service.ts` subscribeToAlerts()
```typescript
this.fleetChannel = this.supabase.client
  .channel('fleet-telemetry')          // unique channel name
  .on(
    'postgres_changes',
    { event: 'INSERT', schema: 'public', table: 'telemetry_logs' },
    (payload) => { /* handle payload.new */ },
  )
  .subscribe((status) => {
    if (status === 'SUBSCRIBED') { /* set connected */ }
    else if (status === 'CHANNEL_ERROR' || status === 'TIMED_OUT') { /* set disconnected */ }
  });
```

### resource() + params guard Pattern
**Source:** `packages/dashboard/src/app/core/services/vehicle.service.ts` lines 48-70
**Apply to:** `alert.service.ts` alertResource
```typescript
params: () => {
  const ids = this.someService.items();
  return ids.length > 0 ? { ids } : undefined;  // undefined → loader skips → status = 'idle'
},
```

### RLS auth.jwt()->>'sub' Pattern
**Source:** `packages/supabase/migrations/010_fix_rls_policies.sql` lines 27, 44
**Apply to:** All new RLS policies in `014_alert_trigger_and_rls.sql`
```sql
user_id = (auth.jwt()->>'sub')
-- NOT auth.uid() — returns NULL for Auth0 users
```

### Optimistic Signal Update Pattern
**Source:** `packages/dashboard/src/app/core/services/vehicle.service.ts` lines 300-314
**Apply to:** `alert.service.ts` acknowledgeAlert()
```typescript
this.telemetryMap.update((map) => {
  const newMap = new Map(map);
  // ... mutate copy, return new reference
  return newMap;
});
// Alert equivalent:
this._dbAlerts.update(alerts =>
  alerts.map(a => a.id === id ? { ...a, acknowledged: true } : a)
);
```

### effect() → Subscribe Bridge Pattern
**Source:** `packages/dashboard/src/app/core/services/vehicle.service.ts` lines 143-152
**Apply to:** `alert.service.ts` constructor
```typescript
effect(() => {
  if (this.vehicleResource.status() === 'resolved' && !this.fleetSubscribed) {
    this.fleetSubscribed = true;
    this.telemetryService.subscribeToFleet();
    // ...
  }
});
```

### Angular Material Badge Pattern
**Source:** `packages/dashboard/src/app/layout/header/header.component.ts` line 11 (MatBadgeModule imported)
**Apply to:** header bell icon, vehicle card alert count
```html
[matBadge]="count()"
[matBadgeHidden]="count() === 0"
matBadgeColor="warn"
```

---

## No Analog Found

No files in this phase lack an analog. All patterns have concrete codebase matches.

---

## Critical Implementation Notes (Anti-Patterns to Avoid)

| Risk | File | What to Do |
|------|------|------------|
| Double-alerting | `vehicle.service.ts` line 311 | Remove `alertService.checkAndCreateAlerts()` call from `processBatch()` |
| Wrong id type | `alert.service.ts` acknowledgeAlert | Use `number` for alert id, NOT string — `alerts.id` is BIGSERIAL |
| Old trigger still runs | `014_alert_trigger_and_rls.sql` | First line after BEGIN: `DROP TRIGGER IF EXISTS trigger_generate_alert ON telemetry_logs;` |
| RLS blocks all reads | `014_alert_trigger_and_rls.sql` | Add SELECT policy using `auth.jwt()->>'sub'` — NOT `auth.uid()` |
| Realtime silent fail | `014_alert_trigger_and_rls.sql` | Include `ALTER PUBLICATION supabase_realtime ADD TABLE alerts;` |
| alertCount camelCase mismatch | `vehicle.service.ts` line 109 | Change filter from `a.vehicleId` to `a.vehicle_id` when switching to `activeDbAlerts()` |

---

## Metadata

**Analog search scope:** `packages/dashboard/src/app/`, `packages/supabase/migrations/`
**Files read:** 9 source files
**Pattern extraction date:** 2026-05-15
