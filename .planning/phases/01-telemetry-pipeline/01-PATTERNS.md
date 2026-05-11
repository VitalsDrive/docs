# Phase 1: Telemetry Pipeline - Pattern Map

**Mapped:** 2026-05-11
**Files analyzed:** 10 new/modified files
**Analogs found:** 10 / 10 (all from single source: `packages/parser/src/index.ts` and `packages/simulator/src/index.ts`)

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `packages/parser/src/index.ts` | entry-point | request-response | self (refactor target) | exact |
| `packages/parser/src/types.ts` | model | — | `packages/parser/src/index.ts` lines 5-28 | exact (extract) |
| `packages/parser/src/logger.ts` | utility | — | `packages/parser/src/index.ts` (console.log sites) | role-match |
| `packages/parser/src/connection-handler.ts` | service | event-driven | `packages/parser/src/index.ts` lines 68-92 | exact (extract) |
| `packages/parser/src/device-auth.ts` | service | request-response | `packages/parser/src/index.ts` lines 102-130 | exact (extract) |
| `packages/parser/src/protocol-parser.ts` | service | transform | `packages/parser/src/index.ts` lines 137-333 | exact (extract) |
| `packages/parser/src/telemetry-writer.ts` | service | CRUD | `packages/parser/src/index.ts` lines 335-409 | exact (extract) |
| `packages/parser/src/health.ts` | utility | request-response | none — new capability | no analog |
| `packages/parser/jest.config.js` | config | — | none — new file | no analog |
| `packages/parser/tests/protocol-parser.test.ts` | test | — | none — no existing tests | no analog |
| `packages/parser/tests/telemetry-writer.test.ts` | test | — | none — no existing tests | no analog |
| `packages/parser/tests/connection-handler.test.ts` | test | — | none — no existing tests | no analog |
| `packages/simulator/src/index.ts` | service | event-driven | self (IO ID update + reconnect) | exact |

---

## Pattern Assignments

### `packages/parser/src/types.ts` (model)

**Analog:** `packages/parser/src/index.ts` lines 5-28

**Core interfaces pattern** (lines 5-28):
```typescript
interface TelemetryPacket {
  vehicleId: string;
  imei: string;
  lat: number;
  lng: number;
  speed: number;
  temp: number;
  voltage: number;
  rpm: number;
  dtcCodes: string[];
  timestamp: string;
}

interface DeviceLookup {
  id: string;
  vehicle_id: string | null;
  fleet_id: string;
  status: 'unassigned' | 'active' | 'inactive';
}

interface ConnectionState {
  imei: string | null;
  authenticated: boolean;
}
```

**Notes for planner:** Add `HealthStatus` interface (status, activeConnections, queueDepth, lastSupabasePush, degraded). Add `TelemetryRecord` as alias or rename `TelemetryPacket`. Export all interfaces — consumed by all four modules.

---

### `packages/parser/src/logger.ts` (utility)

**Analog:** None (replaces 9x `console.log/error/warn` call sites across `packages/parser/src/index.ts`)

**Pattern from RESEARCH.md — Pino singleton:**
```typescript
import pino from 'pino';
export const logger = pino({ level: process.env.LOG_LEVEL || 'info' });

// Usage pattern (copy from Pino docs):
logger.info({ imei, remoteAddress }, 'device authenticated');
logger.error({ err, imei }, 'CRC mismatch');
logger.warn({ queueDepth }, 'queue depth warning');
```

**Notes for planner:** Single exported `logger` constant. All modules import this. `pino` must be added to `packages/parser/package.json` dependencies (`npm install pino` in packages/parser/).

---

### `packages/parser/src/connection-handler.ts` (service, event-driven)

**Analog:** `packages/parser/src/index.ts` lines 68-92 (`handleConnection`) and duplicate-IMEI session management.

**TCP socket setup pattern** (lines 68-92):
```typescript
private handleConnection(socket: net.Socket): void {
  const remoteAddress = `${socket.remoteAddress}:${socket.remotePort}`;
  console.log(`New connection from ${remoteAddress}`);

  let buffer: Buffer = Buffer.alloc(0);
  const state: ConnectionState = { imei: null, authenticated: false };

  socket.on('data', async (chunk: Buffer) => {
    buffer = Buffer.concat([buffer, chunk]);
    const result = await this.processBuffer(buffer, remoteAddress, socket, state);
    if (result.buffer) {
      buffer = result.buffer;
    }
  });

  socket.on('close', () => {
    console.log(`Connection closed: ${remoteAddress}${state.imei ? ` (IMEI: ${state.imei})` : ''}`);
  });

  socket.on('error', (err) => {
    console.error(`Socket error (${remoteAddress}):`, err.message);
  });
}
```

