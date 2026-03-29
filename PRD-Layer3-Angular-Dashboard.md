# VitalsDrive — Layer 3: Angular Dashboard
**Product Requirements Document**

**Version:** 1.0  
**Status:** Draft  
**Parent Document:** VitalsDrive_PRD.md (Main PRD)  
**Last Updated:** March 2026  

---

## 1. Executive Summary

### 1.1 Purpose

This document defines the requirements, architecture, and implementation specifications for **Layer 3 — Angular Dashboard**, the front-end visualization layer of the VitalsDrive fleet health monitoring platform.

### 1.2 Layer Overview

| Attribute | Detail |
|---|---|
| **Product** | VitalsDrive — Fleet Health Monitoring Platform |
| **Layer** | Layer 3: Frontend Dashboard |
| **Framework** | Angular 19/20 |
| **Hosting** | Vercel (Hobby Tier → Pro at scale) |
| **State Management** | Angular Signals + RxJS |
| **Primary Data Source** | Supabase Realtime (WebSocket) |
| **UI Components** | Angular Material + Leaflet.js |

### 1.3 Core Responsibility

The Angular Dashboard receives decoded telemetry data from Supabase via WebSocket and presents real-time vehicle health information to fleet operators through four primary alert systems: DTC alerts, battery health monitoring, coolant temperature warnings, and a live fleet map.

### 1.4 Inherits From Main PRD

- **Problem Statement:** Small fleet operators lack affordable, diagnostics-forward monitoring
- **Target User:** Non-technical fleet owners (2–15 vehicles) needing plain-English alerts
- **MVP Scope:** "The 3 Alerts" + Live Fleet Map
- **Architecture Context:** Data flows from OBD2 device → Node.js Parser → Supabase → Angular Dashboard

---

## 2. Feature Breakdown

### 2.1 MVP Feature Set

The dashboard implements four core features, collectively termed "The 3 Alerts" plus a supplementary map view:

| # | Feature | Priority | Description |
|---|---|---|---|
| 1 | **DTC Alert** | P0 | Real-time Check Engine code detection with plain-English translation |
| 2 | **Battery Health Monitor** | P0 | Voltage tracking with no-start prediction alerts |
| 3 | **Coolant Temperature Alert** | P0 | Hot engine warning when temp > 105°C |
| 4 | **Live Fleet Map** | P1 | Real-time vehicle positions on Leaflet.js map |

### 2.2 Feature Detail

#### Feature 1: DTC Alert (P0)

**Description:** Detect and notify when a vehicle's OBD2 system reports a Diagnostic Trouble Code.

**Behavior:**
- Subscribe to `dtc_codes` array in incoming telemetry records
- On new DTC detection, display toast notification with code + plain-English explanation
- DTC codes persist in UI until manually cleared or vehicle reports DTC cleared
- Support multiple simultaneous DTCs per vehicle

**User Value:** Prevents minor engine issues from becoming major repairs by prompting immediate attention.

#### Feature 2: Battery Health Monitor (P0)

**Description:** Track battery voltage in real-time and alert when voltage trends indicate imminent failure.

**Behavior:**
- Display current voltage per vehicle with trend line (last 30 readings)
- Alert threshold: Voltage < 12.4V sustained for 3 consecutive readings
- Predictive alert: If voltage drop rate suggests < 12.0V within 2 hours, show "May not start tonight" warning
- Historical view: Last 24 hours of voltage data

**User Value:** Prevents most common roadside failure (dead battery).

#### Feature 3: Coolant Temperature Alert (P0)

**Description:** Monitor engine coolant temperature and warn before heat damage occurs.

**Behavior:**
- Display temperature gauge per vehicle (0–130°C range)
- Warning state: Temp > 100°C (amber indicator)
- Critical state: Temp > 105°C (red indicator, alert notification)
- Gauge animates smoothly between values

**User Value:** A $15 thermostat replacement vs. $4,000 engine replacement.

#### Feature 4: Live Fleet Map (P1)

**Description:** Display real-time positions of all vehicles on an interactive map.

**Behavior:**
- Leaflet.js map centered on fleet's geographic region
- Vehicle markers with status indicators:
  - Engine off: Static gray marker
  - Engine running: Pulsing green marker
  - Alert active: Pulsing red marker with alert icon
- Click marker to view vehicle detail panel
- Auto-fit bounds to show all vehicles

**User Value:** Confirms devices are online and provides dispatch context.

---

## 3. Component Architecture

### 3.1 Component Hierarchy

```
AppComponent (root)
├── ShellComponent (layout wrapper)
│   ├── HeaderComponent (logo, nav, user menu)
│   ├── SidebarComponent (fleet list navigation)
│   └── MainContentComponent (router outlet)
│
├── DashboardComponent (route: /dashboard)
│   ├── VehicleGridComponent (responsive vehicle cards)
│   │   └── VehicleCardComponent (per-vehicle summary)
│   │       ├── HealthGaugeComponent (coolant temp)
│   │       ├── BatteryStatusComponent (voltage + trend)
│   │       └── DtcIndicatorComponent (alert badges)
│   │
│   └── AlertBannerComponent (top-level alert notifications)
│
├── FleetMapComponent (route: /map)
│   ├── LeafletMapComponent (core map rendering)
│   ├── VehicleMarkerComponent (per-vehicle markers)
│   └── VehicleDetailPanelComponent (selected vehicle info)
│
├── VehicleDetailComponent (route: /vehicle/:id)
│   ├── TelemetryChartComponent (historical data)
│   ├── DtcListComponent (active DTC codes with translations)
│   └── BatteryAnalysisComponent (voltage trend + predictions)
│
└── SharedComponents
    ├── ToastComponent (notifications)
    ├── LoadingSpinnerComponent
    └── ErrorBoundaryComponent
```

### 3.2 Component Specifications

#### HealthGaugeComponent

| Attribute | Specification |
|---|---|
| **Purpose** | Display coolant temperature as animated SVG gauge |
| **Range** | 0°C to 130°C |
| **Warning Threshold** | > 100°C (amber) |
| **Critical Threshold** | > 105°C (red) |
| **Animation** | CSS transitions, 300ms ease-out |
| **SVG Structure** | Arc path with gradient fill based on value |
| **States** | Normal (blue/green), Warning (amber), Critical (red pulsing) |

#### BatteryStatusComponent

| Attribute | Specification |
|---|---|
| **Purpose** | Display voltage with trend line and prediction |
| **Voltage Display** | Current value (large) + unit (V) |
| **Trend Line** | Last 30 telemetry readings, canvas-rendered |
| **Warning Threshold** | < 12.4V sustained |
| **Critical Threshold** | < 12.0V or predictive failure |
| **Prediction Logic** | Linear regression on voltage trend |
| **Prediction Message** | "May not start tonight" when predicted < 12.0V within 2 hours |

