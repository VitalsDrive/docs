---
phase: 01
plan: 01
subsystem: parser
tags: [parser, teltonika, codec8e, supabase, pino, jest, tcp, health]
dependency_graph:
  requires: []
  provides: [telemetry-ingestion, health-endpoint, fmc003-decode]
  affects: [supabase.telemetry_logs, supabase.devices]
tech_stack:
  added: [pino@10, jest@29, ts-jest@29, "@types/jest@29"]
  patterns: [pino-singleton-logger, supabase-insert-queue-retry, event-emitter-mock-socket, codec8e-crc16]
key_files:
  created:
    - packages/parser/src/types.ts
    - packages/parser/src/logger.ts
    - packages/parser/src/protocol-parser.ts
    - packages/parser/src/telemetry-writer.ts
    - packages/parser/src/device-auth.ts
    - packages/parser/src/connection-handler.ts
    - packages/parser/src/health.ts
    - packages/parser/jest.config.js
    - packages/parser/tests/protocol-parser.test.ts
    - packages/parser/tests/telemetry-writer.test.ts
    - packages/parser/tests/connection-handler.test.ts
    - packages/parser/.env.example
  modified:
    - packages/parser/src/index.ts
    - packages/parser/package.json
    - packages/parser/tsconfig.json
decisions:
  - "FMC003 IO IDs corrected: replaced 67/128/179 with 7040/7044/7045/7059/7038"
  - "TelemetryWriter constructor accepts injected SupabaseClient for testability"
  - "jest.config.js uses transform syntax (not deprecated globals) with rootDir override to allow tests/ outside src/"
  - "tsconfig types extended with jest (types: [node, jest]) without changing module or strict settings"
  - "getAVLRecordSize returns avlRecordSize+2 (includes trailer), so crcEndOffset = 1+2+recordsSize (no extra +2)"
  - "device-auth updateLastSeen logs error and never throws per D-10 pattern"
metrics:
  duration: "~45 minutes"
  completed: "2026-05-12"
  tasks_completed: 2
  files_created: 12
  files_modified: 3
  tests_total: 29
---

# Phase 1 Plan 01: Parser Modularization + FMC003 IO Fix Summary

Replaced the 337-line monolith `src/index.ts` with 8 focused modules. Corrected silent data corruption (all telemetry rows showed temp=0/voltage=0/rpm=0) by replacing wrong IO IDs (67/128/179) with correct FMC003 IDs (7040/7044/7045/7059). Added Pino structured logging, Supabase outage queue with exponential backoff, GET /health endpoint, and full Jest test coverage.

## Tasks Completed

| Task | Commit | Files |
|------|--------|-------|
| 1: types/logger/protocol-parser/telemetry-writer + tests | f8923fd | 9 files |
| 2: device-auth/connection-handler/health/index + connection-handler test | 412adcb | 6 files |

## What Was Built

### 8 Source Modules

| Module | Purpose |
|--------|---------|
| `src/types.ts` | Shared interfaces: TelemetryPacket, DeviceLookup, ConnectionState, HealthStatus, ParsedAVLRecord |
| `src/logger.ts` | Pino singleton logger (LOG_LEVEL env-controlled) |
| `src/protocol-parser.ts` | Pure-function Codec 8 Extended decoder with crc16, FMC003 IO IDs |
| `src/telemetry-writer.ts` | TelemetryWriter class with Supabase INSERT, in-memory queue, exponential backoff |
| `src/device-auth.ts` | DeviceAuth class with IMEI validation, devices lookup, last_seen update |
| `src/connection-handler.ts` | ConnectionHandler class: sessions Map, D-11 duplicate IMEI, D-10 buffer discard |
| `src/health.ts` | createHealthServer factory: GET /health, HTTP 200/503 |
| `src/index.ts` | Thin entry point wiring all modules, TCP :5050 + HTTP :8080 |

### Test Coverage: 29 tests across 3 files

| Suite | Tests | Key Scenarios |
|-------|-------|---------------|
| protocol-parser.test.ts | 11 | CRC accept/reject, IO 7040/7044/7045/7059/7038, partial buffer, unknown codec, legacy IDs prove zero |
| telemetry-writer.test.ts | 10 | INSERT fields, dtc_codes=[], queue enqueue, discard-oldest, 3-failure alert, drain on recovery, backoff delays |
| connection-handler.test.ts | 8 | Duplicate IMEI (D-11), session cleanup, activeConnections, D-10 buffer discard |