**Session Map pattern (D-11 — add, not in existing code):**
```typescript
// From RESEARCH.md Pattern 5
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

**Notes for planner:** On `socket.on('close')` — call `onDisconnect(state.imei)` and discard `buffer` (D-10: no cross-session buffering). Replace all `console.log/error` with `logger`. The `processBuffer` logic splits between `DeviceAuth` (IMEI phase) and `ProtocolParser` (data phase) — `ConnectionHandler` orchestrates, delegates.

---

### `packages/parser/src/device-auth.ts` (service, request-response)

**Analog:** `packages/parser/src/index.ts` lines 102-130 (IMEI login phase of `processBuffer`) and lines 341-376 (`processTelemetry` device lookup).

**IMEI login handshake pattern** (lines 102-130):
```typescript
// Phase 1: Wait for IMEI login (raw 15-digit ASCII)
if (!state.authenticated) {
  if (buffer.length < 15) {
    return { buffer: null };
  }

  const imeiStr = buffer.slice(0, 15).toString('ascii');
  const imeiMatch = imeiStr.match(/^\d{15}$/);

  if (imeiMatch) {
    state.imei = imeiStr;
    state.authenticated = true;
    socket.write(Buffer.from([0x01]));
    console.log(`[${remoteAddress}] Device IMEI: ${state.imei} — authenticated`);
    return { buffer: buffer.slice(15) };
  }

  if (buffer.length > 15) {
    console.error(`[${remoteAddress}] Invalid IMEI prefix: ...`);
    return { buffer: buffer.slice(15) };
  }
  return { buffer: null };
}
```

**Supabase device lookup pattern** (lines 341-376):
```typescript
const { data, error } = await this.supabase
  .from('devices')
  .select('id, vehicle_id, fleet_id, status')
  .eq('imei', telemetry.imei)
  .single() as { data: DeviceLookup | null; error: any };

if (error || !data) {
  console.warn(`[${telemetry.imei}] Unknown device IMEI — not pre-provisioned, skipping telemetry`);
  return;
}

if (data.status === 'inactive') {
  console.warn(`[${telemetry.imei}] Inactive device — skipping telemetry`);
  return;
}

if (!data.vehicle_id) {
  console.warn(`[${telemetry.imei}] Device not assigned to vehicle — skipping telemetry`);
  return;
}
```

**Notes for planner:** Replace `console.*` with `logger`. Accept NACK (send `0x00` byte) when IMEI is invalid — existing code silently closes, PRD requires explicit rejection. `DeviceAuth` also updates `last_seen` (extract `updateDeviceLastSeen` at line 379).

---

### `packages/parser/src/protocol-parser.ts` (service, transform)

**Analog:** `packages/parser/src/index.ts` lines 137-333 (packet framing, CRC, AVL decode).

**Packet framing pattern** (lines 137-216):
```typescript
// Look for preamble: 0x00 0x00 0x00 0x00
let preambleIndex = -1;
for (let i = 0; i <= buffer.length - 4; i++) {
  if (buffer[i] === 0x00 && buffer[i+1] === 0x00 && buffer[i+2] === 0x00 && buffer[i+3] === 0x00) {
    preambleIndex = i;
    break;
  }
}

const dataLength = buffer.readUInt32BE(4);
const totalPacketSize = 4 + 4 + dataLength;
const payload = packet.slice(8); // skip preamble + length
const codecId = payload[0]; // must be 0x8E
const numRecords = payload.readUInt16BE(1);
```

**CRC pattern** (lines 186-199) — PRESERVE EXACTLY:
```typescript
function crc16(data: Buffer): number {
  let crc = 0x0000;
  for (let i = 0; i < data.length; i++) {
    crc ^= data[i] << 8;
    for (let j = 0; j < 8; j++) {
      if (crc & 0x8000) {
        crc = (crc << 1) ^ 0x1021;
      } else {
        crc = crc << 1;
      }
    }
  }
  return crc & 0xFFFF;
}

