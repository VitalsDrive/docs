# Phase 1: Telemetry Pipeline - Research

**Researched:** 2026-05-11
**Domain:** Node.js TCP server, Teltonika Codec 8 Extended binary protocol, Supabase write path
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Refactor `packages/parser/src/index.ts` (337-line monolith) into: `ConnectionHandler`, `ProtocolParser`, `DeviceAuth`, `TelemetryWriter`.
- **D-02:** Jest unit tests per module. ProtocolParser and TelemetryWriter are priority (independently testable).
- **D-03:** Supabase writes synchronous (`await` each INSERT before ACKing device). 1–5s telemetry frequency makes this acceptable.
- **D-04:** Replace `console.log/error` with Pino structured JSON logging. `LOG_LEVEL` env var. PRD N-3.4.1.
- **D-05:** IO element IDs from FMC003 parameter list (https://wiki.teltonika-gps.com/view/FMC003_Parameter_list), NOT PRD §4.2 guessed IDs.
- **D-06:** Correct IO element ID mapping:
  - `temp` (coolant) → IO ID **7040**
  - `rpm` → IO ID **7044**
  - `speed` → IO ID **7045** (also available in AVL header)
  - `voltage` → IO ID **7059**
  - DTC count → IO ID **7038** (parse but do not store)
- **D-07:** `dtc_codes` written as `[]` in Phase 1. DTC code retrieval → Phase 4.
- **D-08:** IO element type sizes per Codec 8 Extended (uint16 IO IDs, variable value sizes: 1=8-bit, 2=16-bit, 3=32-bit, 4=64-bit).
- **D-09:** Supabase outage → in-memory queue, max 1000 records (`SUPABASE_QUEUE_MAX_SIZE`). Exponential backoff. Full → discard oldest, log warning. Alert after 3 consecutive failures.
- **D-10:** TCP disconnect mid-packet → discard partial buffer, clean up session, log disconnect. Rely on device-side retry (Teltonika buffers and retransmits unACKed packets).
- **D-11:** Duplicate IMEI login → close previous session cleanly, log "duplicate login", accept new session.
- **D-12:** Health endpoint `GET /health` on same process (HTTP alongside TCP). Returns status, active connection count, queue depth, last Supabase push timestamp. 5xx when degraded.
- **D-13:** Key log events: server start, device connect/disconnect, login received/rejected, parse error, CRC failure, Supabase push result, queue depth warning.

### Claude's Discretion

- Module file naming and internal class structure within the four modules.
- Exact retry backoff formula (start 1s, cap 60s is reasonable).
- Single HTTP server for health: lightweight `http.createServer` or express — either fine.

### Deferred Ideas (OUT OF SCOPE)

- DTC P-codes (IO 7038 gives count only) → Phase 4
- Codec 8 (non-Extended) fallback → future if other devices added
- Prometheus metrics export → post-MVP
- CI/CD pipeline → post-MVP
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| INGEST-01 | Parser accepts TCP connections from OBD2 devices on port 5050 | Node.js `net` module TCP server — already implemented, needs refactor |
| INGEST-02 | Parser decodes Teltonika Codec 8 Extended binary protocol | Codec 0x8E format documented in PRD §4.2; IO IDs corrected via D-06 |
| INGEST-03 | Parser stores telemetry records in Supabase with vehicle_id, GPS, RPM, temp, voltage, DTC codes | `@supabase/supabase-js` already wired; schema confirmed in migration 001 |
| INGEST-04 | Simulator can emulate FMC003 devices for testing | `packages/simulator/src/index.ts` has working packet builder; needs IO ID update |
| INGEST-05 | Parser handles device disconnect/reconnect gracefully | D-09/D-10/D-11 define exact behavior; device-side retry for packet loss |
</phase_requirements>

---

## Summary

This is a brownfield refactor, not a greenfield build. A working 337-line monolith (`packages/parser/src/index.ts`) already implements TCP accept, IMEI login handshake, Codec 8 Extended packet framing, CRC-16-IBM validation, and Supabase writes. The existing simulator (`packages/simulator/src/index.ts`) builds correctly-framed Codec 8 Extended packets. Both compile and run.

The primary work is: (1) fix the IO element IDs — the existing code uses IDs 67/128/179 which are wrong for FMC003; correct IDs are 7040/7044/7045/7059; (2) refactor the monolith into four modules per D-01; (3) add Pino logging; (4) add the in-memory queue for Supabase resilience; (5) add the health HTTP endpoint; (6) add Jest tests; (7) update the simulator with correct IO IDs and reconnect scenario.

The `telemetry_logs` schema already has all required columns (`vehicle_id`, `lat`, `lng`, `temp`, `voltage`, `rpm`, `dtc_codes`, `timestamp`). No schema migration needed.

**Primary recommendation:** Extract and refactor the working code — don't rewrite from scratch. The framing, CRC, and Supabase write path are correct. Only the IO ID mapping needs fixing.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| TCP connection management | Parser (Node.js) | — | Raw binary protocol, must be server-side |
| IMEI authentication | Parser (DeviceAuth) | Supabase (devices table lookup) | Parser validates; Supabase holds truth |
| Binary protocol decode | Parser (ProtocolParser) | — | Device speaks binary; decode before storage |
| Supabase write + retry | Parser (TelemetryWriter) | — | Parser owns the write path |
| In-memory queue (outage buffer) | Parser (TelemetryWriter) | — | Bounded in-process queue per D-09 |
| Health endpoint | Parser (HTTP server) | — | Same process, D-12 |
| FMC003 emulation | Simulator | — | Test-only; mirrors real device behavior |
| Schema / data persistence | Supabase (DB) | — | PostgreSQL owns durability |

---

## Project Constraints (from CLAUDE.md)

- TypeScript: `commonjs` modules, `strict: true`. Do not change tsconfig settings.
- Package isolation: no shared code between packages. Each package is independent.
- Supabase writes use `SUPABASE_SERVICE_ROLE_KEY` (bypasses RLS).
- Dev commands: `npm run dev` (ts-node-dev), `npm run test` (jest), `npm run lint` (eslint).
- Commit convention: Conventional Commits (`feat`, `fix`, `refactor`, etc.).
- 5 separate git repos — commit parser changes to `packages/parser/`, simulator to `packages/simulator/`.

---

## Standard Stack

### Core (already in repo)

| Library | Version | Purpose | Status |
|---------|---------|---------|--------|
| `@supabase/supabase-js` | ^2.39.0 | Supabase client — DB reads/writes | Already installed [VERIFIED: packages/parser/package.json] |
| `dotenv` | ^16.3.1 | Env var loading | Already installed [VERIFIED: packages/parser/package.json] |
| Node.js `net` | built-in | TCP server | Already used [VERIFIED: packages/parser/src/index.ts] |
| Node.js `http` | built-in | Health endpoint HTTP server | No install needed |

### To Add

| Library | Version | Purpose | Why |
|---------|---------|---------|-----|
| `pino` | ^10.3.1 | Structured JSON logging | PRD N-3.4.1 requirement, D-04 [VERIFIED: npm registry] |

### Dev Dependencies to Add

| Library | Version | Purpose |
|---------|---------|---------|
| `jest` | ^29.7.0 | Test runner — pinned to v29 for ts-jest 29 compatibility [VERIFIED: ts-jest 29.x officially supports Jest 27-29] |
| `ts-jest` | ^29.4.9 | TypeScript transform for Jest [VERIFIED: npm registry] |
| `@types/jest` | ^29.5.12 | Jest type definitions matched to jest@29 |

**Note:** Pinned to a known-compatible pair (jest@29 + ts-jest@29). ts-jest 29.x officially supports Jest 27-29; Jest 30 requires ts-jest 30 which is not yet GA. A6 in Assumptions Log resolved by downgrading jest to ^29.7.0.

**Installation:**
```bash
# In packages/parser/
npm install pino
npm install --save-dev jest@^29.7.0 ts-jest@^29.4.9 @types/jest@^29.5.12
```

---

## Architecture Patterns

### Recommended Module Structure

```
packages/parser/src/
├── index.ts                  # Entry point: creates server, wires modules
├── connection-handler.ts     # TCP socket lifecycle (accept, keepalive, timeout, session Map)
├── device-auth.ts            # IMEI login handshake, Supabase devices table lookup
├── protocol-parser.ts        # Codec 8 Extended framing, CRC, AVL record decode, IO extraction
├── telemetry-writer.ts       # Supabase INSERT, in-memory queue, retry backoff
├── health.ts                 # HTTP server for GET /health
├── logger.ts                 # Pino instance, exported singleton
└── types.ts                  # Shared interfaces (TelemetryPacket, ConnectionState, etc.)

packages/parser/tests/
├── protocol-parser.test.ts   # Unit: packet framing, CRC, IO extraction
└── telemetry-writer.test.ts  # Unit: queue behavior, retry logic (Supabase mocked)
```

### Pattern 1: Module Export Pattern (TypeScript CommonJS)

Each module exports a class or factory. `index.ts` imports and wires them.

```typescript
// logger.ts — singleton logger
import pino from 'pino';
export const logger = pino({ level: process.env.LOG_LEVEL || 'info' });
```

```typescript
// protocol-parser.ts — pure function style (easily unit-testable)
export function parseCodec8ExtendedPacket(buffer: Buffer): ParseResult { ... }
export function crc16(data: Buffer): number { ... }
```

### Pattern 2: In-Memory Queue with Exponential Backoff

```typescript
// telemetry-writer.ts
class TelemetryWriter {
  private queue: TelemetryRecord[] = [];
  private maxSize = parseInt(process.env.SUPABASE_QUEUE_MAX_SIZE || '1000', 10);
  private consecutiveFailures = 0;

  async write(record: TelemetryRecord): Promise<void> {
    const { error } = await this.supabase.from('telemetry_logs').insert(record);
    if (error) {
      this.consecutiveFailures++;
      this.enqueue(record);
      if (this.consecutiveFailures >= 3) {
        logger.error({ consecutiveFailures: this.consecutiveFailures }, 'Supabase failure alert');
      }
      this.scheduleRetry();
    } else {
      this.consecutiveFailures = 0;
    }
  }

  private enqueue(record: TelemetryRecord): void {
    if (this.queue.length >= this.maxSize) {
      logger.warn('Queue full — discarding oldest record');
      this.queue.shift(); // discard oldest
    }
    this.queue.push(record);
  }

  private scheduleRetry(): void {
    // Exponential backoff: 1s → 2s → 4s → ... → 60s cap
    const delay = Math.min(1000 * Math.pow(2, this.retryCount), 60000);
    setTimeout(() => this.drainQueue(), delay);
  }
}
```

### Pattern 3: IO Element Extraction (Correct FMC003 IDs)

```typescript
// In ProtocolParser — replace the switch(ioId) block
switch (ioId) {
  case 7040: // Coolant Temperature (uint16 BE, value in tenths of °C per FMC003 spec)
    temp = readValue(payload, ioOffset, ioType) / 10;
    break;
  case 7044: // Engine RPM (uint16 or uint32 depending on FMC003 config)
    rpm = readValue(payload, ioOffset, ioType);
    break;
  case 7045: // Vehicle Speed (km/h)
    speed = readValue(payload, ioOffset, ioType);
    break;
  case 7059: // Control Module Voltage (mV)
    voltage = readValue(payload, ioOffset, ioType) / 1000;
    break;
  case 7038: // Number of DTC — parse but do not store (D-07)
    // dtcCount = readValue(payload, ioOffset, ioType);
    break;
}
```

**Note:** Exact scaling factors for 7040/7059 (°C×10 and mV respectively) are assumed from the FMC003 OBD parameter list naming conventions. The existing code uses the same conventions for IDs 128/67. [ASSUMED — should be verified against real device output or FMC003 wiki when device is available]

### Pattern 4: Health Endpoint

```typescript
// health.ts — lightweight http.createServer (no express dependency)
import * as http from 'http';

export function createHealthServer(getStatus: () => HealthStatus): http.Server {
  return http.createServer((req, res) => {
    if (req.url === '/health' && req.method === 'GET') {
      const status = getStatus();
      const code = status.degraded ? 503 : 200;
      res.writeHead(code, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(status));
    } else {
      res.writeHead(404).end();
    }
  });
}
```

### Pattern 5: Session Map for Duplicate IMEI (D-11)

```typescript
// connection-handler.ts
private sessions = new Map<string, net.Socket>(); // imei → socket

onLogin(imei: string, socket: net.Socket): void {
  const existing = this.sessions.get(imei);
  if (existing) {
    logger.warn({ imei }, 'duplicate login — closing previous session');
    existing.destroy();
  }
  this.sessions.set(imei, socket);
}

onDisconnect(imei: string): void {
  this.sessions.delete(imei);
}
```

### Anti-Patterns to Avoid

- **Do not use `console.log/error`** — all logging through Pino logger (D-04).
- **Do not rewrite packet framing from scratch** — existing CRC-16-IBM and preamble detection logic is correct; extract and reuse it.
- **Do not buffer across sessions** — when TCP disconnects, discard partial buffer. Teltonika devices retransmit unACKed packets (D-10).
- **Do not use `process.exit` on socket errors** — log and continue.
- **Do not hardcode queue size** — read from `SUPABASE_QUEUE_MAX_SIZE` env var.
- **Do not export `OBD2Parser` class from refactored code** — split responsibilities, export individual modules.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Structured logging | Custom log formatter | `pino` | Zero overhead, Railway-compatible JSON, configurable level |
| CRC-16-IBM | New implementation | Extract from existing `crc16()` in index.ts | Already correct, tested implicitly by working simulator |
| Supabase client | HTTP fetch manually | `@supabase/supabase-js` | Already installed, type-safe, handles auth headers |
| Test runner | Custom runner | Jest + ts-jest | Already declared in scripts, no test infrastructure exists yet |

---

## Common Pitfalls

### Pitfall 1: Wrong IO Element IDs
**What goes wrong:** Parser uses IDs 67/128/179 (PRD guesses). All IO fields read as 0. Telemetry stored but useless.
**Why it happens:** PRD §4.2 was written before real device confirmation.
**How to avoid:** Use D-06 IDs (7040/7044/7045/7059). Update simulator to emit same IDs.
**Warning signs:** `temp=0, rpm=0, voltage=0` in Supabase rows despite simulator running.

### Pitfall 2: CRC Calculation Scope Error
**What goes wrong:** CRC calculated over wrong byte range — packets always fail CRC validation, all data dropped.
**Why it happens:** Codec 8 Extended CRC covers from codec ID byte through second num-records field (NOT including preamble or length field).
**How to avoid:** The existing `crc16` function and its call site are correct — preserve them as-is.
**Warning signs:** All packets logging "CRC mismatch" despite valid simulator input.

### Pitfall 3: Timestamp Source
**What goes wrong:** Using `new Date().toISOString()` (server ingestion time) instead of AVL record timestamp.
**Why it happens:** Current code does this. The AVL record has a device-side timestamp (uint64 BE milliseconds since epoch at offset 0).
**How to avoid:** Decide which timestamp to use. Device timestamp = when data was recorded. Server timestamp = when received. For replay/buffered data, device timestamp is more accurate. PRD says "server-side generated" but device timestamp is also available.
**Warning signs:** Timestamps all identical for burst packets (they were buffered, device recorded them at different times).
**Decision needed:** D-03 says synchronous write; PRD §5.1 says server-generated timestamp. Use device timestamp from AVL record — it's already parsed as `BigInt` in the existing code. Convert: `new Date(Number(timestamp)).toISOString()`. [RESOLVED: device AVL timestamp adopted in 01-01-PLAN.md Task 1.]

### Pitfall 4: Jest + ts-jest Config for CommonJS
**What goes wrong:** Jest imports fail with "Cannot use import statement in a module" or "SyntaxError: Unexpected token 'export'".
**Why it happens:** TypeScript CommonJS target requires ts-jest transform configuration.
**How to avoid:** Use this jest.config in `packages/parser/`:
```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/tests/**/*.test.ts'],
  moduleFileExtensions: ['ts', 'js'],
};
```

### Pitfall 5: Queue Not Draining After Supabase Recovery
**What goes wrong:** Queue fills during outage, Supabase recovers, but queue never drains — records lost.
**Why it happens:** Retry is only scheduled on failure. After recovery, no trigger to drain.
**How to avoid:** `drainQueue()` should run after every successful write (flush remaining), or on a periodic timer regardless of failures.

### Pitfall 6: Health Endpoint on Same Port as TCP
**What goes wrong:** HTTP server binds to port 5050 which is already the TCP server.
**Why it happens:** Railway exposes one port by default.
**How to avoid:** Per D-12, health endpoint is on the same *process* but can listen on a different port (e.g., 5051 or Railway's `PORT` env var for HTTP). TCP stays on `PARSER_PORT=5050`. Alternatively, Railway can expose both. [ASSUMED — verify Railway multi-port support or use `PORT` env var for HTTP health]

---

## Code Examples

### Verified: AVL Record Fixed Layout (from existing index.ts, confirmed against PRD §4.2)

```typescript
// Source: packages/parser/src/index.ts (lines 268-275) + PRD §4.2 AVL Data Record Layout
const timestamp = payload.readBigUInt64BE(offset);       // offset 0-7
// priority: payload[offset + 8]                          // offset 8
const lngRaw = payload.readInt32BE(offset + 9);          // offset 9-12
const latRaw = payload.readInt32BE(offset + 13);         // offset 13-16
// altitude: payload.readInt16BE(offset + 17)             // offset 17-18
// angle: payload.readUInt16BE(offset + 19)               // offset 19-20
// satellites: payload.readUInt16BE(offset + 21)          // offset 21-22 (NOTE: PRD says 2 bytes)
const speedRaw = payload.readUInt16BE(offset + 23);      // offset 23-24
// event ID: payload.readUInt16BE(offset + 25)            // offset 25-26
const ioCount = payload.readUInt16BE(offset + 27);       // offset 27-28
// IO elements begin at offset 29
```

**Note:** existing code uses `readUInt16BE` for satellites (2 bytes) matching PRD. ioOffset starts at `offset + 29`.

### Verified: Packet Framing (from existing index.ts, confirmed against PRD §4.2)

```typescript
// Source: packages/parser/src/index.ts (lines 137-216)
// Total packet size = 4 (preamble) + 4 (length) + dataLength
const dataLength = buffer.readUInt32BE(4);
const totalPacketSize = 4 + 4 + dataLength;
// Payload starts at byte 8 (skip preamble + length)
const payload = packet.slice(8);
const codecId = payload[0]; // must be 0x8E
const numRecords = payload.readUInt16BE(1);
```

### Verified: Pino Basic Setup

```typescript
// Source: [CITED: https://getpino.io/#/docs/api]
import pino from 'pino';
export const logger = pino({ level: process.env.LOG_LEVEL || 'info' });

// Usage:
logger.info({ imei, remoteAddress }, 'device authenticated');
logger.error({ err, imei }, 'CRC mismatch');
```

### Verified: Supabase Insert Pattern (from existing index.ts)

```typescript
// Source: packages/parser/src/index.ts (lines 391-409)
const { error } = await this.supabase
  .from('telemetry_logs')
  .insert({
    vehicle_id: telemetry.vehicleId,
    lat: telemetry.lat,
    lng: telemetry.lng,
    temp: telemetry.temp,
    voltage: telemetry.voltage,
    rpm: telemetry.rpm,
    dtc_codes: telemetry.dtcCodes,  // [] for Phase 1 per D-07
    timestamp: telemetry.timestamp
  });
```

---

## State of the Art

| Old Approach | Current Approach | Notes |
|--------------|------------------|-------|
| `console.log` logging | Pino structured JSON | D-04 migration required |
| Monolith class | 4-module split | D-01 refactor |
| Wrong IO IDs (67/128/179) | Correct FMC003 IDs (7040/7044/7045/7059) | Core fix |
| No error queue | In-memory bounded queue + backoff | D-09 |
| No health endpoint | `GET /health` on HTTP | D-12 |
| No tests | Jest + ts-jest | D-02 |

---

## Environment Availability

| Dependency | Required By | Available | Notes |
|------------|------------|-----------|-------|
| Node.js | Parser runtime | ✓ | Running environment confirmed |
| npm | Package install | ✓ | Present |
| `@supabase/supabase-js` | TelemetryWriter | ✓ installed | ^2.39.0 in package.json |
| `pino` | Logger | ✗ not installed | Add to dependencies |
| `jest` | Tests | ✗ not installed | Add to devDependencies |
| `ts-jest` | Tests | ✗ not installed | Add to devDependencies |
| `@types/jest` | Tests | ✗ not installed | Add to devDependencies |
| Supabase (local/prod) | TelemetryWriter integration | External | Not probed — local Docker or prod project `odwctmlawibhaclptsew` |

**Missing with no fallback:** None — all missing items are installable packages.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Jest ^29.7.0 + ts-jest ^29.4.9 (verified-compatible pair) |
| Config file | `packages/parser/jest.config.js` — does not exist yet (Wave 0 gap) |
| Quick run command | `cd packages/parser && npm test -- --testPathPattern=protocol-parser` |
| Full suite command | `cd packages/parser && npm test` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| INGEST-01 | TCP server accepts connections on :5050 | integration (manual smoke) | Manual: start parser, nc localhost 5050 | — manual only |
| INGEST-02 | Codec 8 Extended binary decode correct | unit | `npm test -- --testPathPattern=protocol-parser` | ❌ Wave 0 |
| INGEST-02 | CRC-16-IBM validation accepts valid / rejects corrupt | unit | `npm test -- --testPathPattern=protocol-parser` | ❌ Wave 0 |
| INGEST-02 | IO ID mapping: 7040/7044/7045/7059 extracted correctly | unit | `npm test -- --testPathPattern=protocol-parser` | ❌ Wave 0 |
| INGEST-03 | Supabase INSERT called with correct fields | unit (mock) | `npm test -- --testPathPattern=telemetry-writer` | ❌ Wave 0 |
| INGEST-03 | dtc_codes always written as `[]` | unit (mock) | `npm test -- --testPathPattern=telemetry-writer` | ❌ Wave 0 |
| INGEST-04 | Simulator emits packets with IO IDs 7040/7044/7045/7059 | integration (e2e) | Manual: run simulator + parser, verify Supabase row | — manual only |
| INGEST-05 | Queue fills on Supabase failure, discards oldest when full | unit | `npm test -- --testPathPattern=telemetry-writer` | ❌ Wave 0 |
| INGEST-05 | Duplicate IMEI closes previous session | unit | `npm test -- --testPathPattern=connection-handler` | ❌ Wave 0 |
| INGEST-05 | Partial buffer discarded on disconnect | unit | `npm test -- --testPathPattern=protocol-parser` | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** `cd packages/parser && npm test`
- **Per wave merge:** `cd packages/parser && npm test` (full suite)
- **Phase gate:** Full suite green before `/gsd-verify-work`

### Wave 0 Gaps

- [ ] `packages/parser/jest.config.js` — Jest + ts-jest config for CommonJS TypeScript
- [ ] `packages/parser/tests/protocol-parser.test.ts` — covers INGEST-02
- [ ] `packages/parser/tests/telemetry-writer.test.ts` — covers INGEST-03, INGEST-05 (queue)
- [ ] `packages/parser/tests/connection-handler.test.ts` — covers INGEST-05 (duplicate IMEI)
- [ ] Framework install: `cd packages/parser && npm install --save-dev jest@^29.7.0 ts-jest@^29.4.9 @types/jest@^29.5.12`

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | Partial | IMEI + Supabase devices table lookup (not full auth) |
| V3 Session Management | Yes | In-memory session Map, duplicate login handling (D-11) |
| V4 Access Control | Yes | SUPABASE_SERVICE_ROLE_KEY in env, never in code; inactive/unassigned devices rejected |
| V5 Input Validation | Yes | Packet CRC validation, IMEI regex, length bounds checks |
| V6 Cryptography | No | No crypto operations in this phase |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| IMEI spoofing | Spoofing | Supabase devices table lookup — unknown IMEIs rejected (existing code line 348) |
| Malformed packet flood | DoS | Buffer discard on invalid preamble/CRC; max buffer size implicit in preamble search |
| Memory exhaustion via session leak | DoS | Session Map cleanup on disconnect (D-10); duplicate IMEI closes previous (D-11) |
| Supabase key exposure | Info Disclosure | Railway encrypted env vars; `DEBUG_RAW_PACKETS=false` in production |
| Unbounded queue growth | DoS | `SUPABASE_QUEUE_MAX_SIZE=1000` cap; discard oldest on overflow (D-09) |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | IO ID 7040 = Coolant Temperature in tenths of °C (÷10 for °C) | Code Examples, IO mapping | Wrong scaling → temp values off by 10x or wrong field entirely |
| A2 | IO ID 7059 = Control Module Voltage in mV (÷1000 for V) | Code Examples, IO mapping | Wrong scaling → voltage values wrong |
| A3 | IO ID 7044 = Engine RPM (raw RPM, no scaling) | Code Examples, IO mapping | If scaled, RPM values off |
| A4 | IO ID 7045 = Vehicle Speed in km/h (no scaling) | Code Examples, IO mapping | Minor — speed also in AVL header |
| A5 | ts-jest 29.x is compatible with Jest 30.x | Standard Stack | RESOLVED — downgraded jest to ^29.7.0 to use verified-compatible ts-jest 29 + jest 29 pair |
| A6 | Device AVL timestamp (uint64 ms epoch) should be used rather than server ingestion time | Pitfall 3 | If device timestamp is used for buffered/replayed data, timestamps accurate; if server time used, buffered records show wrong time |
| A7 | Health HTTP endpoint can bind to a different port than TCP :5050 | Pitfall 6 | If Railway only exposes one port, health check unreachable externally |

**High-risk assumptions:** A1–A4 should be verified against https://wiki.teltonika-gps.com/view/FMC003_Parameter_list before implementation.

---

## Open Questions (RESOLVED)

1. **IO element value scaling for 7040/7059/7044/7045** — RESOLVED per D-06.
   - What we know: IDs confirmed as correct (D-06). Existing code uses ÷10 for temp and ÷1000 for voltage on old IDs.
   - **Resolution:** D-06 locks scaling — temp = uint16/10 (°C), rpm = uint32 raw, speed = uint16 raw (km/h), voltage = uint16/1000 (V). Implemented in 01-01-PLAN.md Task 1 protocol-parser.ts switch block. Live validation of A1–A4 against Teltonika FMC003 wiki deferred to first real-device run (Phase 1 acceptance smoke test).

2. **Device AVL timestamp vs. server ingestion time** — RESOLVED per A6.
   - What we know: PRD §5.1 says "server-generated timestamp". AVL record contains device-side uint64 ms timestamp.
   - **Resolution:** Use device AVL timestamp — `new Date(Number(payload.readBigUInt64BE(offset))).toISOString()`. Locked in 01-01-PLAN.md Task 1 behavior block. Rationale: accurate for buffered/replayed packets; supersedes PRD §5.1 prose since A6 carries higher confidence.

3. **Health endpoint port** — RESOLVED per D-12.
   - What we know: D-12 says same process. Railway exposes TCP :5050.
   - **Resolution:** HEALTH_PORT env var, default 8080 (separate from TCP PARSER_PORT=5050). Locked in 01-01-PLAN.md `<env_vars>` and 01-02-PLAN.md index.ts wiring. Railway must expose both ports; if multi-port is unavailable, fallback is to bind HTTP to Railway's `PORT` env var and TCP to PARSER_PORT.

---

## Sources

### Primary (HIGH confidence)
- `packages/parser/src/index.ts` — existing implementation [VERIFIED: read in session]
- `packages/simulator/src/index.ts` — existing simulator [VERIFIED: read in session]
- `packages/supabase/migrations/001_initial_schema.sql` — telemetry_logs schema [VERIFIED: read in session]
- `packages/parser/package.json` — existing dependencies [VERIFIED: read in session]
- `packages/parser/tsconfig.json` — TypeScript config [VERIFIED: read in session]
- `.planning/phases/01-telemetry-pipeline/01-CONTEXT.md` — locked decisions [VERIFIED: read in session]
- `docs/PRD-Layer1-Ingestion-Server.md` — parser PRD [VERIFIED: read in session]
- npm registry: `pino@10.3.1`, `jest@29.7.0`, `ts-jest@29.4.9`, `@types/jest@29.5.12` [VERIFIED: npm view]

### Secondary (MEDIUM confidence)
- Teltonika FMC003 Parameter List (https://wiki.teltonika-gps.com/view/FMC003_Parameter_list) — IO element IDs 7038/7040/7044/7045/7059 [CITED: referenced in CONTEXT.md D-05/D-06, not fetched in this session]
- Pino docs (https://getpino.io/#/docs/api) — basic setup pattern [CITED: standard usage, not fetched in this session]

### Tertiary (LOW confidence)
- (none — ts-jest/jest compatibility resolved by pinning to verified-compatible 29.x pair)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — verified from package.json and npm registry; jest/ts-jest pinned to verified-compatible pair
- Architecture: HIGH — based on existing working code + locked decisions in CONTEXT.md
- IO element IDs: MEDIUM — locked in CONTEXT.md D-06, not independently verified against Teltonika wiki
- IO value scaling: LOW — assumed from naming conventions, not verified
- Pitfalls: HIGH — derived from reading existing code + standard Node.js patterns
- Test infrastructure: HIGH — confirmed no tests exist, gaps are clear

**Research date:** 2026-05-11
**Valid until:** 2026-06-11 (stable stack; Teltonika protocol spec doesn't change)