### FMC003 IO ID Corrections (D-06)

| Old ID | New ID | Field | Scaling |
|--------|--------|-------|---------|
| 128 | 7040 | Coolant temp | uint16 / 10 = °C |
| 179 | 7044 | Engine RPM | uint32 raw |
| 5 | 7045 | Vehicle speed | uint16 km/h |
| 67 | 7059 | Control voltage | uint16 / 1000 = V |
| — | 7038 | DTC count | parsed, not stored |

### Supabase Outage Queue (D-09)

- Max size: `SUPABASE_QUEUE_MAX_SIZE` (default 1000)
- On overflow: discards oldest, logs warning
- Exponential backoff: 1s → 2s → 4s … capped at 60s
- On recovery: drains full queue via `drainQueue()`, resets retryCount

### Health Endpoint (D-12)

- `GET /health` on `HEALTH_PORT` (default 8080)
- Returns: `{ status, activeConnections, queueDepth, lastSupabasePush, degraded }`
- HTTP 503 when `consecutiveFailures >= 3`

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed incorrect crcEndOffset formula**
- **Found during:** Task 1, protocol-parser.ts implementation
- **Issue:** The plan spec said `crcEndOffset = 1 + 2 + (sum of getAVLRecordSizeAt) + 2`, but `getAVLRecordSize` already adds +2 internally for the trailer numRecords field. Using `1 + 2 + getAVLRecordSize + 2` would compute CRC over 2 extra bytes (out-of-bounds read), causing all CRC validations to fail.
- **Fix:** Used `crcEndOffset = 1 + 2 + recordsSize` (no trailing +2) since `getAVLRecordSize` already accounts for the trailer.
- **Files modified:** `src/protocol-parser.ts`
- **Commit:** f8923fd

**2. [Rule 3 - Blocker] Added jest types to tsconfig**
- **Found during:** Task 1, first test run
- **Issue:** tsconfig.json had `"types": ["node"]` only, causing TS errors: "Cannot find name 'jest'", "Cannot find name 'describe'", etc.
- **Fix:** Added `"jest"` to `types` array in tsconfig; configured jest.config.js `transform` with `tsconfig.rootDir: '.'` to allow test files outside `src/`.
- **Files modified:** `tsconfig.json`, `jest.config.js`
- **Commit:** f8923fd

## Known Stubs

None — all modules are fully implemented. The `vehicleId` in TelemetryPacket is resolved from `devices.vehicle_id` (not hardcoded).

## Threat Surface Scan

All threat mitigations from the STRIDE register are implemented:
- **T-01-01** (Spoofing): IMEI validated with `/^\d{15}$/` in `device-auth.ts`; unknown IMEIs skipped via `lookupDevice` returning null
- **T-01-02** (Key exposure): SUPABASE_SERVICE_ROLE_KEY only in `process.env`, never logged
- **T-01-03** (DoS via sessions): sessions Map bounded by registered devices; duplicate IMEI destroys previous socket
- **T-01-04** (Tampering): CRC-16-IBM validation in `protocol-parser.ts`; mismatch advances buffer past bad packet
- **T-01-05** (Queue DoS): SUPABASE_QUEUE_MAX_SIZE cap with oldest-discard; 60s backoff cap
- **T-01-07** (Repudiation): Pino structured JSON logging with imei + remoteAddress on all parse/auth events

## Self-Check: PASSED

Files verified present:
- packages/parser/src/types.ts ✓
- packages/parser/src/logger.ts ✓
- packages/parser/src/protocol-parser.ts ✓
- packages/parser/src/telemetry-writer.ts ✓
- packages/parser/src/device-auth.ts ✓
- packages/parser/src/connection-handler.ts ✓
- packages/parser/src/health.ts ✓
- packages/parser/src/index.ts ✓
- packages/parser/jest.config.js ✓
- packages/parser/tests/protocol-parser.test.ts ✓
- packages/parser/tests/telemetry-writer.test.ts ✓
- packages/parser/tests/connection-handler.test.ts ✓

Commits verified: f8923fd, 412adcb
Tests: 29/29 passing
tsc --noEmit: 0 errors
console.* in src/: 0 matches
Old IO IDs (67/128/179) in src/: 0 matches
New FMC003 IDs (7038/7040/7044/7045/7059) in protocol-parser.ts: 5 matches