// CRC scope: payload.slice(0, crcEndOffset) — NOT including preamble or length
const crcData = payload.slice(0, crcEndOffset);
const expectedCrc = payload.readUInt16BE(crcEndOffset);
const calculatedCrc = crc16(crcData);
```

**AVL record fixed layout** (lines 268-279) — VERIFIED against PRD §4.2:
```typescript
const timestamp = payload.readBigUInt64BE(offset);       // offset 0-7
const lngRaw = payload.readInt32BE(offset + 9);          // offset 9-12
const latRaw = payload.readInt32BE(offset + 13);         // offset 13-16
const speedRaw = payload.readUInt16BE(offset + 23);      // offset 23-24
const ioCount = payload.readUInt16BE(offset + 27);       // offset 27-28
let ioOffset = offset + 29;                              // IO elements start
```

**IO element iteration pattern** (lines 286-318) — UPDATE IO IDs per D-06:
```typescript
// EXISTING (WRONG — old IDs):
case 67:  voltage = payload.readUInt16BE(ioOffset) / 1000; break;
case 128: temp = payload.readUInt16BE(ioOffset) / 10; break;
case 179: rpm = payload.readUInt32BE(ioOffset); break;

// REPLACE WITH (D-06 correct FMC003 IDs):
case 7059: voltage = payload.readUInt16BE(ioOffset) / 1000; break; // mV → V
case 7040: temp = payload.readUInt16BE(ioOffset) / 10; break;      // tenths °C → °C
case 7044: rpm = payload.readUInt32BE(ioOffset); break;             // raw RPM
case 7045: speed = payload.readUInt16BE(ioOffset); break;           // km/h
case 7038: /* DTC count — parse, do not store (D-07) */ break;

// IO type → value size (lines 309-317) — PRESERVE EXACTLY:
let valueSize = 1;
switch (ioType) {
  case 1: valueSize = 1; break;
  case 2: valueSize = 2; break;
  case 3: valueSize = 4; break;
  case 4: valueSize = 8; break;
  case 5: valueSize = 4; break;
  case 6: valueSize = 8; break;
  default: valueSize = 1;
}
ioOffset += valueSize;
```

**Timestamp pattern** (line 331) — use device timestamp per RESEARCH open question #2:
```typescript
// EXISTING (wrong — server time):
timestamp: new Date().toISOString()

// REPLACE WITH (device AVL timestamp):
timestamp: new Date(Number(timestamp)).toISOString()
```

**Notes for planner:** Export as pure functions (`parseCodec8ExtendedPacket`, `crc16`) for testability (D-02). `getAVLRecordSizeAt` and `getAVLRecordSize` are helpers — keep as unexported functions or move to private methods. IO IDs 7040/7044/7045/7059 are `uint16 BE` IDs (2 bytes each) — already handled by `payload.readUInt16BE(ioOffset)` at the start of the loop.

---

### `packages/parser/src/telemetry-writer.ts` (service, CRUD)

**Analog:** `packages/parser/src/index.ts` lines 390-409 (`writeToSupabase`) and lines 379-388 (`updateDeviceLastSeen`).

**Supabase INSERT pattern** (lines 390-409):
```typescript
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
  } as any);

