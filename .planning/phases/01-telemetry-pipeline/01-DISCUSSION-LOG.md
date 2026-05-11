# Phase 1: Telemetry Pipeline - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-11
**Phase:** 1-Telemetry Pipeline
**Areas discussed:** Parser structure, AVL IO field mapping, Disconnect/reconnect, Simulator fidelity

---

## Parser Structure

| Option | Description | Selected |
|--------|-------------|----------|
| Refactor into modules | Split into ConnectionHandler, ProtocolParser, DeviceAuth, TelemetryWriter as architecture docs suggest. Testable, extensible. | ✓ |
| Fix & complete as-is | Keep single-file structure, patch what's broken. Faster but tech debt stays. | |
| Refactor only if broken | Read first, refactor only if functionally incorrect. | |

**User's choice:** Refactor into modules

| Option | Description | Selected |
|--------|-------------|----------|
| Unit tests per module | Jest unit tests for ProtocolParser and TelemetryWriter. Validates the refactor. | ✓ |
| Integration test only | One end-to-end test only. | |
| Defer tests | Ship without tests, add later. | |

**User's choice:** Unit tests per module

| Option | Description | Selected |
|--------|-------------|----------|
| Synchronous per record | await each INSERT before ACKing device. | ✓ |
| Fire-and-forget | INSERT without await, ACK immediately. | |
| Batch/buffer | Buffer records, flush periodically. | |

**User's choice:** Synchronous per record

| Option | Description | Selected |
|--------|-------------|----------|
| Add Pino as part of refactor | PRD requires structured JSON logging (N-3.4.1). | ✓ |
| Keep console.log | Defer Pino to later phase. | |

**User's choice:** Add Pino

---

## AVL IO Field Mapping

**Key finding during discussion:** User linked to FMC003 Parameter List (https://wiki.teltonika-gps.com/view/FMC003_Parameter_list). Research revealed the PRD's IO element IDs (67/128/179) are incorrect for the FMC003 — the actual OBD AVL IDs are in the 7000+ range.

| Option | Description | Selected |
|--------|-------------|----------|
| FMC003 parameter list (7000+ range) | Use IDs from manufacturer spec: 7040=temp, 7044=RPM, 7045=speed, 7059=voltage | ✓ |
| PRD IDs (67/128/179) | Stick with PRD spec (pre-device guesses). | |
| Verify FMC003 Avl Data wiki page | Check separate authoritative source. | |

**User's choice:** FMC003 parameter list (7000+ range)

| Option | Description | Selected |
|--------|-------------|----------|
| Store dtc_codes = [] for now | Only DTC count (IO 7038) available. Defer P-codes to Phase 4. | ✓ |
| Try IO ID 65 as placeholder | Store count as placeholder. | |
| Skip the field entirely | Omit dtc_codes from Phase 1 writes. | |

**User's choice (via recommendation):** Store dtc_codes = [] — deferred to Phase 4

**Notes:** FMC003 does not transmit actual DTC P-codes (P0301 etc.) in standard AVL telemetry packets. Only DTC count is available via IO ID 7038. Actual code retrieval requires FMC003 event configuration. PRD open question O-6 remains open.

---

## Disconnect / Reconnect

| Option | Description | Selected |
|--------|-------------|----------|
| In-memory queue + exponential backoff | 1000 records max, retry with backoff. PRD N-3.2.2 compliance. | ✓ |
| Fire-and-forget + log | No queue, log loss and move on. | |

**User's choice:** In-memory queue + exponential backoff

| Option | Description | Selected |
|--------|-------------|----------|
| Discard partial buffer on disconnect | Rely on Teltonika device retransmit. Simpler server logic. | ✓ |
| Buffer across reconnect | Hold partial packet for same IMEI across sessions. | |

**User's choice:** Discard partial buffer, rely on device retransmit

---

## Simulator Fidelity

| Option | Description | Selected |
|--------|-------------|----------|
| Update to FMC003-accurate IDs | Emit IO 7040/7044/7045/7059. Required for data to map correctly. | ✓ |
| Keep simulator as-is | Exact IDs corrected when real hardware arrives. | |

**User's choice:** Update to FMC003-accurate IDs

| Option | Description | Selected |
|--------|-------------|----------|
| Add reconnect test scenario | Simulator drops TCP connection after N packets, reconnects, verifies all records in Supabase. | ✓ |
| Manual test only | Reconnect tested manually during development. | |

**User's choice:** Add reconnect test scenario

---

## Claude's Discretion

- Module file naming and internal class structure
- Exact retry backoff formula (1s start, 60s cap recommended)
- HTTP server choice for health endpoint (lightweight http.createServer or express)

## Deferred Ideas

- **DTC P-codes** — FMC003 sends count only. Real P-codes deferred to Phase 4 (Alert System).
- **Codec 8 (non-Extended) fallback** — PRD F-2.4.2. Not needed for Phase 1. Future multi-device support.
- **Prometheus metrics** — PRD N-3.4.4. Post-MVP observability.
- **CI/CD pipeline** — No GitHub Actions. Post-MVP.