#### DtcAlertComponent

| Attribute | Specification |
|---|---|
| **Purpose** | Display DTC notifications with plain-English translations |
| **Notification Types** | Toast (single DTC), Banner (multiple/new) |
| **DTC Display** | Code (e.g., P0301) + Description |
| **Translation Source** | Local DTC database (embedded JSON) |
| **Persistence** | Until acknowledged or cleared |
| **Animation** | Slide-in from top-right, auto-dismiss after 8s (toast only) |

#### FleetMapComponent

| Attribute | Specification |
|---|---|
| **Purpose** | Display real-time vehicle positions |
| **Map Library** | Leaflet.js with OpenStreetMap tiles |
| **Marker Types** | Custom SVG icons per vehicle state |
| **Pulsing Effect** | CSS animation, 1.5s cycle, when engine running |
| **Clustering** | MarkerCluster plugin when > 20 vehicles |
| **Auto-refresh** | Update marker positions on each telemetry message |
| **Detail Panel** | Slide-in from right on marker click |

#### TelemetryService

| Attribute | Specification |
|---|---|
| **Purpose** | Subscribe to Supabase Realtime and distribute telemetry |
| **Connection** | Supabase JS SDK WebSocket |
| **Subscription** | `telemetry_logs` table INSERT events |
| **Distribution** | RxJS Observable per vehicle_id |
| **Buffering** | `bufferTime(500ms)` to batch rapid updates |
| **Reconnection** | Auto-reconnect with exponential backoff |

#### AlertService

| Attribute | Specification |
|---|---|
| **Purpose** | Manage alert state and notifications |
| **Alert Types** | DTC, Battery, Coolant |
| **Alert Priority** | Critical > Warning > Info |
| **Persistence** | Store active alerts in Signal |
| **Callbacks** | `onAlertCreate`, `onAlertDismiss` |

#### VehicleService

| Attribute | Specification |
|---|---|
| **Purpose** | Maintain vehicle registry and derived state |
| **Vehicle List** | Fetch from Supabase `vehicles` table |
| **Telemetry Binding** | Subscribe vehicles to TelemetryService |
| **Derived State** | Computed signals for health scores, alert counts |
| **Cache** | In-memory, cleared on logout |

---

## 4. State Management Design

### 4.1 Signal Architecture

```
Signal Types:
├── vehicleSignal: Signal<Vehicle[]>           (core state)
├── telemetrySignal: Signal<Map<string, TelemetryRecord>>  (real-time data)
├── alertsSignal: Signal<Alert[]>              (active alerts)
├── selectedVehicleSignal: Signal<string | null> (UI selection)
└── connectionStatusSignal: Signal<'connected' | 'disconnected' | 'reconnecting'>
```

### 4.2 Computed Signals

```typescript
// Derived health score per vehicle
const vehicleHealthScore = computed(() => {
  const telemetry = telemetrySignal();
  const voltage = telemetry().get(vehicleId)?.voltage ?? 14.0;
  const temp = telemetry().get(vehicleId)?.temp ?? 90;
  
  let score = 100;
  if (voltage < 12.4) score -= 20;
  if (voltage < 12.0) score -= 30;
  if (temp > 100) score -= 10;
  if (temp > 105) score -= 40;
  
  return Math.max(0, score);
});

// Active alert count
const activeAlertCount = computed(() => 
  alertsSignal().filter(a => a.status === 'active').length
);
```

### 4.3 RxJS Interop

```typescript
// Convert Supabase Realtime channel to Signal
export function toSignal<T>(observable: Observable<T>, initialValue: T): Signal<T> {
  const signal = signal(initialValue);
  observable.subscribe(value => signal.set(value));
  return signal;
}

// bufferTime for high-frequency telemetry
telemetryChannel
  .pipe(bufferTime(500))
  .subscribe(buffer => {
    if (buffer.length > 0) {
      processBatch(buffer);
    }
  });
```

---

## 5. Service Layer Design

### 5.1 TelemetryService

```typescript
@Injectable({ providedIn: 'root' })
export class TelemetryService {
  private supabase = inject(SupabaseClient);
  private channel$: ReplaySubject<TelemetryRecord> = new ReplaySubject(100);
  
  readonly telemetry$ = this.channel$.asObservable();
  readonly telemetryBuffer$ = this.telemetry$.pipe(
    bufferTime(500),
    filter(batch => batch.length > 0)
  );
  
  subscribeToVehicle(vehicleId: string): Observable<TelemetryRecord> {
    return this.supabase
      .channel(`vehicle:${vehicleId}`)
      .on(
        'postgres_changes',
        { event: 'INSERT', schema: 'public', table: 'telemetry_logs', filter: `vehicle_id=eq.${vehicleId}` },
        (payload) => payload.new
      )
      .subscribe()
      .pipe(map(record => record as TelemetryRecord));
  }
  
  subscribeToAllVehicles(): Observable<TelemetryRecord[]> {
    return this.supabase
      .channel('fleet-telemetry')
      .on(
        'postgres_changes',
        { event: 'INSERT', schema: 'public', table: 'telemetry_logs' },
        (payload) => payload.new
      )
      .subscribe()
      .pipe(
        bufferTime(500),
        filter(batch => batch.length > 0),
        map(batch => batch as TelemetryRecord[])
      );
  }
}
```

### 5.2 AlertService