if (error) {
  console.error('Failed to write to Supabase:', error);
} else {
  console.log(`[telemetry] Written: vehicle=${telemetry.vehicleId}, ...`);
}
```

**Supabase client init pattern** (lines 54-57):
```typescript
this.supabase = createClient(
  process.env.SUPABASE_URL || '',
  process.env.SUPABASE_SERVICE_ROLE_KEY || ''
);
```

**Queue + retry pattern (D-09 — new, not in existing code):**
```typescript
// From RESEARCH.md Pattern 2
class TelemetryWriter {
  private queue: TelemetryRecord[] = [];
  private maxSize = parseInt(process.env.SUPABASE_QUEUE_MAX_SIZE || '1000', 10);
  private consecutiveFailures = 0;
  private retryCount = 0;

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
      this.retryCount = 0;
      this.lastPushTimestamp = new Date().toISOString();
      // Drain remaining queue after successful write
      if (this.queue.length > 0) {
        this.drainQueue();
      }
    }
  }

  private enqueue(record: TelemetryRecord): void {
    if (this.queue.length >= this.maxSize) {
      logger.warn({ queueDepth: this.queue.length }, 'Queue full — discarding oldest record');
      this.queue.shift();
    }
    this.queue.push(record);
  }

  private scheduleRetry(): void {
    const delay = Math.min(1000 * Math.pow(2, this.retryCount), 60000);
    this.retryCount++;
    setTimeout(() => this.drainQueue(), delay);
  }
}
```

**Notes for planner:** Expose `queueDepth` and `lastPushTimestamp` as public getters for the health endpoint (D-12). `drainQueue()` must reset `retryCount` on success to avoid permanent backoff cap. `updateDeviceLastSeen` can stay in `TelemetryWriter` or move to `DeviceAuth` — either is fine per Claude's Discretion.

---

### `packages/parser/src/health.ts` (utility, request-response)

**Analog:** None — new capability. No existing HTTP server in parser package.

**Pattern from RESEARCH.md Pattern 4:**
```typescript
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

**HealthStatus shape (from D-12):**
```typescript
interface HealthStatus {
  status: 'ok' | 'degraded';
  activeConnections: number;
  queueDepth: number;
  lastSupabasePush: string | null;
  degraded: boolean;
}
```

**Notes for planner:** Bind to `process.env.PORT || 8080` (Railway HTTP port). TCP stays on `PARSER_PORT=5050`. Node built-in `http` — no express needed. Server started in `index.ts` after TCP server starts (Claude's Discretion on port).

---

### `packages/parser/src/index.ts` (entry-point, refactored)

**Analog:** `packages/parser/src/index.ts` lines 46-66 and 412-415 (class init + start).

**Entry point pattern** (lines 46-66, 412-415):
```typescript
class OBD2Parser {
  private server: net.Server;
  private readonly PORT: number;

  constructor() {
    this.PORT = parseInt(process.env.PARSER_PORT || '5050', 10);
    this.supabase = createClient(
      process.env.SUPABASE_URL || '',
      process.env.SUPABASE_SERVICE_ROLE_KEY || ''
    );
    this.server = net.createServer(this.handleConnection.bind(this));
  }

  start(): void {
    this.server.listen(this.PORT, () => {
      console.log(`OBD2 Parser (Codec 8 Extended) listening on port ${this.PORT}`);
    });
  }
}

const parser = new OBD2Parser();
parser.start();
```

**Notes for planner:** Refactored `index.ts` becomes thin wiring layer — instantiates `TelemetryWriter`, `DeviceAuth`, `ConnectionHandler`, starts TCP server and health HTTP server. Remove `OBD2Parser` class export — export nothing (or export for testing only). Replace `console.log` with `logger`. Add `import 'dotenv/config'` (already present at line 3).

---

### `packages/parser/jest.config.js` (config)

**Analog:** None — file does not exist.

**Pattern from RESEARCH.md Pitfall 4:**
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/tests/**/*.test.ts'],
  moduleFileExtensions: ['ts', 'js'],
};
```

**Notes for planner:** Place at `packages/parser/jest.config.js`. Requires `jest`, `ts-jest`, `@types/jest` installed as devDependencies. CommonJS output (`"module": "commonjs"` in tsconfig) is required for ts-jest to work without ESM transform config.

---

### `packages/parser/tests/protocol-parser.test.ts` (test)

**Analog:** No existing test files. Pattern from Jest + ts-jest conventions.

**Test structure pattern:**
```typescript
import { parseCodec8ExtendedPacket, crc16 } from '../src/protocol-parser';

describe('crc16', () => {
  it('returns correct CRC for known buffer', () => {
    const buf = Buffer.from([0x8E, 0x00, 0x01]);
    expect(crc16(buf)).toBe(/* expected value */);
  });
});

