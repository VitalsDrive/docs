# Roadmap: VitalsDrive

## Overview

VitalsDrive is a brownfield project with existing infrastructure (parser, dashboard, supabase, simulator, auth-service). This roadmap focuses on completing the MVP: a working fleet health monitoring system with the "3 alerts" (DTC, battery, coolant) visible on a real-time dashboard. Core infrastructure exists; roadmap phases wire the end-to-end user value.

## Phases

- [ ] **Phase 1: Telemetry Pipeline** - Parser reliably ingests and stores OBD2 data to Supabase
- [ ] **Phase 2: Auth & Fleet Management** - User auth, vehicle registration, and fleet organization
- [ ] **Phase 3: Live Fleet Dashboard** - Real-time map and vehicle health overview
- [ ] **Phase 4: Alert System** - DTC, battery, and coolant alerts with notifications
- [ ] **Phase 5: Device Onboarding** - New device setup and fleet expansion flow

## Phase Details

### Phase 1: Telemetry Pipeline
**Goal**: Parser reliably receives, decodes, and stores Teltonika Codec 8 Extended telemetry data in Supabase
**Depends on**: Nothing (first phase)
**Requirements**: [INGEST-01, INGEST-02, INGEST-03, INGEST-04, INGEST-05]
**Success Criteria** (what must be TRUE):
  1. Simulator connects to parser and telemetry appears in Supabase
  2. Parser handles device disconnect/reconnect without losing data
  3. Telemetry records contain all required fields (vehicle_id, GPS, RPM, temp, voltage, DTC)
**Plans**: 2 plans

Plans:
- [ ] 01-01: Parser binary protocol decoding and Supabase write path
- [ ] 01-02: Simulator integration and disconnect/reconnect handling

### Phase 2: Auth & Fleet Management
**Goal**: Users can authenticate via Auth0, manage their fleet, and register vehicles
**Depends on**: Phase 1
**Requirements**: [AUTH-01, AUTH-02, AUTH-03, AUTH-04]
**Success Criteria** (what must be TRUE):
  1. User can sign up/login via Auth0 and access protected dashboard routes
  2. Auth0 token exchanged for internal JWT via auth service
  3. Session persists across browser refresh
  4. Unauthenticated users are redirected to login
**Plans**: 2 plans

Plans:
- [ ] 02-01: Auth0 integration and JWT exchange service
- [ ] 02-02: Route guards and session persistence

### Phase 3: Live Fleet Dashboard
**Goal**: Users can see their fleet on a real-time map with vehicle health status
**Depends on**: Phase 2
**Requirements**: [FLEET-01, FLEET-02, FLEET-03]
**Success Criteria** (what must be TRUE):
  1. Fleet map shows all vehicles with GPS markers updating in real-time
  2. Vehicle list shows health score and alert count per vehicle
  3. Supabase Realtime subscriptions push updates without page refresh
**Plans**: 2 plans

Plans:
- [ ] 03-01: Fleet map component with Leaflet and Supabase Realtime subscriptions
- [ ] 03-02: Vehicle health cards with score and alert indicators

### Phase 4: Alert System
**Goal**: System detects and displays DTC, battery, and coolant alerts for each vehicle
**Depends on**: Phase 3
**Requirements**: [DTC-01, DTC-02, DTC-03, BATT-01, BATT-02, BATT-03, COOL-01, COOL-02, COOL-03]
**Success Criteria** (what must be TRUE):
  1. DTC codes trigger alerts with code and plain-English explanation visible in dashboard
  2. Low battery voltage triggers alert when threshold crossed
  3. High coolant temperature triggers alert when overheating threshold crossed
  4. Dashboard displays current battery voltage and coolant temp per vehicle
**Plans**: 2 plans

Plans:
- [ ] 04-01: Alert detection logic and DTC code lookup (plain-English mapping)
- [ ] 04-02: Alert display components and threshold monitoring UI

### Phase 5: Device Onboarding
**Goal**: New OBD2 devices can be registered and associated with a fleet through a guided flow
**Depends on**: Phase 4
**Requirements**: [ONBOARD-01, ONBOARD-02, ONBOARD-03]
**Success Criteria** (what must be TRUE):
  1. User can register a new device ID and associate it with their fleet
  2. Onboarding flow guides through setup steps
  3. New device appears on fleet map once it starts sending telemetry
**Plans**: 1 plan

Plans:
- [ ] 05-01: Device registration API and onboarding wizard UI

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Telemetry Pipeline | 0/2 | Not started | - |
| 2. Auth & Fleet Mgmt | 0/2 | Not started | - |
| 3. Live Fleet Dashboard | 0/2 | Not started | - |
| 4. Alert System | 0/2 | Not started | - |
| 5. Device Onboarding | 0/1 | Not started | - |