```typescript
@Injectable({ providedIn: 'root' })
export class AlertService {
  private alerts = signal<Alert[]>([]);
  readonly activeAlerts = computed(() => 
    this.alerts().filter(a => a.status === 'active')
  );
  
  readonly dtcAlerts = computed(() =>
    this.activeAlerts().filter(a => a.type === 'dtc')
  );
  
  readonly batteryAlerts = computed(() =>
    this.activeAlerts().filter(a => a.type === 'battery')
  );
  
  readonly coolantAlerts = computed(() =>
    this.activeAlerts().filter(a => a.type === 'coolant')
  );
  
  checkAndCreateAlerts(telemetry: TelemetryRecord): void {
    this.checkBatteryAlert(telemetry);
    this.checkCoolantAlert(telemetry);
    this.checkDtcAlert(telemetry);
  }
  
  private checkCoolantAlert(telemetry: TelemetryRecord): void {
    if (telemetry.temp > 105) {
      this.createAlert({
        id: generateId(),
        vehicleId: telemetry.vehicle_id,
        type: 'coolant',
        severity: 'critical',
        message: `Engine overheating: ${telemetry.temp}°C`,
        timestamp: new Date()
      });
    } else if (telemetry.temp > 100) {
      this.createAlert({
        id: generateId(),
        vehicleId: telemetry.vehicle_id,
        type: 'coolant',
        severity: 'warning',
        message: `Engine running hot: ${telemetry.temp}°C`,
        timestamp: new Date()
      });
    }
  }
  
  private checkBatteryAlert(telemetry: TelemetryRecord): void {
    const recentReadings = this.getVoltageHistory(telemetry.vehicle_id);
    if (recentReadings.length < 3) return;
    
    const latest = recentReadings[recentReadings.length - 1];
    const avg = recentReadings.reduce((a, b) => a + b, 0) / recentReadings.length;
    
    if (latest < 12.0) {
      this.createAlert({
        id: generateId(),
        vehicleId: telemetry.vehicle_id,
        type: 'battery',
        severity: 'critical',
        message: 'Battery critical: Vehicle may not start',
        timestamp: new Date()
      });
    } else if (latest < 12.4 && avg < 12.4) {
      this.createAlert({
        id: generateId(),
        vehicleId: telemetry.vehicle_id,
        type: 'battery',
        severity: 'warning',
        message: 'Battery voltage low',
        timestamp: new Date()
      });
    }
  }
  
  private checkDtcAlert(telemetry: TelemetryRecord): void {
    if (telemetry.dtc_codes && telemetry.dtc_codes.length > 0) {
      telemetry.dtc_codes.forEach(code => {
        const existing = this.alerts().find(
          a => a.type === 'dtc' && a.vehicleId === telemetry.vehicle_id && a.code === code
        );
        if (!existing) {
          this.createAlert({
            id: generateId(),
            vehicleId: telemetry.vehicle_id,
            type: 'dtc',
            severity: 'warning',
            code,
            message: `Check Engine: ${code} - ${DtcDatabase.translate(code)}`,
            timestamp: new Date()
          });
        }
      });
    }
  }
  
  dismissAlert(alertId: string): void {
    this.alerts.update(alerts => 
      alerts.map(a => a.id === alertId ? { ...a, status: 'dismissed' } : a)
    );
  }
}
```

### 5.3 VehicleService

```typescript
@Injectable({ providedIn: 'root' })
export class VehicleService {
  private supabase = inject(SupabaseClient);
  private telemetryService = inject(TelemetryService);
  
  readonly vehicles = signal<Vehicle[]>([]);
  readonly selectedVehicle = signal<string | null>(null);
  readonly telemetryMap = signal<Map<string, TelemetryRecord[]>>(
    new Map()
  );
  
  readonly vehiclesWithHealth = computed(() => {
    const map = this.telemetryMap();
    return this.vehicles().map(v => ({
      ...v,
      latestTelemetry: map.get(v.id)?.[0],
      healthScore: this.calculateHealthScore(map.get(v.id)?.[0])
    }));
  });
  
  async loadVehicles(): Promise<void> {
    const { data } = await this.supabase
      .from('vehicles')
      .select('*');
    if (data) {
      this.vehicles.set(data);
      data.forEach(v => this.subscribeToVehicle(v.id));
    }
  }
  
  private subscribeToVehicle(vehicleId: string): void {
    this.telemetryService.subscribeToVehicle(vehicleId)
      .subscribe(record => {
        this.telemetryMap.update(map => {
          const newMap = new Map(map);
          const existing = newMap.get(vehicleId) ?? [];
          newMap.set(vehicleId, [record, ...existing].slice(0, 100));
          return newMap;
        });
      });
  }
  
  private calculateHealthScore(telemetry?: TelemetryRecord): number {
    if (!telemetry) return 100;
    let score = 100;
    if (telemetry.voltage < 12.4) score -= 20;
    if (telemetry.voltage < 12.0) score -= 30;
    if (telemetry.temp > 100) score -= 10;
    if (telemetry.temp > 105) score -= 40;
    return Math.max(0, score);
  }
  
  selectVehicle(vehicleId: string | null): void {
    this.selectedVehicle.set(vehicleId);
  }
}
```

---

## 6. API Integration (Supabase Client Patterns)

### 6.1 Supabase Client Configuration

```typescript
// environment.ts
export const environment = {
  production: false,
  supabase: {
    url: 'https://[project].supabase.co',
    anonKey: '[anon-key]',
    websocketUrl: 'wss://[project].supabase.co/realtime/v1/websocket'
  }
};

// environment.prod.ts
export const environment = {
  production: true,
  supabase: {
    url: 'https://[project].supabase.co',
    anonKey: '[anon-key]',
    websocketUrl: 'wss://[project].supabase.co/realtime/v1/websocket'
  }
};
```

### 6.2 Supabase Module Setup

```typescript
@Injectable({ providedIn: 'root' })
export class SupabaseService {
  private client: SupabaseClient;
  
  constructor() {
    this.client = createClient(
      environment.supabase.url,
      environment.supabase.anonKey,
      {
        realtime: {
          params: { engine: 'supabase-js-v2' }
        }
      }
    );
  }
  
  get client(): SupabaseClient {
    return this.client;
  }
}
```

### 6.3 Realtime Subscription Patterns

```typescript
// Pattern 1: Subscribe to all telemetry (FleetMapComponent)
const channel = supabase.channel('fleet-telemetry')
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'telemetry_logs'
  }, handleNewTelemetry)
  .subscribe(status => {
    if (status === 'SUBSCRIBED') {
      this.connectionStatus.set('connected');
    }
    if (status === 'CHANNEL_ERROR') {
      this.connectionStatus.set('disconnected');
    }
  });

// Pattern 2: Subscribe to specific vehicle
const vehicleChannel = supabase.channel(`vehicle-${vehicleId}`)
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'telemetry_logs',
    filter: `vehicle_id=eq.${vehicleId}`
  }, handleVehicleTelemetry)
  .subscribe();

// Cleanup on destroy
ngOnDestroy(): void {
  supabase.removeChannel(channel);
}
```

### 6.4 Error Handling

```typescript
// Retry with exponential backoff
async function subscribeWithRetry(
  channel: RealtimeChannel,
  maxRetries = 5
): Promise<void> {
  let attempts = 0;
  while (attempts < maxRetries) {
    try {
      await channel.subscribe();
      return;
    } catch (error) {
      attempts++;
      const delay = Math.pow(2, attempts) * 1000;
      await new Promise(r => setTimeout(r, delay));
    }
  }
  throw new Error('Failed to subscribe after max retries');
}
```

---

## 7. Routing Structure

