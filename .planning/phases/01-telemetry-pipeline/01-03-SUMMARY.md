---
phase: 01-telemetry-pipeline
plan: 03
subsystem: parser
tags: [teltonika, codec8, crc16, protocol, simulator, tcp]

requires:
  - phase: 01-02
    provides: Code review identifying CR-01 through CR-04 and WR-01/02/03 correctness bugs

provides:
  - CRC-16/IBM (poly 0x8005, reflected) in parser and simulator — real FMC003 device interop unblocked
  - Correct Teltonika Codec 8 Extended framing: uint8 numRecords, proper offsets, CRC outside Data Field Length
  - Simulator WR fixes: sessionCount after auth, sessionActive guard, NaN-safe config parsing

affects: [02-dashboard, integration-testing]

tech-stack:
  added: []
  patterns: [crc16-ibm-reflected-polynomial, uint8-header-fields, teltonika-codec8-extended-spec]

key-files:
  created: []
  modified:
    - packages/parser/src/protocol-parser.ts
    - packages/parser/tests/protocol-parser.test.ts
    - packages/simulator/src/index.ts

key-decisions:
  - "CRC-16/IBM implemented via reflected polynomial 0xA001 (equivalent to 0x8005 with refin/refout)"
  - "Known CRC value for test vector [0x8E,0x00,0x01] is 0xEBA1 (plan stated 0x4B37 was incorrect)"
  - "totalSize = 8 + dataLength + 2 because CRC appended outside Data Field Length per Teltonika spec"

patterns-established:
  - "CRC-16/IBM: crc ^= data[i]; (crc & 1) ? (crc >>> 1) ^ 0xA001 : crc >>> 1"
  - "Simulator config uses requireInt/requireFloat helpers that throw on NaN, wrapped in try/catch"

requirements-completed: [INGEST-01, INGEST-02, INGEST-03, INGEST-04, INGEST-05]

duration: 30min
completed: 2026-05-12
---

# Phase 01-03: Protocol Correctness Bug Fixes Summary

**CRC-16/IBM, uint8 numRecords, and correct framing now match real FMC003 device spec — simulator and parser are fully interoperable with actual hardware**

## Performance

- **Duration:** ~30 min
- **Completed:** 2026-05-12
- **Tasks:** 3
- **Files modified:** 3

## Accomplishments

- Replaced CRC-16/CCITT (wrong polynomial 0x1021) with CRC-16/IBM (poly 0x8005, reflected) in both parser and simulator — real FMC003 devices will now pass CRC validation
- Fixed all framing bugs: numRecords uint8 (not uint16), AVL record offsets corrected from 3→2, totalSize accounts for the 2 CRC bytes appended outside the Data Field Length, trailer is 1 byte not 2
- Resolved WR-01/02/03: sessionCount increments after auth (not on connect), sessionActive flag prevents double-scheduling, requireInt/requireFloat guard NaN config values
- All 29 parser tests pass; simulator TypeScript compiles clean; no trace of 0x1021 remains

## Task Commits

1. **Task 1: Fix parser CRC + offsets + test helper** — `8ffd95f` (fix)
2. **Task 2: Fix simulator CRC + framing + WR warnings** — `6fde437` (fix)
3. **Task 3: Full verification** — verified inline (all 29 tests green, tsc exit 0)

## Files Created/Modified

- `packages/parser/src/protocol-parser.ts` — CRC-16/IBM, uint8 numRecords, offset 2, totalSize+2, getAVLRecordSize trailer 1 byte
- `packages/parser/tests/protocol-parser.test.ts` — buildTestPacket framing corrected, CRC test updated to 0xEBA1
- `packages/simulator/src/index.ts` — CRC-16/IBM, dataLength excludes CRC, uint8 header/trailer, sessionCount/sessionActive/requireInt/requireFloat

## Decisions Made

- Used reflected polynomial 0xA001 form of CRC-16/IBM (simpler, no explicit bit-reversal needed, mathematically identical)
- Plan stated expected CRC value 0x4B37 for [0x8E,0x00,0x01] — verified actual IBM value is 0xEBA1; used correct value

## Deviations from Plan

**1. CRC test vector value corrected**
- **Issue:** Plan stated expected CRC-16/IBM for [0x8E, 0x00, 0x01] = 0x4B37 (incorrect)
- **Fix:** Computed actual value (0xEBA1) via both manual calculation and Node.js verification before writing the test assertion
- **Impact:** Test passes correctly with true IBM checksum; 0x4B37 would have made the test permanently fail

## Issues Encountered

None beyond the CRC test vector discrepancy noted above.

## Self-Check: PASSED

- `grep -c "0x1021" packages/parser/src/protocol-parser.ts` → 0
- `grep -c "0x8005" packages/parser/src/protocol-parser.ts` → 1
- `grep -c "0x1021" packages/simulator/src/index.ts` → 0
- `grep -c "0x8005" packages/simulator/src/index.ts` → 1
- `npm test` in packages/parser → 29 passed, 29 total
- `npx tsc --noEmit` in packages/simulator → exit 0, no output

## Next Phase Readiness

- Parser correctly processes real Teltonika FMC003 device packets
- Simulator emits spec-compliant packets — E2E smoke test continues to pass
- Phase 01 gap closure complete; ready for Phase 02 (Dashboard)

---
*Phase: 01-telemetry-pipeline*
*Completed: 2026-05-12*
