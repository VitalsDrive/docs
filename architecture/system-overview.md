# VitalsDrive — System Architecture Overview

**Version:** 1.0
**Status:** Draft
**Last Updated:** March 2026
**Document Owner:** VitalsDrive Engineering

---

## 1. Executive Summary

### 1.1 Purpose

This document provides a comprehensive architectural overview of the VitalsDrive platform, a "health-first" vehicle diagnostic and telematics system. It defines how all layers interact to deliver real-time fleet health monitoring to small fleet operators (2–15 vehicles).

### 1.2 Product Overview

| Attribute | Detail |
|---|---|
| **Product** | VitalsDrive — Fleet Health Monitoring Platform |
| **Tagline** | *Fleet Health, Simplified.* |
| **Target Market** | Small fleet operators (2–15 vehicles) |
| **Core Value** | Affordable, diagnostics-forward monitoring with actionable alerts |
| **MVP Features** | DTC Alert, Battery Health Monitor, Coolant Temperature Alert, Live Fleet Map |

### 1.3 Architecture Highlights

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              VitalsDrive Architecture                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐     raw hex/TCP      ┌──────────────┐     clean JSON     │
│   │   Ghost      │────────────────────▶│   Node.js     │───────────────────▶│
│   │   Fleet      │                      │   Parser      │                    │
│   │   Simulator  │                      │   (Railway)   │                    │
│   └──────────────┘                      └──────────────┘                    │
│                                                │                              │
│   ┌──────────────┐     raw hex/TCP           │         ┌──────────────┐     │
│   │   Real OBD2   │──────────────────────────┘────────▶│   Supabase    │     │
│   │   Devices     │                                ┌────▶│   PostgreSQL  │─────┘
│   └──────────────┘                                │     └──────────────┘     │
│                                                   │            │              │
│                                                   │   WebSocket (Realtime)   │
│                                                   │            ▼              │
│                                                   │     ┌──────────────┐       │
│                                                   └─────│   Angular    │       │
│                                                         │   Dashboard   │       │
│                                                         │   (Vercel)    │       │
│                                                         └──────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.4 Key Architectural Principles

1. **Hardware-Agnostic Output:** The Parser normalizes all device data to a common JSON schema regardless of manufacturer
2. **Real-Time First:** Supabase Realtime (WebSocket) enables instant dashboard updates
3. **Simulation-First Development:** Ghost Fleet enables full pipeline validation before hardware arrives
4. **Scale-to-Zero Infrastructure:** Free tiers utilized during MVP/pilot phases

---

## 2. System Overview

### 2.1 Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    VitalsDrive Platform                               │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                              Layer 0: Simulation                                 │ │
│  │  • Generates raw binary/hex packets per SinoTrack protocol                     │ │
│  │  • Supports multi-vehicle scenarios (up to 50 simultaneous)                    │ │
│  │  • Enables deterministic edge case testing                                     │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                           │                                             │
│                                           │ raw hex (TCP port 5050)                       │
│                                           ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                            Layer 1: TCP Ingestion Server                         │ │
│  │  • Accept TCP connections on port 5050                                         │ │
│  │  • Parse Login packets (0x01), Data packets (0x22), Heartbeat (0x23)          │ │
│  │  • Validate CRC-16 checksums                                                   │ │
│  │  • Map device IMEI to vehicle_id via Supabase lookup                           │ │
│  │  • Push decoded JSON to Supabase REST API                                      │ │
│  │  • Queue outgoing messages during Supabase outage                              │ │
│  │  • Protocols: Alibaba SinoTrack (primary), Micodus (secondary)                 │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                           │                                             │
│                                           │ decoded JSON (REST API)                      │
│                                           ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                          Layer 2: Data Storage (Supabase)                       │ │
│  │  Core Tables: telemetry_logs, vehicles, fleets, users, alerts                  │ │
│  │  Features: RLS for tenant isolation, Realtime WebSocket, DB triggers           │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                           │                                             │
│                                           │ WebSocket (Supabase Realtime)                │
│                                           ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                       Layer 3: Angular Dashboard (Vercel)                       │ │
│  │  State: Angular Signals (sync) + RxJS (async)                                  │ │
│  │  Core Features: DTC Alerts, Battery Monitor, Coolant Temp, Fleet Map            │ │
│  │  Components: VehicleGrid, HealthGauge, BatteryStatus, DtcIndicator, FleetMap    │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Layer Responsibility Summary

