# VitalsDrive — Layer 3: Angular Dashboard
**Product Requirements Document**

**Version:** 2.0
**Status:** Draft
**Parent Document:** VitalsDrive_PRD.md (Main PRD)
**Last Updated:** March 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Feature Breakdown](#2-feature-breakdown)
3. [Component Architecture](#3-component-architecture)
4. [State Management](#4-state-management)
5. [Routing](#5-routing)
6. [Real-Time Data Handling](#6-real-time-data-handling)
7. [DTC Reference Database](#7-dtc-reference-database)
8. [Alert System](#8-alert-system)
9. [Performance Requirements](#9-performance-requirements)
10. [Responsive Design](#10-responsive-design)
11. [Accessibility](#11-accessibility)
12. [Testing Strategy](#12-testing-strategy)

---

## 1. Executive Summary

### 1.1 Purpose

This document defines the requirements for **Layer 3 — Angular Dashboard**, the front-end visualization layer of the VitalsDrive fleet health monitoring platform.

### 1.2 Layer Overview

| Attribute | Detail |
|---|---|
| **Framework** | Angular 19+ |
| **Hosting** | Vercel (Hobby → Pro at scale) |
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
└── ShellComponent (layout wrapper)
    ├── HeaderComponent (logo, nav, user menu)
    ├── SidebarComponent (fleet list navigation)
    └── MainContentComponent (router outlet)
        ├── /dashboard → DashboardComponent
        │   ├── VehicleGridComponent (responsive vehicle cards)
        │   │   └── VehicleCardComponent (per-vehicle summary)
        │   │       ├── HealthGaugeComponent (coolant temp)
        │   │       ├── BatteryStatusComponent (voltage + trend)
        │   │       └── DtcIndicatorComponent (alert badges)
        │   └── AlertBannerComponent (top-level alert notifications)
        ├── /map → FleetMapComponent
        │   ├── LeafletMapComponent (core map rendering)
        │   └── VehicleDetailPanelComponent (selected vehicle info)
        ├── /vehicle/:id → VehicleDetailComponent
        │   ├── TelemetryChartComponent (historical data)
        │   ├── DtcListComponent (active DTC codes)
        │   └── BatteryAnalysisComponent (voltage trend + predictions)
        └── /alerts → AlertsComponent
```

### 3.2 Component Specifications

#### HealthGaugeComponent

| Attribute | Specification |
|---|---|
| **Purpose** | Display coolant temperature as animated gauge |
| **Range** | 0°C to 130°C |
| **Warning Threshold** | > 100°C (amber) |
| **Critical Threshold** | > 105°C (red) |
| **Animation** | CSS transitions, 300ms ease-out |
| **States** | Normal (blue/green), Warning (amber), Critical (red pulsing) |

#### BatteryStatusComponent

| Attribute | Specification |
|---|---|
| **Purpose** | Display voltage with trend line and prediction |
| **Voltage Display** | Current value (large) + unit (V) |
| **Trend Line** | Last 30 telemetry readings |
| **Warning Threshold** | < 12.4V sustained |
| **Critical Threshold** | < 12.0V or predictive failure |
| **Prediction Logic** | Linear regression on voltage trend |
| **Prediction Message** | "May not start tonight" when predicted < 12.0V within 2 hours |

#### DtcIndicatorComponent

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

---

## 4. State Management

### 4.1 Signal Architecture

The dashboard uses Angular Signals for synchronous state and RxJS for asynchronous streams (Supabase Realtime subscriptions).

**Core Signals:**

| Signal | Type | Description |
|---|---|---|
| vehicleSignal | `Signal<Vehicle[]>` | All vehicles in user's fleets |
| telemetrySignal | `Signal<Map<string, TelemetryRecord[]>>` | Latest telemetry per vehicle |
| alertsSignal | `Signal<Alert[]>` | Active alerts |
| selectedVehicleSignal | `Signal<string \| null>` | Currently selected vehicle ID |
| connectionStatusSignal | `Signal<'connected' \| 'disconnected'>` | Supabase connection status |

**Computed Signals:**

| Signal | Derives From | Description |
|---|---|---|
| vehicleHealthScore | telemetrySignal | Composite health score from temp, voltage, DTCs |
| activeAlertCount | alertsSignal | Count of unacknowledged alerts |
| vehiclesWithHealth | vehicleSignal + telemetrySignal | Vehicles merged with latest telemetry |

### 4.2 Service Responsibilities

| Service | Responsibility |
|---|---|
| TelemetryService | Supabase Realtime subscription, telemetry distribution via Signals |
| AlertService | Alert creation, dismissal, active alert list management |
| VehicleService | Vehicle registry loading, derived state (health scores) |
| FleetService | Fleet list loading, vehicle-to-fleet mapping |
| DtcTranslationService | DTC code → human-readable description mapping |

### 4.3 Real-Time Data Batching

Supabase Realtime can produce high-frequency updates. The TelemetryService batches incoming messages using `bufferTime(500ms)` to prevent UI thrashing, then updates Signals in a single batch.

---

## 5. Routing

### 5.1 Route Table

| Path | Component | Guard | Description |
|---|---|---|---|
| `/` | Redirect | Auth guard | Redirect to /dashboard |
| `/dashboard` | DashboardComponent | Auth guard | Main fleet overview |
| `/map` | FleetMapComponent | Auth guard | Live fleet map view |
| `/vehicle/:id` | VehicleDetailComponent | Auth guard | Single vehicle deep-dive |
| `/alerts` | AlertsComponent | Auth guard | Alert history and management |
| `/login` | LoginComponent | Guest guard | Sign-in page |
| `/signup` | SignupComponent | Guest guard | Registration page |
| `/auth/callback` | AuthCallbackComponent | None | Supabase OAuth callback |

### 5.2 Auth Guards

- **Auth guard:** Redirects unauthenticated users to /login
- **Guest guard:** Redirects authenticated users away from login/signup to /dashboard
- All guards check Supabase session state before allowing navigation

---

## 6. Real-Time Data Handling

### 6.1 Subscription Model

The TelemetryService subscribes to Supabase Realtime channels for:
- `telemetry_logs` INSERT events — new readings arrive
- `alerts` INSERT and UPDATE events — new alerts and acknowledgment status changes

Subscriptions are scoped to the user's fleet membership (enforced by RLS). The dashboard receives only data the authenticated user is permitted to see.

### 6.2 Memory Management

- Telemetry history is capped at the last N readings per vehicle (default 100)
- Old readings are discarded as new ones arrive
- Channels are cleaned up on component destroy to prevent subscription leaks

### 6.3 Disconnection Handling

- Display a connection status indicator in the header
- When Supabase WebSocket disconnects, show "Reconnecting..." banner
- Supabase SDK auto-reconnects; dashboard resumes receiving data automatically

---

## 7. DTC Reference Database

The dashboard includes a local DTC code reference for translating codes to human-readable descriptions:

| Code | Description | Severity |
|---|---|---|
| P0300 | Random/multiple cylinder misfire detected | Critical |
| P0301 | Cylinder 1 misfire detected | Critical |
| P0302 | Cylinder 2 misfire detected | Critical |
| P0420 | Catalyst system efficiency below threshold | Warning |
| P0128 | Coolant thermostat below regulating temperature | Warning |
| P0171 | System too lean (Bank 1) | Warning |
| P0172 | System too rich (Bank 1) | Warning |
| P0440 | Evaporative emission control system malfunction | Warning |
| P0442 | Evaporative emission system leak detected | Warning |
| P0500 | Vehicle speed sensor malfunction | Info |
| P0505 | Idle air control system malfunction | Info |

The full DTC database should cover all common OBD-II codes. Missing codes fall back to displaying the raw code only.

---

## 8. Alert System

### 8.1 Alert Types

| Type | Trigger | Severity | Display |
|---|---|---|---|
| DTC Alert | New DTC code detected | Varies by code | Toast + banner |
| Battery Alert | Voltage < 12.4V sustained | Warning/Critical | Banner + gauge |
| Coolant Alert | Temperature > 105°C | Critical | Gauge + banner |
| Predictive Alert | Voltage trend predicts failure | Warning | Banner |

### 8.2 Notification Config

- Toast notifications: Single alerts, auto-dismiss after 8 seconds
- Banner notifications: Critical alerts persist until acknowledged
- Sound notification: Optional audio chime for critical alerts (configurable in settings)

---

## 9. Performance Requirements

### 9.1 Bundle Size Budgets

| Category | Warning | Error |
|---|---|---|
| Initial bundle | 500KB | 1MB |
| Lazy-loaded component | 50KB | 100KB |

### 9.2 Change Detection

All components use `OnPush` change detection strategy. Signals handle reactive updates without triggering full change detection cycles.

### 9.3 Virtual Scrolling

For fleets with > 20 vehicles, the vehicle grid uses virtual scrolling to render only visible cards.

### 9.4 Loading Performance

- Initial page load (cold cache): < 3 seconds on 3G
- Time to interactive: < 5 seconds
- Real-time data visible within 1 second of page load

---

## 10. Responsive Design

### 10.1 Breakpoints

| Breakpoint | Width | Layout |
|---|---|---|
| Mobile | < 768px | Single column, stacked vehicle cards |
| Tablet | 768px – 1024px | 2-column grid |
| Desktop | > 1024px | 3-column grid + sidebar |

### 10.2 Mobile Considerations

- Fleet map: Full-screen on mobile, detail panel overlays
- Alert banners: Full-width, swipe to dismiss
- Vehicle cards: Stacked vertically, tap to expand detail
- Sidebar: Collapsible drawer, hamburger menu

---

## 11. Accessibility

### 11.1 Requirements

- All interactive elements have ARIA labels
- Color is never the sole indicator of state (supplement with icons/text)
- Keyboard navigation support for all features
- Screen reader compatible: gauge values read as text, not just visual
- High contrast mode support for dashboard views

---

## 12. Testing Strategy

### 12.1 Unit Tests

- TelemetryService: Subscription setup, message batching, Signal updates
- AlertService: Threshold checks, alert creation, dismissal
- VehicleService: Vehicle loading, health score computation
- Individual components: Template rendering, input/output bindings

**Framework:** Jasmine/Karma with Angular TestBed

### 12.2 E2E Tests

- Full login → dashboard flow
- Real-time telemetry display update
- Alert creation and acknowledgment
- Vehicle detail page navigation

**Framework:** Playwright or Cypress

---

*Layer Owner: Frontend Engineering
Next Review: End of Phase 2 (Dashboard development complete)*
