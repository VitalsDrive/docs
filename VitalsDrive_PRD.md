# VitalsDrive — Product Requirements Document
**Version:** 1.0  
**Status:** Draft  
**Last Updated:** March 2026

### Related Documents

| Document | Path |
|---|---|
| System Architecture Overview | `architecture/system-overview.md` |
| Consolidated Data Model | `architecture/data-model.md` |
| Auth Architecture | `architecture/auth-architecture.md` |
| Layer 1: Ingestion Server PRD | `PRD-Layer1-Ingestion-Server.md` |
| Layer 2: Data Storage PRD | `PRD-Layer2-Data-Storage.md` |
| Layer 3: Angular Dashboard PRD | `PRD-Layer3-Angular-Dashboard.md` |
| Ghost Fleet Simulator PRD | `PRD-Ghost-Fleet-Simulator.md` |
| Billing & Subscriptions PRD | `PRD_Billing.md` |
| Customer Management PRD | `PRD_CustomerManagement.md` |
| Hardware Lifecycle PRD | `PRD_HardwareLifecycle.md` |
| Support & Ticketing PRD | `PRD_Support.md` |
| Onboarding PRD | `PRD-Onboarding.md` |

---

## 1. Executive Summary

**Product Name:** VitalsDrive  
**Tagline:** *Fleet Health, Simplified.*

VitalsDrive is a "health-first" vehicle diagnostic and telematics platform targeting small fleet operators (2–15 vehicles). It combines affordable 4G OBD2 hardware with a real-time Angular dashboard to surface the diagnostics that matter most — before they become expensive failures.

The platform is designed to undercut enterprise competitors like Samsara and Verizon Connect on price while beating them on diagnostic depth for small fleets. By leveraging a scale-to-zero cloud stack (Supabase + Railway + Vercel), VitalsDrive can run at near-zero infrastructure cost during the testing and early-pilot phases.

---

## 2. Problem Statement

Small fleet owners (contractors, delivery operators, local service businesses) face two compounding problems:

1. **Unexpected vehicle downtime** caused by undetected mechanical issues (dead batteries, overheating, unchecked engine faults).
2. **Enterprise telematics tools are priced and sized for large fleets** — Samsara and competitors start at $25–$45/vehicle/month with annual contracts and require dedicated IT resources.

There is no affordable, diagnostics-forward option purpose-built for the 2–15 vehicle segment.

---

## 3. Target User

| Attribute | Detail |
|---|---|
| **Primary User** | Small fleet owner/operator (2–15 vehicles) |
| **Industries** | Local delivery, contractors, landscaping, mobile services |
| **Technical Profile** | Non-technical; needs plain-English alerts, not raw data |
| **Pain Points** | Surprise repair bills, vehicles going down mid-shift, no visibility into fleet health |
| **Buying Trigger** | A single costly breakdown or missed delivery due to vehicle failure |

---

## 4. Product Goals

- **MVP Goal:** Prove that real-time OBD2 data can flow from a $25 hardware unit into a polished web dashboard at near-zero recurring cost.
- **Pilot Goal:** Validate that small fleet owners find the "3 Critical Alerts" valuable enough to pay for.
- **Business Goal:** Achieve sustainable unit economics at the $19–$49/vehicle/month price range with >80% gross margins.

---

## 5. MVP Feature Set ("The 3 Alerts")

The MVP is intentionally narrow. We focus exclusively on the three diagnostic signals that prevent the most expensive failures.

| # | Feature | Value Delivered |
|---|---|---|
| 1 | **DTC Alert** | Instant notification when a Check Engine code appears, with the specific code and a plain-English explanation. Prevents owners from ignoring codes until a minor issue becomes a major repair. |
| 2 | **Battery Health Monitor** | Real-time voltage tracking with alerts when the battery is trending toward a no-start event. Prevents the most common roadside failure. |
| 3 | **Coolant Temperature Alert** | Alerts when a vehicle is running hot before engine damage occurs. A $15 thermostat replaced proactively vs. a $4,000 engine replacement. |
| 4 | **Live Fleet Map** | Basic real-time map view of all vehicles. Provides dispatch context and confirms devices are online. |

Features explicitly **excluded from MVP**: Remote DTC clearing, fuel analytics, maintenance scheduling, predictive AI scoring, driver behavior, geofencing.

---

## 6. Pricing & Business Model

### Subscription Tiers (Monthly, Per Vehicle)

| Tier | Price/Vehicle/Mo | Features |
|---|---|---|
| **Foundation** | $19 | Live GPS, Battery Voltage, Engine Temp, Odometer Tracking |
| **Diagnostic+** | $35 | Foundation + Remote DTC Clearing, Fuel Efficiency Analytics, Maintenance Scheduling |
| **Predictive** | $49 | Diagnostic+ + AI-driven parts failure prediction (Alternator/Starter health scores) |

### Unit Economics (Diagnostic+ Tier, Per Vehicle)

