# Requirements: VitalsDrive

**Defined:** 2026-05-11
**Core Value:** Prevent expensive vehicle failures — DTC codes, dead batteries, and overheating engines — before they cause breakdowns. Fleet Health, Simplified.

## v1 Requirements

### Telemetry Ingestion

- [ ] **INGEST-01**: Parser accepts TCP connections from OBD2 devices on port 5050
- [ ] **INGEST-02**: Parser decodes Teltonika Codec 8 Extended binary protocol
- [ ] **INGEST-03**: Parser stores telemetry records in Supabase with vehicle_id, GPS, RPM, temp, voltage, DTC codes
- [ ] **INGEST-04**: Simulator can emulate FMC003 devices for testing
- [ ] **INGEST-05**: Parser handles device disconnect/reconnect gracefully

### Authentication

- [ ] **AUTH-01**: User can sign up with email and password via Auth0
- [ ] **AUTH-02**: User session persists across browser refresh
- [ ] **AUTH-03**: Auth0 token exchanged for internal JWT via auth service
- [ ] **AUTH-04**: Route guards protect dashboard from unauthenticated access

### Fleet Dashboard

- [ ] **FLEET-01**: User can view live map of all their vehicles with GPS positions
- [ ] **FLEET-02**: Dashboard updates vehicle positions in real-time via Supabase subscriptions
- [ ] **FLEET-03**: User can see health status of each vehicle (health score + alert count)

### DTC Alerts

- [ ] **DTC-01**: System detects when DTC codes appear on a vehicle
- [ ] **DTC-02**: User receives alert with specific DTC code and plain-English explanation
- [ ] **DTC-03**: User can view history of DTC alerts for a vehicle

### Battery Health

- [ ] **BATT-01**: System monitors battery voltage from OBD2 telemetry
- [ ] **BATT-02**: User receives alert when battery voltage trends toward no-start threshold
- [ ] **BATT-03**: Dashboard displays current battery voltage for each vehicle

### Coolant Temperature

- [ ] **COOL-01**: System monitors engine coolant temperature from OBD2 telemetry
- [ ] **COOL-02**: User receives alert when vehicle runs hot (overheating threshold)
- [ ] **COOL-03**: Dashboard displays current coolant temperature for each vehicle

### Device Onboarding

- [ ] **ONBOARD-01**: New OBD2 device can be registered and associated with a fleet
- [ ] **ONBOARD-02**: Onboarding flow guides user through device setup
- [ ] **ONBOARD-03**: Device appears on fleet map once online

## v2 Requirements

### Notifications

- **NOTF-01**: Email notifications for DTC/battery/coolant alerts
- **NOTF-02**: Push notifications for critical alerts
- **NOTF-03**: User can configure notification preferences per vehicle

### Diagnostics+ Tier

- **DIAG-01**: Remote DTC clearing via OBD2 device command
- **DIAG-02**: Fuel efficiency analytics and reporting
- **DIAG-03**: Maintenance scheduling and reminders

### Predictive Tier

- **PRED-01**: AI-driven parts failure prediction (alternator/starter health scores)
- **PRED-02**: Predictive maintenance recommendations

### Admin & Management

- **ADMIN-01**: Fleet admin role management
- **ADMIN-02**: Organization allowlists for access control
- **ADMIN-03**: Vehicle assignment/unassignment

## Out of Scope

| Feature | Reason |
|---------|--------|
| Driver behavior tracking | Not core to vitals monitoring |
| Geofencing | Complex, not aligned with "3 alerts" MVP |
| Fuel analytics | Deferred to Diagnostic+ tier |
| Maintenance scheduling | Deferred to Diagnostic+ tier |
| AI predictive scoring | Deferred to Predictive tier |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| INGEST-01 | Phase 1 | Pending |
| INGEST-02 | Phase 1 | Pending |
| INGEST-03 | Phase 1 | Pending |
| INGEST-04 | Phase 1 | Pending |
| INGEST-05 | Phase 1 | Pending |
| AUTH-01 | Phase 2 | Pending |
| AUTH-02 | Phase 2 | Pending |
| AUTH-03 | Phase 2 | Pending |
| AUTH-04 | Phase 2 | Pending |
| FLEET-01 | Phase 3 | Pending |
| FLEET-02 | Phase 3 | Pending |
| FLEET-03 | Phase 3 | Pending |
| DTC-01 | Phase 4 | Pending |
| DTC-02 | Phase 4 | Pending |
| DTC-03 | Phase 4 | Pending |
| BATT-01 | Phase 4 | Pending |
| BATT-02 | Phase 4 | Pending |
| BATT-03 | Phase 4 | Pending |
| COOL-01 | Phase 4 | Pending |
| COOL-02 | Phase 4 | Pending |
| COOL-03 | Phase 4 | Pending |
| ONBOARD-01 | Phase 5 | Pending |
| ONBOARD-02 | Phase 5 | Pending |
| ONBOARD-03 | Phase 5 | Pending |

**Coverage:**
- v1 requirements: 23 total
- Mapped to phases: 23
- Unmapped: 0

---
*Requirements defined: 2026-05-11*
*Last updated: 2026-05-11 after initial definition*
