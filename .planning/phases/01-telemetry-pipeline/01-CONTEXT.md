# Phase 1: Telemetry Pipeline - Context

**Gathered:** 2026-05-11
**Status:** Ready for planning

<domain>
## Phase Boundary

Parser reliably receives, decodes, and stores Teltonika Codec 8 Extended telemetry data in Supabase. Covers the full path: TCP connection → IMEI auth → binary protocol decode → IO element mapping → Supabase INSERT. Also covers simulator accuracy for end-to-end testing and reconnect handling.

**In scope:** packages/parser/ refactor + Supabase write path, packages/simulator/ IO ID update + reconnect scenario
**Out of scope:** dashboard, auth, alerts, DTC code resolution (Phase 4), device onboarding (Phase 5)

</domain>

<decisions>
## Implementation Decisions

### Parser Structure
- **D-01:** Refactor the 337-line monolith (`packages/parser/src/index.ts`) into separate modules: `ConnectionHandler`, `ProtocolParser`, `DeviceAuth`, `TelemetryWriter`.
- **D-02:** Add Jest unit tests per module. ProtocolParser and TelemetryWriter are the priority — they are independently testable without a live TCP connection.
- **D-03:** Supabase writes are synchronous (`await` each INSERT before ACKing the device). Vehicle telemetry frequency (1–5s) makes synchronous writes acceptable.
- **D-04:** Replace `console.log/error` with Pino structured JSON logging. Log level configurable via `LOG_LEVEL` env var (debug/info/warn/error). PRD requirement N-3.4.1.

