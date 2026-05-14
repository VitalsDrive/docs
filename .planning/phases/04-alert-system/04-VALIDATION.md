---
phase: 4
slug: alert-system
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-05-14
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest (dashboard: jest + ts-jest) |
| **Config file** | `packages/dashboard/jest.config.js` |
| **Quick run command** | `npm test --prefix packages/dashboard -- --testPathPattern="alert" --passWithNoTests` |
| **Full suite command** | `npm test --prefix packages/dashboard` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm test --prefix packages/dashboard -- --testPathPattern="alert" --passWithNoTests`
- **After every plan wave:** Run `npm test --prefix packages/dashboard`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 4-01-01 | 01 | 0 | DTC-02 | — | DtcTranslationService.translate() returns description for known P-code | unit | `npm test --prefix packages/dashboard -- --testPathPattern="dtc-translation"` | ❌ Wave 0 | ⬜ pending |
| 4-01-02 | 01 | 0 | BATT-02 | — | AlertService.unacknowledgedCount updates when Realtime INSERT arrives | unit | `npm test --prefix packages/dashboard -- --testPathPattern="alert.service"` | ❌ Wave 0 | ⬜ pending |
| 4-01-03 | 01 | 0 | DTC-03 | — | AlertsComponent shows paginated list filtered to last 7 days | unit | `npm test --prefix packages/dashboard -- --testPathPattern="alerts.component"` | ❌ Wave 0 | ⬜ pending |
| 4-01-04 | 01 | 1 | DTC-01 | — | Trigger inserts alert row for each DTC code in telemetry_logs INSERT | manual | supabase SQL smoke | N/A | ⬜ pending |
| 4-01-05 | 01 | 1 | BATT-01 | — | Trigger inserts BATTERY_WARNING when voltage < 11.8V | manual | supabase SQL smoke | N/A | ⬜ pending |
| 4-01-06 | 01 | 1 | COOL-01 | — | Trigger inserts COOLANT_WARNING when temp > 100°C | manual | supabase SQL smoke | N/A | ⬜ pending |
| 4-02-01 | 02 | 2 | COOL-02 | — | Toast shows when new alert arrives via Realtime | unit | `npm test --prefix packages/dashboard -- --testPathPattern="alert.service"` | ❌ Wave 0 | ⬜ pending |
| 4-02-02 | 02 | 2 | BATT-03 | — | VehicleCard shows battery voltage from latestTelemetry | unit | `npm test --prefix packages/dashboard -- --testPathPattern="vehicle-card"` | existing | ⬜ pending |
| 4-02-03 | 02 | 2 | COOL-03 | — | VehicleCard shows coolant temp from latestTelemetry | unit | `npm test --prefix packages/dashboard -- --testPathPattern="vehicle-card"` | existing | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `packages/dashboard/src/app/core/services/alert.service.spec.ts` — stubs for BATT-02, COOL-02 (Realtime mock, unacknowledgedCount)
- [ ] `packages/dashboard/src/app/features/alerts/alerts.component.spec.ts` — stubs for DTC-03 (paginated list, 7-day filter)
- [ ] `packages/dashboard/src/app/core/services/dtc-translation.service.spec.ts` — stubs for DTC-02

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Trigger inserts BATTERY_WARNING when voltage < 11.8V | BATT-01 | PostgreSQL trigger — no Jest harness for DB-side PL/pgSQL | INSERT into telemetry_logs with voltage=11.7, check alerts table for new row |
| Trigger inserts COOLANT_WARNING when temp > 100°C | COOL-01 | PostgreSQL trigger — no Jest harness for DB-side PL/pgSQL | INSERT into telemetry_logs with coolant_temp=105, check alerts table for new row |
| Trigger inserts alert per DTC code | DTC-01 | PostgreSQL trigger — no Jest harness for DB-side PL/pgSQL | INSERT into telemetry_logs with dtc_codes=['P0300','P0301'], verify 2 rows in alerts |
| Dedup guard prevents re-alerting within 1 hour | D-01/D-08 | Requires live DB + timing | INSERT same vehicle+alert_type twice within 1 hour, verify only 1 alert row |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