| Layer | Component | Hosting | Protocol | Auth | Output | Reference |
|---|---|---|---|---|---|---|
| **L0** | Ghost Fleet Simulator | Local/Dev | TCP (raw hex) | N/A | Binary packets to Parser | `PRD-Ghost-Fleet-Simulator.md` |
| **L1** | TCP Ingestion Server | Railway | TCP → REST | Service role | Normalized JSON to Supabase | `PRD-Layer1-Ingestion-Server.md` |
| **L2** | Data Storage | Supabase | REST/WebSocket | RLS | Telemetry records to Dashboard | `PRD-Layer2-Data-Storage.md` |
| **L3** | Angular Dashboard | Vercel | WebSocket | Supabase Auth | Real-time UI updates | `PRD-Layer3-Angular-Dashboard.md` |

---

## 3. Data Flow Architecture

### 3.1 End-to-End Telemetry Path

```
Vehicle Engine → OBD2 GPS Device → 4G TCP Network → Node.js Parser → Supabase PostgreSQL
     │                │                                    │                    │
  CAN bus         0x78 0x78...                          Parse +              Store +
  telemetry       raw hex                               Validate             Publish via WS
```

### 3.2 Data Transformation at Each Layer

| Stage | Input | Output | Transformation |
|---|---|---|---|
| Vehicle Sensors | Analog/digital signals | CAN bus data | Vehicle ECUs generate OBD-II PIDs |
| OBD2 Device | CAN bus frames | Binary protocol packet | Device firmware encodes to SinoTrack format |
| Network | Binary packet | TCP stream | 4G module transmits over cellular |
| Parser (L1) | Raw hex bytes | Normalized JSON | Protocol decoding, unit conversion, IMEI→vehicle_id mapping |
| Supabase (L2) | JSON record | Database row + WebSocket event | INSERT trigger fires alert generation |
| Angular (L3) | WebSocket payload | UI state update | Signal updates, threshold checks, alert creation |

---

## 4. Component Architecture

### 4.1 Layer 0 — Ghost Fleet Simulator

**Reference:** `PRD-Ghost-Fleet-Simulator.md`

```
┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│  Config        │───▶│  Packet         │───▶│  TCP           │
│  Manager       │    │  Builder        │    │  Client       │
│                │    │                 │    │               │
│  • YAML config │    │  • Login (0x01) │    │  • connect()  │
│  • Env vars    │    │  • Data (0x22)  │    │  • send()     │
│  • CLI flags   │    │  • Heartbeat    │    │  • reconnect  │
│  • Scenarios   │    │    (0x23)       │    │               │
└────────────────┘    └────────────────┘    └───────┬────────┘
                                                    │ TCP Port 5050
                                                    ▼
```

**Simulation Modes:**
| Mode | Description |
|------|-------------|
| Single | 1 vehicle TCP connection |
| Multi-vehicle | N simultaneous TCP connections |
| Scenario-based | Deterministic state transitions (normal → warning → critical) |

### 4.2 Layer 1 — TCP Ingestion Server

**Reference:** `PRD-Layer1-Ingestion-Server.md`

**State Machine per Connection:**
```
CONNECTED → WAIT_LOGIN → AUTHENTICATED → DATA_ACTIVE
             │              │                    │
         recv 0x01      send 0x81 ACK      recv 0x22/0x23
```

**Error Handling Strategy:**

| Scenario | Response |
|---|---|
| Malformed Packet | Discard invalid bytes, log error, continue |
| CRC Failure | Log warning, discard packet |
| Supabase Down | Queue messages, retry with backoff, alert after 3 failures |
| Device Disconnect | Clean up session, log disconnect |
| Memory Pressure | Alert at >200MB, restart at >256MB |

### 4.3 Layer 2 — Data Storage

**Reference:** `PRD-Layer2-Data-Storage.md`

**Schema Relationships:**
```
fleets ──▶ vehicles ◀── fleet_members
              │
              │ references
              ▼
    telemetry_logs ◀── telemetry_rules (trigger → alerts)
```

**Realtime Configuration:**
- `telemetry_logs` published to `supabase_realtime` for INSERT events
- `alerts` published to `supabase_realtime` for all events

**RLS Strategy:**
- `get_user_fleet_ids()` helper function returns fleet UUIDs where user is a member
- telemetry_logs, vehicles, alerts: SELECT allowed when user is a fleet member
- Service role key bypasses RLS for ingestion server writes

### 4.4 Layer 3 — Angular Dashboard

**Reference:** `PRD-Layer3-Angular-Dashboard.md`

**Component Hierarchy:**
```
AppComponent
└── ShellComponent
    ├── HeaderComponent (logo, nav, user menu)
    ├── SidebarComponent (fleet list navigation)
    └── MainContentComponent (router outlet)
        ├── /dashboard → DashboardComponent
        │   ├── VehicleGridComponent
        │   │   └── VehicleCardComponent (HealthGauge, BatteryStatus, DtcIndicator)
        │   └── AlertBannerComponent
        ├── /map → FleetMapComponent
        ├── /vehicle/:id → VehicleDetailComponent
        └── /alerts → AlertsComponent
```