### AVL IO Element Mapping (FMC003-accurate)
- **D-05:** Use IO element IDs from the FMC003 parameter list (https://wiki.teltonika-gps.com/view/FMC003_Parameter_list), NOT the PRD's guessed IDs (67/128/179 — those are incorrect for the FMC003).
- **D-06:** Correct IO element ID mapping for `telemetry_logs` fields:
  - `temp` (coolant) → IO ID **7040** (Coolant Temperature)
  - `rpm` → IO ID **7044** (Engine RPM)
  - `speed` → IO ID **7045** (Vehicle Speed) — also available in AVL record header
  - `voltage` → IO ID **7059** (Control Module Voltage / battery voltage)
  - DTC count → IO ID **7038** (Number of DTC) — parse but do not store
- **D-07:** `dtc_codes` written as `[]` (empty array) in Phase 1. The FMC003 only exposes DTC count (not P-codes) in standard AVL packets. DTC code retrieval deferred to Phase 4. No schema migration needed.
- **D-08:** IO element type sizes per Codec 8 Extended (uint16 IO IDs, variable value sizes based on type byte: 1=8-bit, 2=16-bit, 3=32-bit, 4=64-bit).

### Disconnect / Reconnect
- **D-09:** Supabase outage handling: in-memory queue, max 1000 records (configurable via `SUPABASE_QUEUE_MAX_SIZE`). Exponential backoff retry. When queue is full, log warning and discard oldest (not newest). Alert after 3 consecutive Supabase failures.
- **D-10:** TCP disconnect mid-packet: discard partial buffer, clean up session from in-memory Map, log disconnect event. Teltonika devices buffer and retransmit unACKed packets on reconnect — rely on device-side retry, no server-side buffer across sessions.
- **D-11:** Duplicate IMEI login (reconnect): close previous session cleanly, log "duplicate login", accept new session. Do not reject reconnects.

### Logging & Health
- **D-12:** Health endpoint `GET /health` on the same process (HTTP server alongside TCP). Returns status, active connection count, queue depth, last Supabase push timestamp. Returns 5xx when degraded (triggers Railway restart).
- **D-13:** Key log events: server start, device connect/disconnect, login received/rejected, parse error, CRC failure, Supabase push result, queue depth warning.

### Claude's Discretion
- Module file naming and internal class structure within the four modules.
- Exact retry backoff formula (start at 1s, cap at 60s is reasonable).
- Whether to use a single HTTP server for health or a lightweight express/http.createServer — either is fine.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Protocol Specification
- `docs/PRD-Layer1-Ingestion-Server.md` — Primary spec for the parser. Defines packet framing (§4.2), error handling strategy (§4.3), health checks (§4.4), normalized JSON schema (§5.1), env vars (§6.1), test strategy (§9). **NOTE:** IO element IDs in §4.2 (67/128/179) are incorrect for FMC003 — use D-06 in this document instead.
- `https://wiki.teltonika-gps.com/view/FMC003_Parameter_list` — Authoritative FMC003 IO element IDs. OBD II (Bluetooth) section contains the AVL IO IDs used in Codec 8 Extended data packets (7038–7059 range).

### Codebase — Parser
- `packages/parser/src/index.ts` — Existing 337-line OBD2Parser to be refactored. Contains current Codec 8 Extended parsing logic, CRC validation, Supabase write path. Read before implementing to understand what already works.
- `packages/parser/package.json` — Current dependencies (@supabase/supabase-js ^2.39.0, dotenv ^16.3.1). Add `pino` for logging.
- `packages/parser/.env.example` — Environment variable reference (SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, PARSER_PORT, LOG_LEVEL).

### Codebase — Simulator
- `packages/simulator/src/index.ts` — GhostFleetSimulator to be updated with correct FMC003 IO element IDs and reconnect test scenario.

### Database Schema
- `packages/supabase/migrations/` — `telemetry_logs` table definition (vehicle_id, lat/lng, temp, voltage, rpm, dtc_codes TEXT[], timestamp). No schema changes needed for Phase 1.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `packages/parser/src/index.ts` (OBD2Parser): Has working TCP server setup, IMEI login handling (`0x01` ACK), basic Codec 8 Extended packet framing detection, Supabase client initialization. Do not rewrite from scratch — extract and refactor.
- `packages/simulator/src/index.ts` (GhostFleetSimulator): Has working Codec 8 Extended packet builder. Update IO element IDs rather than rewriting.

### Established Patterns
- **Module structure:** Each package has a single entry point and flat src/ structure. New modules go in `packages/parser/src/` (e.g., `connection-handler.ts`, `protocol-parser.ts`, `device-auth.ts`, `telemetry-writer.ts`).
- **TypeScript config:** `commonjs` modules, `strict: true`. Maintain these settings.
- **Supabase writes:** `@supabase/supabase-js` with `SUPABASE_SERVICE_ROLE_KEY` (bypasses RLS). `packages/parser/` already has this wired.

### Integration Points
- **Parser → Supabase:** INSERT into `telemetry_logs`. UPDATE `devices.last_seen`. IMEI lookup from `devices` table.
- **Simulator → Parser:** TCP :5050. Same binary protocol. Updated simulator must emit IO IDs 7040/7044/7045/7059 for data to appear correctly in Supabase.
- **Railway deployment:** TCP port 5050 exposed. Health endpoint must be reachable for Railway health checks. Parser redeployed from `packages/parser/`.

### Known Anti-Patterns to Fix
- Parser monolith (ARCHITECTURE.md) → resolved by D-01 refactor
- `console.log/error` throughout → resolved by D-04 (Pino)

</code_context>

<specifics>
## Specific Ideas

- User confirmed the FMC003 Parameter List (https://wiki.teltonika-gps.com/view/FMC003_Parameter_list) as the authoritative IO element reference during discussion. Downstream agents should treat this as the ground truth for IO element IDs, not the PRD.
- The PRD was written before real device confirmation; its IO IDs are speculative. Section §10.1 open questions (O-1, O-6, O-8) remain open — don't try to resolve them in Phase 1 beyond what's confirmed.
- Simulator reconnect test: simulator should connect, send N packets, force-close the TCP socket, wait 2–3 seconds, reconnect, send another N packets, then verify all records appear in Supabase.

</specifics>

<deferred>
## Deferred Ideas

- **DTC P-codes:** FMC003 sends count only (IO 7038). Actual P-codes (P0301 etc.) require FMC003 event configuration or separate OBD query. → Phase 4 (Alert System).
- **Codec 8 (non-Extended) fallback:** PRD F-2.4.2 mentions Codec 8 fallback. Not needed for Phase 1 — FMC003 uses Codec 8 Extended exclusively. → Future if other devices are added.
- **Prometheus metrics export:** PRD N-3.4.4. Valuable for production observability but not needed for Phase 1. → Post-MVP.
- **CI/CD pipeline:** No GitHub Actions detected. Not scoped to Phase 1. → Post-MVP.

</deferred>

---

*Phase: 1-Telemetry Pipeline*
*Context gathered: 2026-05-11*