describe('parseCodec8ExtendedPacket', () => {
  it('extracts IO ID 7040 as temp', () => {
    const packet = buildTestPacket({ ioId: 7040, value: 900 }); // 90.0°C
    const result = parseCodec8ExtendedPacket(packet);
    expect(result.records[0].temp).toBe(90);
  });

  it('rejects packet with wrong CRC', () => {
    const packet = buildTestPacket({ corruptCrc: true });
    expect(() => parseCodec8ExtendedPacket(packet)).toThrow(/CRC/);
    // OR: expect(result.error).toMatch(/CRC/);
  });
});
```

**Notes for planner:** Use the actual simulator's `buildCodec8ExtendedPacket` logic (from `packages/simulator/src/index.ts` lines 167-242) as the test packet builder — it is verified to produce valid packets. Mock nothing — pure buffer transform. Covers INGEST-02 requirements: framing, CRC accept/reject, IO ID extraction for 7040/7044/7045/7059, partial buffer discard.

---

### `packages/parser/tests/telemetry-writer.test.ts` (test)

**Analog:** No existing test files.

**Mock pattern for Supabase:**
```typescript
// Mock @supabase/supabase-js
jest.mock('@supabase/supabase-js', () => ({
  createClient: jest.fn(() => ({
    from: jest.fn(() => ({
      insert: jest.fn().mockResolvedValue({ error: null }),
      select: jest.fn().mockReturnThis(),
      eq: jest.fn().mockReturnThis(),
      single: jest.fn().mockResolvedValue({ data: mockDevice, error: null }),
    })),
  })),
}));

describe('TelemetryWriter queue behavior', () => {
  it('enqueues record on Supabase failure', async () => { ... });
  it('discards oldest when queue is full', async () => { ... });
  it('alerts after 3 consecutive failures', async () => { ... });
  it('writes dtc_codes as empty array', async () => { ... });
});
```

**Notes for planner:** Covers INGEST-03 (INSERT fields) and INGEST-05 (queue D-09). Mock `pino` logger to avoid output noise in test runs: `jest.mock('../src/logger', () => ({ logger: { info: jest.fn(), warn: jest.fn(), error: jest.fn() } }))`.

---

### `packages/parser/tests/connection-handler.test.ts` (test)

**Analog:** No existing test files.

**Mock socket pattern:**
```typescript
import { EventEmitter } from 'events';

function mockSocket(): net.Socket {
  const s = new EventEmitter() as any;
  s.write = jest.fn();
  s.destroy = jest.fn();
  s.remoteAddress = '127.0.0.1';
  s.remotePort = 9999;
  return s;
}

describe('ConnectionHandler', () => {
  it('closes previous session on duplicate IMEI login (D-11)', () => {
    const handler = new ConnectionHandler(...);
    const socket1 = mockSocket();
    const socket2 = mockSocket();
    handler.onLogin('123456789012345', socket1);
    handler.onLogin('123456789012345', socket2);
    expect(socket1.destroy).toHaveBeenCalled();
  });
});
```

---

### `packages/simulator/src/index.ts` (service, event-driven — UPDATE)

**Analog:** Self — update in place.

**IO element build pattern** (lines 176-201) — UPDATE IDs per D-06:
```typescript
// EXISTING (WRONG — old IDs, update these 3 blocks):

// IO ID 67: Battery Voltage → REPLACE WITH IO ID 7059
const voltBuf = Buffer.alloc(4);
voltBuf.writeUInt16BE(67, 0);   // ← change to 7059
voltBuf[2] = 0x02;
voltBuf.writeUInt16BE(voltageMv, 3);

// IO ID 128: Engine Temperature → REPLACE WITH IO ID 7040
const tempBuf = Buffer.alloc(4);
tempBuf.writeUInt16BE(128, 0);  // ← change to 7040
tempBuf[2] = 0x02;
tempBuf.writeUInt16BE(tempX10, 3);

// IO ID 179: Engine RPM → REPLACE WITH IO ID 7044
const rpmBuf = Buffer.alloc(6);
rpmBuf.writeUInt16BE(179, 0);   // ← change to 7044
rpmBuf[2] = 0x03;
rpmBuf.writeUInt32BE(data.rpm, 3);