| Cost Category | Monthly Cost | Notes |
|---|---|---|
| SIM Connectivity | $0.65 | ~10MB/mo at IoT pay-as-you-go rates (Telnyx/Monogoto) |
| Cloud Hosting | $0.50 | Supabase + Vercel, amortized across user base |
| Hardware Amortization | $4.15 | ~$50 device cost amortized over 12-month contract |
| **Total COGS** | **$5.30** | |
| **Gross Profit** | **$29.70** | **~85% gross margin** |

> **Hardware note:** Test units sourced from Alibaba (SinoTrack/Micodus) at ~$25/unit. Bulk pricing drops to ~$22/unit. The device is provided to customers as part of a 12-month commitment.

---

## 7. Technical Architecture

### 7.1 System Overview

```
[OBD2 Device in Vehicle]
        │
        │  4G TCP (raw hex packets)
        ▼
[Node.js TCP Ingestion Server]  ← Railway.app
        │
        │  Decoded JSON
        ▼
[Supabase PostgreSQL]
        │
        │  WebSocket (Postgres Realtime)
        ▼
[Angular 19/20 Dashboard]  ← Vercel
```

### 7.2 Layer 1 — Ingestion (The Listener)

- **Hosting:** Railway.app (TCP port, e.g. 5050)
- **Role:** Accepts raw TCP connections from 4G OBD2 devices, parses the hex protocol (Login/Data/Heartbeat packets), and decodes binary payloads.
- **Output:** Pushes clean JSON records to Supabase via the REST API.

**Key principle:** The parser must be hardware-agnostic at the output layer — it writes normalized JSON regardless of whether the source is a real device or the Ghost Fleet simulator (see Section 9). Detailed protocol specification is in `PRD-Layer1-Ingestion-Server.md`.

### 7.3 Layer 2 — Data Storage (Supabase)

- **Database:** PostgreSQL (Supabase free tier for MVP)
- **Primary Table:** `telemetry_logs` (vehicle_id, lat, lng, temp, voltage, rpm, dtc_codes, timestamp)
- **Realtime:** Postgres Changes enabled on `telemetry_logs` so the DB pushes new rows directly to subscribed Angular clients via WebSocket.

Full schema, RLS policies, and indexing strategy are in `PRD-Layer2-Data-Storage.md`.

### 7.4 Layer 3 — Frontend Dashboard (Angular)

- **Framework:** Angular 19+
- **State Management:** Angular Signals (`signal` for current vehicle data, `computed` for derived health scores)
- **Component Library:** Angular Material (admin UI) + Leaflet.js (fleet map)

**Key components:**

| Component | Description |
|---|---|
| `TelemetryService` | Subscribes to `telemetry_logs` via Supabase SDK WebSocket |
| `FleetMapComponent` | Leaflet.js map with pulsing vehicle icons when engine is running |
| `HealthGaugeComponent` | Custom SVG gauge — turns red if `coolantTemp > 105°C` |
| `DtcAlertComponent` | Toast/banner notifications with plain-English DTC translations |
| `BatteryStatusComponent` | Voltage trend line with predictive "will fail tonight" logic |

**Performance:** Use RxJS `bufferTime` on high-frequency update streams to prevent UI stuttering during rapid telemetry bursts. Full component architecture is in `PRD-Layer3-Angular-Dashboard.md`.

---

## 8. Infrastructure & Cost Stack

### MVP Phase (Scale-to-Zero / Free Tier)

| Category | Provider | MVP Cost | Scale Cost |
|---|---|---|---|
| Frontend Hosting | Vercel | $0 (Hobby) | ~$20/mo |
| Backend / Database | Supabase | $0 (Free Tier) | ~$25/mo |
| TCP Ingestion Server | Railway.app | $5 free credit | ~$5/mo (usage-based) |
| Hardware (test unit) | Alibaba (SinoTrack/Micodus) | ~$25 (1 unit) | ~$22/unit (bulk) |
| SIM Connectivity | Telnyx / Monogoto | $1/SIM + $0.06/MB | ~$1.50/mo total |
| **Total OpEx** | | **~$0/month** | **~$7/mo per vehicle** |

> The $0/month MVP phase excludes the one-time hardware purchase. All cloud services operate within free tiers during development and early piloting.

---

## 9. Ghost Fleet Simulation Strategy

The "Ghost Fleet" approach allows the full data pipeline to be built and tested before hardware arrives. This is the correct engineering sequence: build against the real architecture, not mocked services.

### How It Works

1. **Ghost Script (Python or Node.js):** Runs locally. Opens a TCP connection to the Railway ingestion server and sends fake hex packets that are byte-for-byte identical to what the Alibaba device sends.
2. **Parser (Railway):** Receives the connection, decodes the hex, writes to Supabase. It has no knowledge of whether the source is a real vehicle or the ghost script.
3. **Angular Dashboard:** Subscribes to Supabase Realtime. Receives and renders the data.

### Why This Matters

- The Angular dashboard is built against **live data flowing through the real pipeline** — not a local mock service.
- When the physical hardware arrives, the only change required is sending an SMS to the device with the Railway server IP and port: `adminip123456 [Railway-IP] [Port]`
- **Zero Angular code changes required** when transitioning from simulation to live hardware.

