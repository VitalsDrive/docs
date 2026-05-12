---
status: partial
phase: 01-telemetry-pipeline
source: [VERIFICATION.md]
started: 2026-05-12T00:00:00Z
updated: 2026-05-12T00:00:00Z
---

## Current Test

[awaiting human confirmation]

## Tests

### 1. Live End-to-End Pipeline Confirmation (INGEST-04)

expected: Parser+simulator run against live Supabase — telemetry_logs shows temp 80-115, voltage 11.0-13.5, rpm 1000-3500, dtc_codes={}, recent timestamp; RECONNECT_TEST yields 9-10 rows across 2 sessions
result: [pending — or accept 01-02-SUMMARY evidence from 2026-05-12]

Steps:
1. `cd packages/parser && npm run dev` — verify startup logs, TCP :5050, HTTP :8080
2. `curl http://localhost:8080/health` — expect {"status":"ok","activeConnections":0,...}
3. `cd packages/simulator && npm run dev` — run 30 seconds
4. Query telemetry_logs in Supabase — confirm field values and timestamp
5. `RECONNECT_TEST=true PACKETS_PER_SESSION=5 npm run dev` — expect 9-10 new rows

Note: The smoke test was performed on 2026-05-12 during plan 01-02 execution against production project odwctmlawibhaclptsew with all checks passing. If that execution is accepted, mark result: passed and respond "approved".

## Summary

total: 1
passed: 0
issues: 0
pending: 1
skipped: 0
blocked: 0

## Gaps