### 7.1 Route Configuration

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: '',
    component: ShellComponent,
    children: [
      {
        path: '',
        redirectTo: 'dashboard',
        pathMatch: 'full'
      },
      {
        path: 'dashboard',
        loadComponent: () => import('./pages/dashboard/dashboard.component')
          .then(m => m.DashboardComponent),
        title: 'Fleet Dashboard - VitalsDrive'
      },
      {
        path: 'map',
        loadComponent: () => import('./pages/fleet-map/fleet-map.component')
          .then(m => m.FleetMapComponent),
        title: 'Fleet Map - VitalsDrive'
      },
      {
        path: 'vehicle/:id',
        loadComponent: () => import('./pages/vehicle-detail/vehicle-detail.component')
          .then(m => m.VehicleDetailComponent),
        title: 'Vehicle Details - VitalsDrive'
      },
      {
        path: 'alerts',
        loadComponent: () => import('./pages/alerts/alerts.component')
          .then(m => m.AlertsComponent),
        title: 'Active Alerts - VitalsDrive'
      }
    ]
  },
  {
    path: 'login',
    loadComponent: () => import('./pages/login/login.component')
      .then(m => m.LoginComponent),
    title: 'Login - VitalsDrive'
  },
  {
    path: '**',
    redirectTo: 'dashboard'
  }
];
```

### 7.2 Route Guards

```typescript
// auth.guard.ts
@Injectable({ providedIn: 'root' })
export class authGuard {
  private authService = inject(AuthService);
  
