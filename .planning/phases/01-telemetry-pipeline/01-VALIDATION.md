---
phase: 01
slug: telemetry-pipeline
status: draft
nyquist_compliant: true
wave_0_complete: true
created: 2026-05-11
---

# Phase 01 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest ^29.7.0 + ts-jest ^29.4.9 (verified-compatible pair) |
| **Config file** | `packages/parser/jest.config.js` — created in Wave 1 by 01-01-PLAN.md Task 1 |
| **Quick run command** | `cd packages/parser && npm test -- --testPathPattern=protocol-parser` |
| **Full suite command** | `cd packages/parser && npm test` |
| **Estimated runtime** | ~10 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd packages/parser && npm test`
- **After every plan wave:** Run `cd packages/parser && npm test` (full suite)
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** ~10 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| INGEST-01 | 01-01 | 1 | INGEST-01 | — | N/A | manual | `nc localhost 5050` | — | ⬜ pending |
| INGEST-02a | 01-01 | 1 | INGEST-02 | — | N/A | unit | `npm test -- --testPathPattern=protocol-parser` | ✅ W1 Task 1 | ⬜ pending |
| INGEST-02b | 01-01 | 1 | INGEST-02 | T-01-04 | CRC rejects corrupt packets | unit | `npm test -- --testPathPattern=protocol-parser` | ✅ W1 Task 1 | ⬜ pending |
| INGEST-02c | 01-01 | 1 | INGEST-02 | — | IO 7040/7044/7045/7059 extracted correctly | unit | `npm test -- --testPathPattern=protocol-parser` | ✅ W1 Task 1 | ⬜ pending |
| INGEST-03a | 01-01 | 1 | INGEST-03 | T-01-02 | SUPABASE_SERVICE_ROLE_KEY from env only | unit | `npm test -- --testPathPattern=telemetry-writer` | ✅ W1 Task 1 | ⬜ pending |
| INGEST-03b | 01-01 | 1 | INGEST-03 | — | dtc_codes always `[]` | unit | `npm test -- --testPathPattern=telemetry-writer` | ✅ W1 Task 1 | ⬜ pending |
| INGEST-04 | 01-02 | 2 | INGEST-04 | — | N/A | manual e2e | Run simulator + parser, verify Supabase row | — | ⬜ pending |
| INGEST-05a | 01-01 | 1 | INGEST-05 | T-01-05 | Queue discards oldest, not newest | unit | `npm test -- --testPathPattern=telemetry-writer` | ✅ W1 Task 1 | ⬜ pending |
| INGEST-05b | 01-01 | 1 | INGEST-05 | T-01-03 | Duplicate IMEI closes previous session | unit | `npm test -- --testPathPattern=connection-handler` | ✅ W1 Task 2 | ⬜ pending |
| INGEST-05c | 01-01 | 1 | INGEST-05 | — | Partial buffer discarded on disconnect | unit | `npm test -- --testPathPattern=protocol-parser` | ✅ W1 Task 1 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Wave 0 scaffolding is satisfied inline in 01-01-PLAN.md Task 1 (jest.config + 3 test files + framework install all created before any production module is exercised). No separate Wave 0 plan is required.

- [x] `packages/parser/jest.config.js` — created in 01-01-PLAN.md Task 1
- [x] `packages/parser/tests/protocol-parser.test.ts` — created in 01-01-PLAN.md Task 1 (covers INGEST-02)
- [x] `packages/parser/tests/telemetry-writer.test.ts` — created in 01-01-PLAN.md Task 1 (covers INGEST-03, INGEST-05 queue)
- [x] `packages/parser/tests/connection-handler.test.ts` — created in 01-01-PLAN.md Task 2 (covers INGEST-05 duplicate IMEI)
- [x] Framework install: `cd packages/parser && npm install --save-dev jest@^29.7.0 ts-jest@^29.4.9 @types/jest@^29.5.12` — in 01-01-PLAN.md Task 1 action step 1

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| TCP server accepts connections on :5050 | INGEST-01 | Requires live TCP server process | Start parser, run `nc localhost 5050`, verify connection |
| Simulator emits correct FMC003 IO IDs | INGEST-04 | Requires live parser + Supabase connection | Run simulator + parser, query telemetry_logs, verify IO field values |
| Reconnect scenario: data survives disconnect | INGEST-05 | Requires live TCP + Supabase | Run simulator reconnect test, verify all N packets in Supabase |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references (satisfied inline in Wave 1 Task 1)
- [x] No watch-mode flags
- [x] Feedback latency < 30s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending (awaiting execution)