### Responsibility Boundary (Critical)

```
Ghost Script → [raw hex bytes over TCP] → Parser → [clean JSON] → Supabase → Angular
```

The ghost script's output is **raw binary/hex**, not JSON. JSON is the *parser's output*. This boundary is intentional — it means the parser is fully exercised during simulation, not bypassed.

Full protocol byte specifications and simulator configuration details are in `PRD-Ghost-Fleet-Simulator.md`.

---

## 10. Development Roadmap

### Phase 1 — Simulation Build (Week 1)

**Goal:** Full pipeline running with simulated data.

| Task | Detail |
|---|---|
| Supabase setup | Create project, define `telemetry_logs` schema, enable Realtime |
| Railway setup | Deploy Node.js TCP ingestion server, open port 5050 |
| Ghost Script | Write Python/Node script that sends fake hex packets to Railway every 5 seconds |
| Verify pipeline | Confirm ghost data appears in Supabase table in real time |

**Deliverable:** Data flowing through the complete pipeline. Supabase table shows live rows.

### Phase 2 — Angular Dashboard (Days 4–7)

**Goal:** Polished dashboard built against the live simulation pipeline.

| Task | Detail |
|---|---|
| TelemetryService | Supabase SDK WebSocket subscription, Signal-based state |
| Fleet map | Leaflet.js map with vehicle markers, pulsing icon when RPM > 0 |
| Health gauges | Coolant temp SVG gauge, battery voltage trend |
| DTC alerts | Real-time notification panel with plain-English code descriptions |

**Deliverable:** A fully functional dashboard rendering live data from the ghost script.

### Phase 3 — Hardware Integration (Week 2–4)

**Goal:** One real 4G OBD2 device connected to the live cloud pipeline.

| Task | Detail |
|---|---|
| Hardware procurement | Order 1 SinoTrack/Micodus unit from Alibaba + 1 Telnyx SIM |
| Device configuration | SMS command to set Railway server IP/port on the device |
| Live validation | Drive the test vehicle; confirm dashboard updates in real time |
| Protocol refinement | Validate hex parser against real device packets; adjust offsets as needed |

**Deliverable:** Real vehicle data visible in the Angular dashboard. Zero frontend code changes from Phase 2.

### Phase 4 — Pilot Program (Month 2)

**Goal:** First external user providing real-world feedback.

| Task | Detail |
|---|---|
| Identify pilot user | A local business contact with 1–3 vehicles (delivery, contractor, etc.) |
| Hardware deployment | Provide device free of charge in exchange for structured feedback |
| Monitoring | Set up basic alerting to catch pipeline failures during the pilot |
| Feedback loop | Weekly check-ins; capture pain points, missing features, UI confusion |

**Deliverable:** Real-world validation. Documented bugs and feature gaps. Proof of value for the "3 Alerts" concept.

---

## 11. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Hardware failure / DOA unit | Low | Medium | Source from high-rated 4G Cat-1 Alibaba suppliers; order 2 units for redundancy |
| Device protocol mismatch | Medium | High | Ghost script validates parser logic before hardware arrives; request protocol doc from supplier before purchase |
| Supabase Realtime latency | Low | Low | RxJS `bufferTime` on frontend absorbs burst latency; Supabase Realtime is generally <500ms |
| Free tier limits hit during pilot | Medium | Medium | Supabase free tier supports 500MB DB + 2GB bandwidth — sufficient for 1–5 pilot vehicles |
| Name/brand conflict ("VitalsDrive") | Low | Medium | Verify trademark availability before any customer-facing launch |
| SIM connectivity dead zones | Low | High | Validate SIM coverage in the pilot user's operating area before deployment |

---

## 12. Success Metrics

| Phase | Metric | Target |
|---|---|---|
| Phase 1–2 | Pipeline latency (ghost → dashboard) | < 2 seconds end-to-end |
| Phase 3 | Real hardware data visible in dashboard | Within 30 min of device installation |
| Phase 4 | Pilot user retention after 30 days | 100% (it's free, but they should find it useful) |
| Phase 4 | Bugs/issues reported per vehicle | < 3 critical issues |
| Post-pilot | Conversion to paid tier | ≥ 1 pilot user willing to pay $19+/mo/vehicle |

---

## 13. Open Questions

1. **DTC code database:** Which plain-English DTC translation library will be used? (e.g., open-source OBD-II code databases vs. a licensed API)
2. **Remote DTC clearing (Diagnostic+ tier):** Does the selected hardware support bidirectional commands? Needs confirmation before including in roadmap.
3. **Multi-tenant architecture:** When does the single-tenant Supabase setup need to be refactored for multiple fleet customers?
4. **Mobile app:** Is a native mobile app in scope for Year 1, or is a responsive Angular PWA sufficient?
5. **Hardware subsidy model:** Will VitalsDrive sell/rent hardware directly, or partner with a distributor?

---

*Document Owner: VitalsDrive Product  
Next Review: End of Phase 1*