  canActivate(): boolean {
    return this.authService.isAuthenticated();
  }
}
```

---

## 8. UI/UX Requirements

### 8.1 HealthGaugeComponent

| Requirement | Specification |
|---|---|
| **Visual Type** | SVG arc gauge |
| **Size** | 160px × 120px (card), 200px × 150px (detail view) |
| **Range** | 0°C to 130°C |
| **Normal Zone** | 0–100°C (fill: `#22c55e` green) |
| **Warning Zone** | 100–105°C (fill: `#f59e0b` amber) |
| **Critical Zone** | > 105°C (fill: `#ef4444` red, pulsing animation) |
| **Needle** | Animated SVG path, 300ms transition |
| **Value Display** | Large centered text (24px bold) |
| **Unit Label** | "°C" below value (14px) |
| **Background** | Semi-transparent dark (#1a1a2e at 90% opacity) |
| **Border Radius** | 12px |

### 8.2 BatteryStatusComponent

| Requirement | Specification |
|---|---|
| **Layout** | Vertical stack: voltage display + trend chart |
| **Voltage Display** | 32px bold, color-coded (green/amber/red) |
| **Trend Chart** | Canvas element, 100% width × 60px height |
| **Chart Type** | Line chart with gradient fill |
| **Chart Colors** | Stroke: `#3b82f6` blue, Fill: `#3b82f6` at 20% opacity |
| **Warning Color** | Amber (#f59e0b) when < 12.4V |
| **Critical Color** | Red (#ef4444) when < 12.0V |
| **Chart Points** | Last 30 readings, 2px dots on hover |
| **Prediction Line** | Dashed line extending 10 readings into future |
| **No-Start Alert** | Yellow banner with warning icon when predicted < 12.0V |

### 8.3 DtcAlertComponent

| Requirement | Specification |
|---|---|
| **Toast Width** | 360px |
| **Toast Position** | Top-right, 16px from edges |
| **Toast Background** | White with left border (4px, color by severity) |
| **Icon** | Engine check icon, 24px, left-aligned |
| **Code Display** | Bold, monospace (e.g., "P0301") |
| **Description** | Regular weight, below code |
| **Dismiss Button** | X icon, top-right, appears on hover |
| **Animation** | Slide in from right (300ms), fade out (200ms) |
| **Auto-dismiss** | 8 seconds for info, 12 seconds for warning, manual for critical |
| **Stacking** | Max 3 visible, "+N more" indicator |

### 8.4 FleetMapComponent

| Requirement | Specification |
|---|---|
| **Map Provider** | OpenStreetMap via Leaflet.js |
| **Initial View** | Auto-fit to show all vehicles with 10% padding |
| **Tile Layer** | `https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png` |
| **Attribution** | "© OpenStreetMap contributors" |
| **Marker Size** | 32px × 32px (SVG) |
| **Marker States** | Idle (gray), Running (green pulse), Alert (red pulse) |
| **Pulse Animation** | Scale 1.0→1.2→1.0, opacity 1.0→0.7→1.0, 1.5s cycle |
| **Popup** | Vehicle name + quick stats on click |
| **Detail Panel** | 400px wide, slides from right |
| **Mobile View** | Full-screen map, bottom sheet for details |

### 8.5 VehicleCardComponent

| Requirement | Specification |
|---|---|
| **Card Size** | 320px × 200px (grid), 100% width (list) |
| **Card Background** | Dark (#1a1a2e) |
| **Border** | 1px solid #2d2d44, 2px when alert (red/amber) |
| **Border Radius** | 16px |
| **Shadow** | 0 4px 12px rgba(0,0,0,0.3) |
| **Header** | Vehicle name + status badge |
| **Body** | HealthGaugeComponent + BatteryStatusComponent side-by-side |
| **Footer** | Last updated timestamp |
| **Hover State** | Border color lightens, subtle scale (1.02) |
| **Click Action** | Navigate to /vehicle/:id |

### 8.6 ShellComponent

| Requirement | Specification |
|---|---|
| **Layout** | Sidebar (240px) + Main content |
| **Sidebar Background** | Dark (#0f0f1a) |
| **Header Height** | 64px |
| **Content Padding** | 24px |
| **Mobile Breakpoint** | 768px (sidebar becomes drawer) |
| **Logo** | Top-left of sidebar, 120px wide |
| **Nav Items** | Icon + label, 48px height |
| **Active State** | Background highlight, left border accent |

---

## 9. Real-Time Data Handling

### 9.1 BufferTime Configuration

```typescript
// High-frequency telemetry batching
telemetry$ = this.channel$.pipe(
  bufferTime(500), // Batch updates every 500ms
  filter(batch => batch.length > 0),
  map(batch => this.mergeBatch(batch))
);

// mergeBatch consolidates multiple updates for same vehicle
private mergeBatch(batch: TelemetryRecord[]): Map<string, TelemetryRecord> {
  const result = new Map<string, TelemetryRecord>();
  batch.forEach(record => {
    const existing = result.get(record.vehicle_id);
    if (!existing || new Date(record.timestamp) > new Date(existing.timestamp)) {
      result.set(record.vehicle_id, record);
    }
  });
  return result;
}
```

### 9.2 Throttling Patterns

```typescript
// Throttle marker updates on map (prevent excessive re-renders)
markerUpdate$ = this.telemetry$.pipe(
  throttleTime(1000, asyncScheduler, { leading: true, trailing: true })
);

// Debounce vehicle selection to prevent rapid API calls
selectVehicle$ = this.selectedVehicle$.pipe(
  debounceTime(300),
  distinctUntilChanged()
);
```

### 9.3 Memory Management

```typescript
// Limit telemetry history per vehicle
const MAX_TELEMETRY_HISTORY = 100;

addTelemetry(record: TelemetryRecord): void {
  this.telemetryMap.update(map => {
    const vehicleHistory = map.get(record.vehicle_id) ?? [];
    const newHistory = [record, ...vehicleHistory].slice(0, MAX_TELEMETRY_HISTORY);
    const newMap = new Map(map);
    newMap.set(record.vehicle_id, newHistory);
    return newMap;
  });
}

// Cleanup on component destroy
ngOnDestroy(): void {
  this.subscriptions.forEach(s => s.unsubscribe());
  this.telemetryMap.set(new Map());
}
```

---

## 10. DTC Plain-English Translation

### 10.1 DTC Database Structure

```typescript
// dtc-database.ts
export interface DtcEntry {
  code: string;        // e.g., "P0301"
  category: string;    // e.g., "Powertrain"
  system: string;      // e.g., "Misfire"
  severity: 'low' | 'medium' | 'high' | 'critical';
  description: string; // Plain-English explanation
  causes: string[];   // Common causes
  symptoms: string[]; // What driver might notice
  recommendations: string; // What to do
}

export const DTC_DATABASE: Record<string, DtcEntry> = {
  'P0300': {
    code: 'P0300',
    category: 'Powertrain',
    system: 'Misfire',
    severity: 'medium',
    description: 'Random/multiple cylinder misfire detected',
    causes: [
      'Faulty spark plugs or ignition coils',
      'Low fuel pressure',
      'Vacuum leaks',
      'EGR valve issues'
    ],
    symptoms: ['Rough idle', 'Engine shaking', 'Increased fuel consumption'],
    recommendations: 'Check spark plugs first. If problem persists, have diagnostics scanned for specific cylinder.'
  },
  'P0301': {
    code: 'P0301',
    category: 'Powertrain',
    system: 'Misfire',
    severity: 'medium',
    description: 'Cylinder 1 misfire detected',
    causes: [
      'Faulty spark plug in cylinder 1',
      'Ignition coil failure',
      'Fuel injector issues',
      'Vacuum leak near cylinder 1'
    ],
    symptoms: ['Rough idle', 'Engine vibration', 'Check Engine light flashing'],
    recommendations: 'Inspect spark plug and ignition coil for cylinder 1. Replace if damaged.'
  },
  'P0420': {
    code: 'P0420',
    category: 'Powertrain',
    system: 'Catalyst System',
    severity: 'low',
    description: 'Catalyst system efficiency below threshold (Bank 1)',
    causes: [
      'Failing catalytic converter',
      'Oxygen sensor malfunction',
      'Exhaust leaks',
      'Rich fuel mixture'
    ],
    symptoms: ['Reduced fuel efficiency', 'Sulfur smell from exhaust'],
    recommendations: 'Vehicle is drivable but has reduced efficiency. Have inspected within 500 miles.'
  },
  // ... additional codes
};
```

### 10.2 DTC Translation Service

```typescript
@Injectable({ providedIn: 'root' })
export class DtcTranslationService {
  translate(code: string): DtcEntry | null {
    const normalized = code.toUpperCase().trim();
    return DTC_DATABASE[normalized] ?? this.unknownCodeHandler(normalized);
  }
  
  private unknownCodeHandler(code: string): DtcEntry {
    // Parse standard OBD-II structure
    const prefix = code.charAt(0);
    const system = code.charAt(1);
    const generic = code.substring(2);
    
    const prefixes: Record<string, string> = {
      'P': 'Powertrain',
      'B': 'Body',
      'C': 'Chassis',
      'U': 'Network'
    };
    
    const systems: Record<string, string> = {
      '0': 'Generic',
      '1': 'Fuel & Air Metering',
      '2': 'Fuel & Air Metering (Specific)',
      '3': 'Ignition/Misfire',
      '4': 'Emissions',
      '5': 'Speed/Idle Control',
      '6': 'Computer/Output Circuit',
      '7': 'Transmission'
    };
    
    return {
      code,
      category: prefixes[prefix] ?? 'Unknown',
      system: systems[system] ?? 'Unknown System',
      severity: 'medium',
      description: `Generic ${prefixes[prefix] ?? 'Unknown'} fault code`,
      causes: ['Diagnostic scan recommended'],
      symptoms: ['Check Engine light illuminated'],
      recommendations: 'Code requires professional diagnosis'
    };
  }
  
  getSeverityColor(severity: string): string {
    switch (severity) {
      case 'critical': return '#ef4444';
      case 'high': return '#f97316';
      case 'medium': return '#f59e0b';
      case 'low': return '#22c55e';
      default: return '#6b7280';
    }
  }
}
```

### 10.3 Initial DTC Coverage

The MVP DTC database includes translations for the 50 most common OBD-II codes affecting fleet vehicles:

| Category | Codes | Coverage |
|---|---|---|
| Misfire (P0300-P0309) | 10 | Primary + Cylinder-specific |
| Oxygen Sensor (P0130-P0169) | 8 | Bank 1 & 2 sensors |
| Catalyst (P0420-P0430) | 3 | Common failure codes |
| EVAP (P0440-P0469) | 6 | Vapor recovery system |
| Throttle/Pedal (P2120-P2138) | 5 | Electronic throttle |
| Transmission (P0700-P0739) | 8 | General transmission errors |
| General (P0100-P0699) | 10 | Various sensors and systems |

---

## 11. Alerting System Design

### 11.1 Alert Types and Severity

```typescript
enum AlertType {
  DTC = 'dtc',
  BATTERY = 'battery',
  COOLANT = 'coolant',
  CONNECTION = 'connection'
}

enum AlertSeverity {
  INFO = 'info',
  WARNING = 'warning',
  CRITICAL = 'critical'
}

interface Alert {
  id: string;
  vehicleId: string;
  vehicleName?: string;
  type: AlertType;
  severity: AlertSeverity;
  code?: string;          // For DTC alerts
  message: string;
  timestamp: Date;
  status: 'active' | 'acknowledged' | 'dismissed' | 'resolved';
  metadata?: Record<string, unknown>;
}
```

### 11.2 Alert Creation Rules

| Alert Type | Condition | Severity | Auto-Dismiss |
|---|---|---|---|
| DTC | New DTC code detected | Warning | No |
| DTC (Critical) | P0300, P0325, P0335 | Critical | No |
| Battery Low | Voltage < 12.4V sustained (3 readings) | Warning | 5 min after normal |
| Battery Critical | Voltage < 12.0V | Critical | No |
| Battery Predictive | Predicted < 12.0V within 2 hours | Warning | No |
| Coolant Warning | Temp > 100°C | Warning | 2 min after normal |
| Coolant Critical | Temp > 105°C | Critical | No |
| Connection Lost | No telemetry for > 2 minutes | Warning | Immediately on reconnect |

### 11.3 Alert Notification UX

```typescript
// Toast notification hierarchy
const notificationConfig = {
  [AlertSeverity.CRITICAL]: {
    duration: 0, // No auto-dismiss
    sound: true,
    vibration: true, // Mobile
    stacking: true,
    maxVisible: 5
  },
  [AlertSeverity.WARNING]: {
    duration: 12000,
    sound: false,
    vibration: false,
    stacking: true,
    maxVisible: 3
  },
  [AlertSeverity.INFO]: {
    duration: 8000,
    sound: false,
    vibration: false,
    stacking: false
  }
};
```

### 11.4 Alert Banner

- Fixed position below header
- Shows highest-priority active alert
- Expandable to show all active alerts
- Color-coded by severity (red/amber/blue)
- Click to navigate to relevant vehicle

---

## 12. Performance Optimization

### 12.1 Change Detection

```typescript
// Use OnPush change detection strategy globally
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})

// Or for high-frequency components
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  providers: [provideChangeDetection({ onPush: true })]
})
```

### 12.2 Signal-Based Reactivity

```typescript
// Prefer signals over observables for synchronous state
readonly vehicleHealth = computed(() => 
  this.telemetryMap().get(this.vehicleId)?.healthScore ?? 100
);

// Use toObservable() for async operations
readonly vehicle$ = toObservable(this.selectedVehicle).pipe(
  switchMap(id => id ? this.vehicleService.getVehicle(id) : of(null))
);
```

### 12.3 Lazy Loading

```typescript
// Route-level lazy loading
export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./pages/dashboard/dashboard.component')
      .then(m => m.DashboardComponent)
  },
  {
    path: 'map',
    loadComponent: () => import('./pages/fleet-map/fleet-map.component')
      .then(m => m.FleetMapComponent)
  }
];

// Feature module lazy loading
@Injectable({ providedIn: 'root' })
export class FleetMapFeatureModule {}
```

### 12.4 Virtual Scrolling

```typescript
// For vehicle lists with many items
@Component({
  template: `
    <cdk-virtual-scroll-viewport itemSize="200" class="vehicle-viewport">
      <app-vehicle-card
        *cdkVirtualFor="let vehicle of vehicles(); trackBy: trackByVehicleId"
        [vehicle]="vehicle"
      />
    </cdk-virtual-scroll-viewport>
  `
})
```

### 12.5 Image/Marker Optimization

```typescript
// Preload and cache marker icons
const markerIcons = {
  idle: L.icon({ iconUrl: '/assets/markers/idle.svg', iconSize: [32, 32] }),
  running: L.icon({ iconUrl: '/assets/markers/running.svg', iconSize: [32, 32] }),
  alert: L.icon({ iconUrl: '/assets/markers/alert.svg', iconSize: [32, 32] })
};

// Use CSS transforms for animations (GPU accelerated)
.marker-pulse {
  animation: pulse 1.5s ease-in-out infinite;
  transform: translate3d(0, 0, 0); // Force GPU
}
```

### 12.6 Bundle Optimization

```typescript
// angular.json optimization
{
  "projects": {
    "dashboard": {
      "architect": {
        "build": {
          "options": {
            "optimization": true,
            "sourceMap": false,
            "namedChunks": false,
            "extractLicenses": true,
            "budgets": [
              {
                "type": "initial",
                "maximumWarning": "500kb",
                "maximumError": "1mb"
              },
              {
                "type": "anyComponent",
                "maximumWarning": "50kb",
                "maximumError": "100kb"
              }
            ]
          }
        }
      }
    }
  }
}
```

---

## 13. Responsive Design

### 13.1 Breakpoints

| Breakpoint | Width | Layout |
|---|---|---|
| Mobile | < 576px | Single column, bottom nav, full-width cards |
| Tablet | 576–768px | 2-column grid, collapsible sidebar |
| Desktop | 768–1024px | Sidebar + 3-column grid |
| Large Desktop | > 1024px | Full sidebar + 4-column grid |

### 13.2 Grid System

```scss
// Vehicle grid responsive
.vehicle-grid {
  display: grid;
  grid-template-columns: repeat(1, 1fr);
  
  @media (min-width: 576px) {
    grid-template-columns: repeat(2, 1fr);
  }
  
  @media (min-width: 768px) {
    grid-template-columns: repeat(3, 1fr);
  }
  
  @media (min-width: 1024px) {
    grid-template-columns: repeat(4, 1fr);
  }
  
  gap: 16px;
  padding: 16px;
}
```

### 13.3 Mobile Navigation

```scss
// Bottom navigation bar (mobile only)
.bottom-nav {
  display: flex;
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  height: 64px;
  background: #1a1a2e;
  border-top: 1px solid #2d2d44;
  z-index: 1000;
  
  @media (min-width: 768px) {
    display: none;
  }
  
  .nav-item {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    color: #9ca3af;
    
    &.active {
      color: #3b82f6;
    }
  }
}
```

### 13.4 Touch Interactions

```typescript
// Gesture handling for mobile map
@Component({
  template: `
    <div
      class="map-container"
      (touchstart)="onTouchStart($event)"
      (touchmove)="onTouchMove($event)"
      (touchend)="onTouchEnd($event)"
    >
      <leaflet-map />
    </div>
  `
})
```

---

## 14. Accessibility Requirements

### 14.1 WCAG 2.1 AA Compliance

| Requirement | Implementation |
|---|---|
| Color Contrast | All text meets 4.5:1 ratio (body) or 3:1 (large text) |
| Focus Indicators | Visible focus ring on all interactive elements |
| Keyboard Navigation | Full app accessible via keyboard |
| Screen Reader | ARIA labels on all components |
| Motion | Respect `prefers-reduced-motion` |

### 14.2 ARIA Labels

```html
<!-- Health gauge -->
<div
  role="meter"
  [attr.aria-valuenow]="temperature"
  aria-valuemin="0"
  aria-valuemax="130"
  [attr.aria-valuetext]="temperature + ' degrees Celsius'"
>
  <svg>...</svg>
</div>

<!-- Vehicle card -->
<article
  class="vehicle-card"
  role="article"
  [attr.aria-label]="vehicle.name + ' status: ' + vehicle.status"
>
  <h3>{{ vehicle.name }}</h3>
  <span [class]="getHealthClass(vehicle.healthScore)">
    {{ vehicle.healthScore }}% healthy
  </span>
</article>

<!-- Alert toast -->
<div
  role="alert"
  aria-live="assertive"
  class="toast"
>
  <img src="/assets/icons/engine.svg" aria-hidden="true" />
  <span>{{ alert.message }}</span>
  <button (click)="dismiss()" aria-label="Dismiss alert">×</button>
</div>
```

### 14.3 Keyboard Navigation

```typescript
// Focus management
@Component({
  template: `
    <button
      #alertButton
      (keydown.enter)="openAlerts()"
      (keydown.escape)="closeAlerts()"
    >
      Alerts ({{ alertCount() }})
    </button>
  `
})

// Skip link
@Component({
  template: `
    <a href="#main-content" class="skip-link">
      Skip to main content
    </a>
    <main id="main-content">
      <router-outlet />
    </main>
  `
})
```

### 14.4 Reduced Motion

```scss
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
  
  .marker-pulse {
    animation: none;
  }
}
```

---

## 15. Testing Strategy

### 15.1 Unit Tests (Jasmine/Karma)

```typescript
// dtc-translation.service.spec.ts
describe('DtcTranslationService', () => {
  let service: DtcTranslationService;
  
  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [DtcTranslationService]
    });
    service = TestBed.inject(DtcTranslationService);
  });
  
  it('should translate known DTC code', () => {
    const result = service.translate('P0301');
    expect(result).toBeTruthy();
    expect(result!.code).toBe('P0301');
    expect(result!.severity).toBe('medium');
  });
  
  it('should handle unknown DTC codes gracefully', () => {
    const result = service.translate('P9999');
    expect(result).toBeTruthy();
    expect(result!.category).toBe('Powertrain');
  });
});

// alert.service.spec.ts
describe('AlertService', () => {
  let service: AlertService;
  
  it('should create critical alert when coolant > 105', () => {
    const telemetry: TelemetryRecord = {
      vehicle_id: 'test-1',
      temp: 108,
      voltage: 12.6,
      dtc_codes: []
    };
    
    service.checkAndCreateAlerts(telemetry);
    
    expect(service.coolantAlerts().length).toBe(1);
    expect(service.coolantAlerts()[0].severity).toBe('critical');
  });
});
```

### 15.2 Component Tests

```typescript
// health-gauge.component.spec.ts
describe('HealthGaugeComponent', () => {
  let component: HealthGaugeComponent;
  let fixture: ComponentFixture<HealthGaugeComponent>;
  
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [HealthGaugeComponent],
      schemas: [NO_ERRORS_SCHEMA]
    }).compileComponents();
    
    fixture = TestBed.createComponent(HealthGaugeComponent);
    component = fixture.componentInstance;
  });
  
  it('should render gauge with value', () => {
    component.value = 85;
    fixture.detectChanges();
    
    const compiled = fixture.nativeElement;
    expect(compiled.querySelector('.gauge-value').textContent).toContain('85');
  });
  
  it('should apply critical class when temp > 105', () => {
    component.value = 108;
    fixture.detectChanges();
    
    const compiled = fixture.nativeElement;
    expect(compiled.querySelector('.gauge-critical')).toBeTruthy();
  });
});
```

### 15.3 Integration Tests

```typescript
// telemetry-integration.spec.ts
describe('Telemetry Flow Integration', () => {
  let telemetryService: TelemetryService;
  let alertService: AlertService;
  
  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        TelemetryService,
        AlertService,
        { provide: SupabaseService, useClass: MockSupabaseService }
      ]
    });
    
    telemetryService = TestBed.inject(TelemetryService);
    alertService = TestBed.inject(AlertService);
  });
  
  it('should create alert when telemetry triggers threshold', fakeAsync(() => {
    const testRecord: TelemetryRecord = {
      vehicle_id: 'test-1',
      temp: 108,
      voltage: 11.8,
      dtc_codes: ['P0301']
    };
    
    // Simulate telemetry arriving
    (telemetryService as any).channel$.next(testRecord);
    tick(600); // Wait for bufferTime
    
    expect(alertService.activeAlerts().length).toBeGreaterThan(0);
  }));
});
```

### 15.4 E2E Tests (Playwright)

```typescript
// dashboard.e2e.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Dashboard', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/dashboard');
  });
  
  test('should display vehicle cards', async ({ page }) => {
    await page.waitForSelector('.vehicle-card', { timeout: 5000 });
    const cards = await page.locator('.vehicle-card').count();
    expect(cards).toBeGreaterThan(0);
  });
  
  test('should navigate to vehicle detail on click', async ({ page }) => {
    await page.click('.vehicle-card:first-child');
    await expect(page).toHaveURL(/\/vehicle\/.+/);
  });
  
  test('should show alert toast for DTC', async ({ page }) => {
    // Simulate DTC through test API or UI action
    await page.evaluate(() => {
      window.postMessage({ type: 'TEST_DTC', code: 'P0301' }, '*');
    });
    
    await page.waitForSelector('.toast', { timeout: 3000 });
    await expect(page.locator('.toast')).toContainText('P0301');
  });
  
  test('should be accessible', async ({ page }) => {
    await expect(page).toHaveTitle(/VitalsDrive/);
    await expect(page.locator('main')).toBeVisible();
    await expect(page.locator('.skip-link')).toBeAttached();
  });
});
```

### 15.5 Test Coverage Targets

| Layer | Target | Critical Components |
|---|---|---|
| Services | 90% | AlertService, TelemetryService, DtcTranslationService |
| Components | 80% | HealthGauge, BatteryStatus, DtcAlert, FleetMap |
| Integration | 70% | Realtime flow, Alert creation |
| E2E | Critical paths | Dashboard load, Alert display, Vehicle selection |

---

## 16. Vercel Deployment Configuration

### 16.1 Project Structure

```
dashboard/
├── src/
│   ├── app/
│   │   ├── app.component.ts
│   │   ├── app.config.ts
│   │   ├── app.routes.ts
│   │   ├── core/                    # Singleton services
│   │   ├── features/               # Feature modules
│   │   ├── shared/                 # Shared components
│   │   └── pages/                  # Route pages
│   ├── environments/
│   └── main.ts
├── angular.json
├── package.json
├── tsconfig.json
├── vercel.json                     # Vercel config
└── .vercelignore
```

### 16.2 vercel.json

```json
{
  "framework": "angular",
  "buildCommand": "ng build",
  "outputDirectory": "dist/vitalsdrive-dashboard/browser",
  "devCommand": "ng serve",
  "installCommand": "npm ci",
  "regions": ["iad1"],
  "functions": {
    "api/**/*.ts": {
      "runtime": "nodejs20.x"
    }
  },
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-XSS-Protection",
          "value": "1; mode=block"
        }
      ]
    },
    {
      "source": "/realtime/(.*)",
      "headers": [
        {
          "key": "Upgrade",
          "value": "websocket"
        },
        {
          "key": "Connection",
          "value": "upgrade"
        }
      ]
    }
  ],
  "rewrites": [
    {
      "source": "/api/(.*)",
      "destination": "/api/index.html"
    }
  ]
}
```

### 16.3 Environment Variables (Vercel Dashboard)

| Variable | Value | Description |
|---|---|---|
| `SUPABASE_URL` | `https://[project].supabase.co` | Supabase project URL |
| `SUPABASE_ANON_KEY` | `[anon-key]` | Supabase anonymous key |
| `NG_APP_VERSION` | `{{VERSION}}` | Auto-populated build version |

### 16.4 Deployment Pipeline

```yaml
# .github/workflows/deploy.yml (for GitHub integration)
name: Deploy to Vercel

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build Angular app
        run: ng build --configuration=production
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_ANON_KEY: ${{ secrets.SUPABASE_ANON_KEY }}
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.ORG_ID }}
          vercel-project-id: ${{ secrets.PROJECT_ID }}
          vercel-args: '--prod'
```

### 16.5 Vercel Configuration Checklist

- [ ] Connect GitHub repository to Vercel project
- [ ] Configure environment variables in Vercel dashboard
- [ ] Set build command: `ng build`
- [ ] Set output directory: `dist/vitalsdrive-dashboard/browser`
- [ ] Configure region (closest to Supabase region)
- [ ] Enable PR preview deployments
- [ ] Set up custom domain (if applicable)
- [ ] Configure cache headers for static assets

### 16.6 Build Optimization

```json
// angular.json production build
{
  "production": {
    "optimization": true,
    "outputHashing": "all",
    "sourceMap": false,
    "namedChunks": false,
    "extractLicenses": true,
    "budgets": [
      {
        "type": "initial",
        "maximumWarning": "500kb",
        "maximumError": "1mb"
      }
    ]
  }
}
```

### 16.7 Supabase Realtime Considerations

The Supabase Realtime WebSocket connection must remain functional through Vercel's serverless architecture:

```typescript
// Ensure WebSocket URL is correctly configured
const supabase = createClient(
  process.env['SUPABASE_URL']!,
  process.env['SUPABASE_ANON_KEY']!,
  {
    realtime: {
      params: {
        apikey: process.env['SUPABASE_ANON_KEY']!
      }
    }
  }
);
```

For Vercel Hobby tier (serverless functions with 10s timeout), the WebSocket connection is maintained client-side only. For production scale, consider:
- Supabase Realtime Pro tier for higher connection limits
- Vercel Pro for increased function timeout
- Dedicated Supabase project for production workloads

---

## Appendix A: Component API Summary

### HealthGaugeComponent

| Input | Type | Default | Description |
|---|---|---|---|
| `value` | `number` | 0 | Current temperature (°C) |
| `size` | `'small' \| 'medium' \| 'large'` | `'medium'` | Gauge size variant |
| `showLabel` | `boolean` | `true` | Show/hide "Coolant Temp" label |

### BatteryStatusComponent

| Input | Type | Default | Description |
|---|---|---|---|
| `vehicleId` | `string` | required | Vehicle identifier |
| `history` | `TelemetryRecord[]` | `[]` | Historical readings |
| `showPrediction` | `boolean` | `true` | Show/hide prediction |

### DtcAlertComponent

| Input | Type | Default | Description |
|---|---|---|---|
| `alert` | `Alert` | required | Alert to display |
| `autoDismiss` | `boolean` | `true` | Auto-dismiss after duration |
| Output: `dismissed` | `EventEmitter<void>` | | Emitted on dismiss |

### FleetMapComponent

| Input | Type | Default | Description |
|---|---|---|---|
| `vehicles` | `Vehicle[]` | `[]` | Vehicles to display |
| `selectedVehicleId` | `string \| null` | `null` | Currently selected |
| `center` | `[number, number]` | auto | Map center |
| `zoom` | `number` | auto | Initial zoom |
| Output: `vehicleSelected` | `EventEmitter<string>` | | Emitted on marker click |

---

## Appendix B: DTC Database (MVP)

### P0xx Codes (Powertrain - Fuel/Air)

| Code | Severity | Description |
|---|---|---|
| P0100 | Medium | Mass Air Flow Circuit Malfunction |
| P0101 | Medium | Mass Air Flow Range/Performance |
| P0110 | Low | Intake Air Temperature Circuit Malfunction |
| P0115 | Low | Engine Coolant Temperature Circuit |
| P0120 | Medium | Throttle Position Sensor Circuit |
| P0130 | Low | O2 Sensor Circuit (Bank 1 Sensor 1) |
| P0135 | Medium | O2 Sensor Heater Circuit (Bank 1 Sensor 1) |

### P03xx Codes (Powertrain - Ignition/Misfire)

| Code | Severity | Description |
|---|---|---|
| P0300 | Medium | Random/Multiple Cylinder Misfire |
| P0301 | Medium | Cylinder 1 Misfire |
| P0302 | Medium | Cylinder 2 Misfire |
| P0303 | Medium | Cylinder 3 Misfire |
| P0304 | Medium | Cylinder 4 Misfire |
| P0325 | High | Knock Sensor 1 Circuit |
| P0335 | High | Crankshaft Position Sensor Circuit |

### P04xx Codes (Powertrain - Emissions)

| Code | Severity | Description |
|---|---|---|
| P0420 | Low | Catalyst System Efficiency Below Threshold |
| P0440 | Low | EVAP System Malfunction |
| P0442 | Low | EVAP System Leak (Small) |
| P0446 | Medium | EVAP Vent System Performance |
| P0455 | Medium | EVAP System Leak (Large) |

---

## Appendix C: Open Questions

1. **Multi-fleet support:** Should dashboard support switching between multiple fleet accounts in MVP?
2. **Historical data retention:** How long should telemetry history be retained in-memory vs. paginated from API?
3. **Alert persistence:** Should alerts persist across sessions (localStorage) or only during active session?
4. **Offline support:** Is PWA/offline mode in scope for MVP?
5. **Dark mode:** Primary design is dark; should light mode be supported?
6. **User authentication:** Full Auth0/Supabase Auth integration or simpler email-based for MVP?

---

*Document Owner: VitalsDrive Engineering  
Next Review: End of Phase 2 (Angular Dashboard completion)*  