**Signal-Based State Management:**
- `vehicleSignal: Signal<Vehicle[]>`
- `telemetrySignal: Signal<Map<string, TelemetryRecord[]>>`
- `alertsSignal: Signal<Alert[]>`
- `selectedVehicleSignal: Signal<string | null>`
- Computed signals for health scores, active alert counts

---

## 5. Infrastructure Topology

### 5.1 Development Environment

All services run locally:
- Angular dev server: localhost:4200
- TCP Ingestion server: localhost:5050
- Supabase CLI (Docker): localhost:54322 (PostgreSQL), localhost:54323 (Studio)

### 5.2 Staging Environment

- Parser: Railway staging container (TCP port 5050 exposed)
- Supabase: Staging project (Free tier)
- Dashboard: Vercel preview deployment from `develop` branch
- Ghost Fleet runs from GitHub Actions against staging endpoint

### 5.3 Production Environment

- Parser: Railway production (DNS: `ingestion.vitalsdrive.com`)
- Supabase: Production project (Pro tier at scale, Free tier for MVP)
- Dashboard: Vercel production (DNS: `vitalsdrive.com`, main branch only)
- OBD2 devices configured via SMS to connect to Railway IP on port 5050

---

## 6. Security Architecture

### 6.1 Network Security

| Path | Protection |
|---|---|
| OBD2 Device → Railway | 4G cellular network, Railway network isolation, only port 5050 exposed |
| Railway → Supabase | HTTPS (TLS 1.2+), service role key validation |
| Browser → Vercel | HTTPS (TLS 1.3), HSTS, security headers |
| Vercel → Supabase | WebSocket (wss://), anon key + RLS enforcement |

### 6.2 Data Security

| Data Category | At Rest | In Transit | Access Control |
|---|---|---|---|
| Telemetry Logs | Supabase (encrypted) | HTTPS/WSS | RLS (fleet membership) |
| Vehicle Data | Supabase (encrypted) | HTTPS/WSS | RLS (fleet membership) |
| User Credentials | Supabase Auth (bcrypt) | HTTPS | Supabase Auth only |
| Supabase Keys | Railway env vars | N/A | Service role only |

### 6.3 Authentication Architecture

VitalsDrive uses **Supabase Auth** for all authentication:

1. User signs in via Supabase Auth (email/password or OAuth)
2. Session token stored in browser (SDK auto-refresh)
3. Dashboard uses session token for all Supabase requests
4. RLS policies enforce access control at database level via `auth.uid()`

**Migration history (April 2026):**
- Auth0 and custom auth-service (NestJS) removed
- Direct Supabase Auth replaces token exchange flow
- RLS policies now use `auth.uid()` directly

---

## 7. Deployment Architecture

### 7.1 CI/CD Pipeline

```
push to develop → GitHub Actions → Vercel Preview + Railway Staging + Supabase Staging (auto)
merge to main   → GitHub Actions → Vercel Production + Railway Production + Supabase Production (auto)
```

### 7.2 Key Configuration

| Component | Config File | Health Check |
|---|---|---|
| Parser | `railway.toml` (nixpacks builder) | GET /health |
| Dashboard | `vercel.json` (Angular framework) | Vercel platform |

### 7.3 Deployment Steps

**Parser (Railway):**
1. Connect GitHub repo to Railway project
2. Configure build: `npm install && npm start`
3. Set env vars: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `LOG_LEVEL`
4. Auto-deploy on main branch push

**Dashboard (Vercel):**
1. Connect GitHub repo to Vercel project
2. Configure framework: Angular
3. Set env vars: `SUPABASE_URL`, `SUPABASE_ANON_KEY`
4. Auto-deploy on main branch push, preview on PRs

---

## 8. Environment Configuration

### 8.1 Parser Environment Variables

| Variable | Type | Default | Required | Description |
|---|---|---|---|---|
| `PORT` | number | 5050 | Yes | TCP listen port |
| `SUPABASE_URL` | string | — | Yes | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | string | — | Yes | Service role key for writes |
| `LOG_LEVEL` | string | info | No | debug, info, warn, error |
| `CONNECTION_TIMEOUT_MS` | number | 300000 | No | 5 minutes |
| `KEEPALIVE_INTERVAL_MS` | number | 60000 | No | TCP keep-alive probe |
| `SUPABASE_QUEUE_MAX_SIZE` | number | 1000 | No | Max queued writes |
| `DEBUG_RAW_PACKETS` | boolean | false | No | Log hex of each packet |

### 8.2 Dashboard Environment Variables

| Variable | Type | Required | Description |
|---|---|---|---|
| `SUPABASE_URL` | string | Yes | Supabase project URL |
| `SUPABASE_ANON_KEY` | string | Yes | Anonymous key for client |
| `NG_APP_VERSION` | string | No | Auto-populated build version |

### 8.3 Environment Matrix

| Environment | Parser Host | Supabase Project | Dashboard | Purpose |
|---|---|---|---|---|
| **Local** | localhost:5050 | localhost:54322 | localhost:4200 | Individual dev |
| **Staging** | staging.railway.app:5050 | staging project | staging.vercel.app | Integration testing |
| **Production** | ingestion.vitalsdrive.com | production project | vitalsdrive.com | Live system |

---

## 9. Monitoring and Observability

### 9.1 Key Metrics

| Layer | Metric | Alert Threshold |
|---|---|---|
| Parser | `tcp_connections_active` | > 450 (90% capacity) |
| Parser | `packets_parse_errors_total` | > 1% of total |
| Parser | `supabase_push_duration_ms` | p95 > 500ms |
| Parser | `supabase_push_failures_total` | > 0 |
| Parser | `queue_depth` | > 800 |
| Supabase | `database_size` | > 400MB (80%) |
| Supabase | `connection_count` | > 40 |

### 9.2 Alerting

| Alert | Condition | Severity | Action |
|---|---|---|---|
| Server down | /health returns 5xx | P1 | Railway auto-restart |
| Supabase unreachable | 3 consecutive failures | P2 | Slack alert |
| Queue depth critical | > 900 for 5min | P2 | Slack alert |
| Parse error spike | > 5% error rate | P3 | Log review |

---

## 10. Scaling Strategy

### 10.1 Scaling Phases

| Phase | Vehicles | Railway | Supabase | Vercel | Monthly Cost |
|---|---|---|---|---|---|
| MVP | 1-10 | 1 replica | Free | Hobby | $0-7 |
| Growth | 10-50 | 1-2 replicas | Free → Pro ($25) | Hobby → Pro ($20) | $25-50 |
| Scale | 50-200 | 2+ replicas | Pro ($25) | Pro ($20) | $50-100 |

### 10.2 Connection Scaling

| Vehicles | Parser Replicas | Session Management |
|---|---|---|
| 1-50 | 1 | In-memory Map |
| 50-100 | 2 | TCP sticky sessions or Redis |
| 100-500 | 2+ | Redis shared state |

---

## 11. Disaster Recovery

### 11.1 Backup Strategy

| Backup Type | Frequency | Retention | RTO | RPO |
|---|---|---|---|---|
| Supabase Automated Daily | Daily | 7 days | < 1 hour | 24 hours |
| Supabase Point-in-time | Continuous | 30 days | < 1 hour | 1 hour |

### 11.2 Data Retention

| Data Type | Policy |
|---|---|
| telemetry_logs | 90 days, monthly automatic cleanup |
| alerts | Retained indefinitely until manually resolved |

---

## 12. Glossary

| Term | Definition |
|---|---|
| **CAN** | Controller Area Network — vehicle bus standard |
| **DTC** | Diagnostic Trouble Code — OBD-II fault codes |
| **IMEI** | International Mobile Equipment Identity — 15-digit device identifier |
| **OBD2** | On-Board Diagnostics II — vehicle self-diagnostics standard |
| **PID** | Parameter ID — OBD-II data parameter identifier |
| **RLS** | Row Level Security — PostgreSQL access control |
| **RTO** | Recovery Time Objective — time to restore service |
| **RPO** | Recovery Point Objective — maximum acceptable data loss |

---

*Document Version: 1.0 — Last Updated: March 2026*

### Related Documents

| Document | Path |
|---|---|
| Main PRD | `../VitalsDrive_PRD.md` |
| Consolidated Data Model | `data-model.md` |
| Auth Architecture | `auth-architecture.md` |
| Parser PRD | `../PRD-Layer1-Ingestion-Server.md` |
| Data Storage PRD | `../PRD-Layer2-Data-Storage.md` |
| Dashboard PRD | `../PRD-Layer3-Angular-Dashboard.md` |
| Simulator PRD | `../PRD-Ghost-Fleet-Simulator.md` |
| Billing PRD | `../PRD_Billing.md` |
| Hardware Lifecycle PRD | `../PRD_HardwareLifecycle.md` |
| Support PRD | `../PRD_Support.md` |
| Onboarding PRD | `../PRD-Onboarding.md` |