// IO ID 5: Speed → KEEP AS-IS (speed from AVL header is primary; IO 7045 optional)
// Add IO ID 7045 for speed if simulator should emit it:
// ioElements.push(Buffer.from([0x1B, 0x65, 0x02, 0x00, data.speed]));
// (7045 = 0x1B85, type 2 = uint16 BE)
```

**Reconnect test scenario pattern (D-11 / RESEARCH specifics — new):**
```typescript
// Add to GhostFleetSimulator — reconnect test mode
private async runReconnectTest(client: net.Socket, packetsPerSession: number): Promise<void> {
  // Send N packets
  for (let i = 0; i < packetsPerSession; i++) {
    const data = this.generateSimulatedData();
    const packet = this.buildCodec8ExtendedPacket(data);
    client.write(packet);
    await sleep(this.config.intervalMs);
  }
  // Force-close to simulate device disconnect
  client.destroy();
  console.log(`[reconnect-test] Session closed after ${packetsPerSession} packets. Reconnecting in 2s...`);
  await sleep(2000);
  // runSimulation() → reconnect cycle handles the rest
}
```

**Notes for planner:** Buffer allocation for IO IDs must use correct byte widths. IO ID 7059 = 0x1B97 (2 bytes), not 1-byte value — existing `voltBuf.writeUInt16BE(id, 0)` already writes 2-byte ID correctly, only the number changes. IO ID 7044 = 0x1B84, 7040 = 0x1B80, 7045 = 0x1B85 (verify hex values against decimal). Reconnect test may be a separate script or a `RECONNECT_TEST=true` env flag in the same file.

---

## Shared Patterns

### Supabase Client Initialization
**Source:** `packages/parser/src/index.ts` lines 54-57
**Apply to:** `telemetry-writer.ts`, `device-auth.ts`
```typescript
import { createClient } from '@supabase/supabase-js';
import 'dotenv/config';

const supabase = createClient(
  process.env.SUPABASE_URL || '',
  process.env.SUPABASE_SERVICE_ROLE_KEY || ''
);
```

### Pino Logger Import
**Source:** `packages/parser/src/logger.ts` (to be created)
**Apply to:** All four modules (`connection-handler.ts`, `device-auth.ts`, `protocol-parser.ts`, `telemetry-writer.ts`) and `index.ts`
```typescript
import { logger } from './logger';

// Replace console.log → logger.info
// Replace console.error → logger.error
// Replace console.warn → logger.warn
// All log calls use structured object first arg: logger.info({ imei }, 'message')
```

### TypeScript Import Pattern
**Source:** `packages/parser/src/index.ts` lines 1-3
**Apply to:** All new modules
```typescript
import * as net from 'net';
import { createClient } from '@supabase/supabase-js';
import 'dotenv/config';
// Path aliases: not used — flat src/ structure, use relative imports (e.g., './types', './logger')
```

### Error Handling Pattern
**Source:** `packages/parser/src/index.ts` lines 374-376
**Apply to:** `telemetry-writer.ts`, `device-auth.ts`
```typescript
try {
  // Supabase operation
} catch (err) {
  logger.error({ err, imei }, 'Failed to process telemetry');
  // Do NOT call process.exit() — log and continue (anti-pattern in RESEARCH.md)
}
```

### Jest Mock Pattern for Logger
**Source:** RESEARCH.md Pattern guidance
**Apply to:** All test files
```typescript
jest.mock('../src/logger', () => ({
  logger: { info: jest.fn(), warn: jest.fn(), error: jest.fn(), debug: jest.fn() }
}));
```

---

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| `packages/parser/src/health.ts` | utility | request-response | No HTTP server exists in parser package — new capability |
| `packages/parser/jest.config.js` | config | — | No Jest config exists anywhere in codebase |
| `packages/parser/tests/*.test.ts` | test | — | Zero test files exist in entire monorepo |

---

## IO ID Hex Reference (for simulator buffer writes)

| IO ID (decimal) | IO ID (hex) | Purpose |
|-----------------|-------------|---------|
| 7038 | 0x1B7E | DTC Count (parse, do not store) |
| 7040 | 0x1B80 | Coolant Temperature (÷10 = °C) |
| 7044 | 0x1B84 | Engine RPM (raw) |
| 7045 | 0x1B85 | Vehicle Speed (km/h) |
| 7059 | 0x1B93 | Control Module Voltage (÷1000 = V) |

**Verify:** `Buffer.writeUInt16BE(7040, 0)` writes `0x1B 0x80`. Confirm against `(7040).toString(16) === '1b80'`.

---

## Metadata

**Analog search scope:** `packages/parser/src/`, `packages/simulator/src/`, `packages/supabase/migrations/`
**Files scanned:** 11 source files
**Pattern extraction date:** 2026-05-11
