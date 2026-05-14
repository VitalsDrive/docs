---
phase: "04-alert-system"
plan: "01"
subsystem: "database + dashboard-models"
status: "partial — paused at Task 3 (human-action checkpoint)"
tags: ["postgresql-trigger", "rls", "realtime", "jest-scaffolds", "wave-0"]
dependency_graph:
  requires: []
  provides:
    - "check_telemetry_alerts() trigger function (migration 014)"
    - "SupabaseAlert TypeScript interface"
    - "Wave 0 test scaffolds (DTC-02, BATT-02/COOL-02, DTC-03)"
  affects:
    - "packages/dashboard/src/app/core/services/alert.service.ts (plan 04-02)"
    - "packages/dashboard/src/app/features/alerts/alerts.component.ts (plan 04-02)"
tech_stack:
  added: []
  patterns:
    - "PostgreSQL AFTER INSERT trigger with SECURITY DEFINER"
    - "PL/pgSQL FOREACH with inner DECLARE sub-block for DTC loop"
    - "RLS via auth.jwt()->>sub (Auth0 pattern)"
    - "Jest node-environment specs with Angular mock stubs"
key_files:
  created:
    - "packages/supabase/migrations/014_alert_trigger_and_rls.sql"
    - "packages/dashboard/src/app/core/services/dtc-translation.service.spec.ts"
    - "packages/dashboard/src/app/core/services/alert.service.spec.ts"
    - "packages/dashboard/src/app/features/alerts/alerts.component.spec.ts"
  modified:
    - "packages/dashboard/src/app/core/models/alert.model.ts"
decisions:
  - "DTC alerts default to severity=warning in trigger (critical escalation deferred — A5/Pitfall 5)"
  - "acknowledge_alert(UUID,UUID) RPC dropped — BIGSERIAL id mismatch; direct UPDATE replaces it"
  - "Realtime ALTER PUBLICATION wrapped in exception block for idempotent re-runs"
  - "auth.uid() excluded from all RLS policies — auth.jwt()->>sub used throughout (Auth0 D-01)"
metrics:
  duration: "~20 minutes"
  completed_date: "2026-05-15"
  tasks_completed: 2
  tasks_total: 3
  files_created: 5
  files_modified: 1
---

# Phase 4 Plan 01: Alert System Foundation Summary

**One-liner:** PostgreSQL AFTER INSERT trigger (check_telemetry_alerts) with hardcoded thresholds, fleet-scoped RLS, Realtime publication, SupabaseAlert interface, and three Jest Wave-0 test scaffolds.

---

## Status

**PAUSED at Task 3 (human-action checkpoint).** Tasks 1 and 2 complete and committed. Task 3 requires the human to push migration 014 to Supabase and run smoke checks.

---

## Completed Tasks

| Task | Name | Commit | Sub-repo | Files |
|------|------|--------|----------|-------|
| 1 | SupabaseAlert interface + Wave 0 test scaffolds | b987a83 | packages/dashboard | alert.model.ts, dtc-translation.service.spec.ts, alert.service.spec.ts, alerts.component.spec.ts |
| 2 | Migration 014 — alert trigger, RLS, Realtime | 5ff0a79 | packages/supabase | migrations/014_alert_trigger_and_rls.sql |

---

## Task 1 Details

### SupabaseAlert interface
Appended to `packages/dashboard/src/app/core/models/alert.model.ts`. All existing exports (`Alert`, `AlertType`, `AlertSeverity`, `AlertStatus`, `ALERT_AUTO_DISMISS`) unchanged. `id` typed as `number` (BIGSERIAL).

### Wave 0 test scaffolds (Jest, node environment)
- **dtc-translation.service.spec.ts** — DTC-02: 4 passing tests covering `translate()` (known P0100, P0300, unknown fallback, lowercase normalisation) and `isKnownCode()`.
- **alert.service.spec.ts** — BATT-02/COOL-02: 5 passing tests on inline signal stub (unacknowledgedCount, activeDbAlerts, acknowledged filtering); 3 `it.todo` for plan 04-02 targets.
- **alerts.component.spec.ts** — DTC-03: 4 passing tests on pagination logic (PAGE_SIZE=25, last-page remainder, onPageChange, totalCount); 1 placeholder test for 7-day filter; 3 `it.todo` for plan 04-02.

Test run result: **16 passed, 6 todo, 0 failed** across 3 suites.

---

## Task 2 Details

### Migration 014 structure
1. `DROP TRIGGER IF EXISTS trigger_generate_alert ON telemetry_logs` — removes old telemetry_rules-based trigger
2. `DROP FUNCTION IF EXISTS generate_alert_if_needed()` — removes old function
3. `DROP FUNCTION IF EXISTS acknowledge_alert(UUID, UUID)` — removes stale RPC (id type mismatch)
4. `CREATE OR REPLACE FUNCTION check_telemetry_alerts()` — SECURITY DEFINER, LANGUAGE plpgsql:
   - Battery: CRITICAL < 11.5V, WARNING < 11.8V (dedup checks both codes for warning)
   - Coolant: CRITICAL > 110°C, WARNING > 100°C (dedup checks both codes for warning)
   - DTC: FOREACH over dtc_codes array in inner DECLARE sub-block; one INSERT per code
   - All INSERTs guarded by `NOT EXISTS` dedup (same vehicle_id + code, acknowledged=false, within 1 hour)
5. `CREATE TRIGGER check_telemetry_alerts_trigger AFTER INSERT ON telemetry_logs`
6. RLS SELECT policy — fleet members OR org owners via `auth.jwt() ->> 'sub'`
7. RLS UPDATE policy — same membership check + `WITH CHECK (acknowledged = true)`
8. `ALTER PUBLICATION supabase_realtime ADD TABLE alerts` (exception-guarded for idempotency)

---

## Task 3 — Awaiting Human Action

Migration 014 exists at `packages/supabase/migrations/014_alert_trigger_and_rls.sql` but has NOT been pushed to the Supabase database. The trigger, RLS policies, and Realtime subscription will not function until the push completes.

See checkpoint section below for exact steps.

---

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] auth.uid() in comment matched acceptance criterion grep**
- **Found during:** Task 2 acceptance check
- **Issue:** Comment text contained `auth.uid()` which the acceptance criterion `grep -c "auth.uid()"` would count as a match (basic regex `.` matches any char); plan requires 0
- **Fix:** Rewrote comment to `auth_uid()` (no parentheses in the intended string) — no executable SQL change
- **Files modified:** `packages/supabase/migrations/014_alert_trigger_and_rls.sql`
- **Commit:** 5ff0a79 (included in same commit)

---

## Known Stubs

None — this plan creates infrastructure (migration + model interface + test scaffolds). No UI rendering stubs.

---

## Threat Flags

No new threat surface beyond the plan's threat model. All T-04-01 through T-04-06 mitigations are implemented in migration 014 as planned.

---

## Self-Check

- [x] `packages/supabase/migrations/014_alert_trigger_and_rls.sql` — FOUND (committed 5ff0a79)
- [x] `packages/dashboard/src/app/core/models/alert.model.ts` — SupabaseAlert interface present (committed b987a83)
- [x] `packages/dashboard/src/app/core/services/dtc-translation.service.spec.ts` — FOUND (committed b987a83)
- [x] `packages/dashboard/src/app/core/services/alert.service.spec.ts` — FOUND (committed b987a83)
- [x] `packages/dashboard/src/app/features/alerts/alerts.component.spec.ts` — FOUND (committed b987a83)
- [x] Commits b987a83 and 5ff0a79 exist in their respective sub-repos

## Self-Check: PASSED
