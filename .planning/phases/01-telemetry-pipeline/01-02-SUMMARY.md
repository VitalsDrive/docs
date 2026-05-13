---
plan: 01-02
phase: 01
status: complete
completed: "2026-05-12"
requirements_satisfied: [INGEST-04, INGEST-05]
---

# Plan 01-02 Summary: Simulator FMC003 IO ID Update + End-to-End Smoke Test

## What Was Built

Updated `packages/simulator/src/index.ts` to emit correct FMC003 IO IDs matching the refactored parser (Plan 01-01). Added `RECONNECT_TEST` mode to exercise the parser's queue and reconnect handling. Ran end-to-end smoke test confirming full pipeline: simulator → TCP → parser → Supabase.

## IO ID Changes

| Field | Old ID | New ID | Encoding |
|-------|--------|--------|----------|
| Control voltage | 67 | 7059 | uint16 BE, mV (÷1000 = V) |
| Coolant temp | 128 | 7040 | uint16 BE, °C×10 (÷10 = °C) |
| Engine RPM | 179 | 7044 | uint32 BE |
| Vehicle speed | 5 (1-byte) | 7045 | uint16 BE, km/h |

Buffer layout fix also applied: uint16 IO elements use `Buffer.alloc(5)` (2 ID + 1 type + 2 value), uint32 uses `Buffer.alloc(7)` (2 ID + 1 type + 4 value). The plan spec had `alloc(4)`/`alloc(6)` — those caused a runtime `RangeError: offset out of range` on first run and were corrected.

## RECONNECT_TEST Mode

- `RECONNECT_TEST=true` + `PACKETS_PER_SESSION=5`: sends 5 packets, force-closes socket, reconnects, sends 5 more, then exits
- Module-level `sessionCount` counter tracks sessions; `process.exit(0)` after session 2
- `.env` and `.env.example` both document the new vars

## End-to-End Smoke Test Results

Tested against Supabase project `odwctmlawibhaclptsew` (production).

- Parser authenticated IMEI `359632098765432` → returned 0x01 ACK
- Per-packet telemetry_logs rows: `temp` 85–110°C, `voltage` 11.5–13.2V, `rpm` 1500–3000, `speed` 30–80 km/h — all non-zero (proves IO ID mapping correct)
- `dtc_codes = {}` (empty array per D-07) ✓
- RECONNECT_TEST: 10 rows inserted across 2 sessions of 5 packets each ✓
- Health endpoint `/health` returned 200 with `activeConnections: 1` during session, `0` between sessions ✓

## Deviations

- **Buffer size bug in plan spec**: Plan's `<interfaces>` table specified `Buffer.alloc(4)` for uint16 and `Buffer.alloc(6)` for uint32 IO elements. Correct sizes are 5 and 7 respectively (3-byte prefix + value). Fixed at runtime; no impact on committed code.
- **`dotenv` not called**: `packages/simulator/src/index.ts` had `dotenv` as a dependency but never called `dotenv.config()`. Added at session start; `.env` now loads correctly.

## Requirements Satisfied

| Requirement | Evidence |
|-------------|----------|
| INGEST-04: Telemetry data decoded and stored with correct field values | telemetry_logs rows show non-zero temp/voltage/rpm matching simulator-emitted values |
| INGEST-05: Reconnect does not lose data | 10/10 packets persisted across 2 sessions in reconnect test |

## Files Modified

- `packages/simulator/src/index.ts` — FMC003 IO IDs, RECONNECT_TEST mode, buffer size fix, dotenv
- `packages/simulator/.env.example` — added RECONNECT_TEST and PACKETS_PER_SESSION vars
- `packages/simulator/.env` — RECONNECT_TEST=true, PACKETS_PER_SESSION=5 (local only, not committed)
