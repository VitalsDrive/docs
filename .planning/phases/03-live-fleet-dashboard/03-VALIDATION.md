---
phase: 3
slug: live-fleet-dashboard
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-05-14
---

# Phase 3 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

Wave 0 executes as Task 1 of Plan 03-01 before any implementation tasks.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest + ts-jest (dashboard) |
| **Config file** | `packages/dashboard/jest.config.js` |
| **Quick run command** | `npm test -- --testPathPattern=vehicle.service` |
| **Full suite command** | `npm test` (from `packages/dashboard/`) |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm test -- --testPathPattern=vehicle.service`
- **After every plan wave:** Run `npm test` (full suite)
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 3-01-01 | 01 | 1 | FLEET-01/02 | — | RLS blocks cross-fleet telemetry reads | manual | Supabase SQL: count returns 0 for foreign user | ❌ W0 | ⬜ pending |
| 3-01-02 | 01 | 1 | FLEET-02 | — | resource() reloads on org change | unit | `npm test -- --testPathPattern=vehicle.service` | ❌ W0 | ⬜ pending |
| 3-01-03 | 01 | 1 | FLEET-01 | — | Stale state returned when telemetry > 15 min old | unit | `npm test -- --testPathPattern=fleet-map` | ❌ W0 | ⬜ pending |
| 3-01-04 | 01 | 1 | FLEET-02 | — | Realtime subscription starts after resource resolves | unit | `npm test -- --testPathPattern=vehicle.service` | ❌ W0 | ⬜ pending |
| 3-01-05 | 01 | 2 | FLEET-01 | — | Map markers update without page refresh | manual | Simulator → parser → Supabase → /map smoke test | — | ⬜ pending |
| 3-02-01 | 02 | 2 | FLEET-03 | — | Health score displayed on vehicle card | unit | `npm test -- --testPathPattern=vehicle-card` | ❌ W0 | ⬜ pending |
| 3-02-02 | 02 | 2 | FLEET-03 | — | Empty state shown when no vehicles | unit | `npm test -- --testPathPattern=dashboard` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `packages/dashboard/src/app/core/services/vehicle.service.spec.ts` — stubs for resource() reactive chain (FLEET-01/02)
- [ ] `packages/dashboard/src/app/features/fleet-map/fleet-map.component.spec.ts` — stubs for `getVehicleState()` stale logic (FLEET-01)
- [ ] `packages/dashboard/src/app/features/dashboard/vehicle-grid/vehicle-card/vehicle-card.component.spec.ts` — stubs for health score display (FLEET-03)
- [ ] `packages/dashboard/src/app/features/dashboard/dashboard.component.spec.ts` — stubs for empty state (FLEET-03)

*Wave 0 must be committed before any Wave 1 task begins execution. Wave 0 is Task 1 of Plan 03-01.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Realtime markers update live | FLEET-01/02 | Requires running simulator+parser+Supabase stack | 1. `npm run dev:parser` 2. `npm run dev:simulator` 3. Open `/map` — verify markers move without refresh |
| RLS blocks cross-fleet reads | FLEET-02 | Requires two real user accounts | Log in as user B, query telemetry for user A's vehicle — expect 0 rows |
| Stale marker visual (grey dimmed) | FLEET-01 | Visual CSS check | Insert telemetry with `timestamp = NOW() - 20min`, open `/map`, verify grey opacity marker |
| Toast on connection status change | FLEET-02 | Visual timing check | Kill parser mid-session, verify disconnect toast; restart, verify reconnect toast |
| Connection pill reflects status | FLEET-02 | Visual/interactive | Open `/map`, verify pill shows 'connected'; kill parser, verify transitions to 'disconnected' |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
