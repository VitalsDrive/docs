---
phase: 01-02-simulator
reviewed: 2026-05-12T00:00:00Z
depth: standard
files_reviewed: 1
files_reviewed_list:
  - packages/simulator/src/index.ts
findings:
  critical: 4
  warning: 3
  info: 2
  total: 9
status: issues_found
---

# Phase 01-02: Simulator Code Review

**Reviewed:** 2026-05-12
**Depth:** standard
**Files Reviewed:** 1
**Status:** issues_found

## Summary

Reviewed `packages/simulator/src/index.ts` — the GhostFleetSimulator class that emits Teltonika FMC003 Codec 8 Extended packets over TCP.

Four critical bugs found: the CRC algorithm is wrong (CCITT not IBM), the Data Field Length header includes the CRC bytes it must not, the Codec 8 Extended record count fields are encoded as uint16 instead of uint8 (corrupting framing), and `speedInt` is computed but never written — the speed IO element sends raw km/h rather than the km/h×10 the variable implies. Together these four issues mean every packet sent is structurally malformed and will be rejected or misparsed by a spec-compliant parser.

Three warnings: RECONNECT_TEST exit counts connection attempts not successful data sessions (a transient error ends the test early), a timeout-vs-close race can trigger duplicate `runSimulation()` calls, and integer inputs from environment variables are not validated (NaN propagates silently).

---

## Critical Issues

### CR-01: Wrong CRC Polynomial — CCITT Used Instead of IBM

**File:** `packages/simulator/src/index.ts:26-39`

**Issue:** The comment says "CRC-16-IBM" but the polynomial used is `0x1021`, which is CRC-16/CCITT (also known as CRC-CCITT). Teltonika Codec 8 Extended specifies CRC-16/IBM: polynomial `0x8005`, initial value `0x0000`, reflected input and output (LSB-first bit processing). Using `0x1021` with no reflection produces a completely different checksum. Any parser implementing the Teltonika spec will reject every packet.

**Fix:**
```typescript
function crc16(data: Buffer): number {
  let crc = 0x0000;
  for (let i = 0; i < data.length; i++) {
    // Reflect input byte (LSB-first)
    let byte = data[i];
    byte = ((byte & 0xF0) >> 4) | ((byte & 0x0F) << 4);
    byte = ((byte & 0xCC) >> 2) | ((byte & 0x33) << 2);
    byte = ((byte & 0xAA) >> 1) | ((byte & 0x55) << 1);
    crc ^= byte << 8;
    for (let j = 0; j < 8; j++) {
      if (crc & 0x8000) {
        crc = (crc << 1) ^ 0x8005;
      } else {
        crc = crc << 1;
      }
    }
  }
  // Reflect output
  let out = crc & 0xffff;
  out = ((out & 0xFF00) >> 8) | ((out & 0x00FF) << 8);
  out = ((out & 0xF0F0) >> 4) | ((out & 0x0F0F) << 4);
  out = ((out & 0xCCCC) >> 2) | ((out & 0x3333) << 2);
  out = ((out & 0xAAAA) >> 1) | ((out & 0x5555) << 1);
  return out;
}
```

Alternatively, use the `crc` npm package: `crc.crc16ibm(data)`.

---

### CR-02: Data Field Length Header Incorrectly Includes CRC Bytes

**File:** `packages/simulator/src/index.ts:277`

**Issue:** Teltonika framing is:
```
[Preamble 4B][Data Field Length 4B][Data Field ...][CRC 4B]
```
"Data Field" runs from Codec ID through the final record count byte. The CRC is *outside* the length. The code computes:

```typescript
const dataLength = payload.length + 2; // payload + CRC  ← WRONG
```

This inflates the declared length by 2, causing any parser that uses the length field to read 2 extra bytes into the CRC region, corrupting framing for all subsequent data.

**Fix:**
```typescript
const dataLength = payload.length; // CRC is outside the length field
```

---

### CR-03: Record Count Fields Encoded as uint16 Instead of uint8 (Codec 8 Extended Framing Corruption)

**File:** `packages/simulator/src/index.ts:261-266`

**Issue:** Codec 8 Extended frame layout:

```
[Codec ID: 1B][Number of Data 1: 1B][AVL Records...][Number of Data 2: 1B]
```

Both `Number of Data` fields are **1-byte uint8**. The code allocates a 3-byte header and writes `num_records` as uint16BE at offset 1, making the header 3 bytes with num_records occupying bytes 1–2. Similarly the trailer is 2 bytes with uint16BE. This adds 2 extra bytes to the frame and misaligns every field that follows.

```typescript
// Current — wrong sizes
const header = Buffer.alloc(3);
header[0] = 0x8e;
header.writeUInt16BE(1, 1);   // 2 bytes for a 1-byte field

const trailer = Buffer.alloc(2);
trailer.writeUInt16BE(1, 0);  // 2 bytes for a 1-byte field
```

**Fix:**
```typescript
const header = Buffer.alloc(2);
header[0] = 0x8e;       // codec ID
header[1] = 0x01;       // number of data (uint8)

const trailer = Buffer.alloc(1);
trailer[0] = 0x01;      // number of data repeat (uint8)
```

---

### CR-04: `speedInt` Computed But Never Written — IO Element Sends Raw km/h

**File:** `packages/simulator/src/index.ts:207,218`

**Issue:** `speedInt` is calculated as `data.speed * 10` (speed scaled ×10), but the speed IO buffer writes `data.speed` (unscaled):

```typescript
const speedInt = data.speed * 10; // computed, never used
// ...
speedBuf.writeUInt16BE(data.speed, 3); // ← should be speedInt
```

If the parser/spec expects km/h×10 (matching the scale implied by the variable name and the analogous `tempX10`/`voltageMv` conversions), every speed reading is off by a factor of 10. If km/h is correct (no scaling), then `speedInt` is dead code and the comment "km/h × 10" on line 207 is misleading. Either way, the code is inconsistent and almost certainly wrong given the pattern of the other IO elements.

**Fix (if spec requires km/h×10):**
```typescript
speedBuf.writeUInt16BE(speedInt, 3); // use the scaled value
```

**Fix (if spec requires raw km/h):** Remove the `speedInt` variable and the misleading comment.

---

## Warnings

### WR-01: RECONNECT_TEST Counts Connection Attempts, Not Successful Data Sessions

**File:** `packages/simulator/src/index.ts:42,117`

**Issue:** `sessionCount` is a module-level variable incremented at the top of `runSimulation()` (line 69), which is also called from the `error` handler (line 133). In RECONNECT_TEST mode, if a transient TCP connection error occurs before auth completes, `runSimulation()` is called, `sessionCount` increments, and the exit condition `sessionCount >= 2` can trigger after only one successful data session (or zero). The test silently exits reporting a wrong packet count.

**Fix:** Only increment `sessionCount` after authentication succeeds, or use a separate `successfulSessions` counter incremented in the auth handler:

```typescript
// In the 'data' handler, after this.authenticated = true:
successfulSessions++;
```

And check `successfulSessions >= 2` in the `close` handler.

---

### WR-02: Scheduled Timeout Can Fire After Socket Destroy — Triggers Spurious Reconnect

**File:** `packages/simulator/src/index.ts:137-159`

**Issue:** `scheduleNext` posts a `setTimeout` at `intervalMs`. If the socket is destroyed (line 154) within one interval, the already-queued timeout from the *previous* call to `scheduleNext` will still fire. At that point `this.authenticated` is `false` (reset by `close` event), so the guard at line 141 prevents writing to the socket. However, the guard does NOT prevent `scheduleNext` from being called again recursively at the bottom (line 158) when a *new* session is already running — because `this.authenticated` will be `true` for the new session. This creates a second scheduling loop on top of the one started by `waitForAuth` in the new session, doubling the send rate.

**Fix:** Bind `scheduleNext` to a per-session token or cancel flag:

```typescript
let sessionActive = true;
client.on('close', () => {
  sessionActive = false;
  // ...
});

const scheduleNext = () => {
  if (!this.running || !this.authenticated || !sessionActive) return;
  setTimeout(() => {
    if (!this.running || !this.authenticated || !sessionActive) return;
    // ... send packet ...
    scheduleNext();
  }, intervalMs);
};
```

---

### WR-03: Environment Variable Integers Not Validated — NaN Propagates Silently

**File:** `packages/simulator/src/index.ts:288-295`

**Issue:** `parseInt` and `parseFloat` return `NaN` for invalid input. `NaN` is then stored in `config.port`, `config.intervalMs`, `config.latitude`, `config.longitude`, and `config.packetsPerSession`. Downstream:

- `client.connect(NaN, ...)` connects to port `NaN` — Node.js will emit an error, but no clear message is shown about the root cause.
- `setTimeout(..., NaN)` fires immediately (treated as 0ms), causing a tight reconnect loop that floods the console.
- `packetsPerSession` as `NaN` makes `packetsSentThisSession >= NaN` always `false`, so RECONNECT_TEST never closes the session.

**Fix:**
```typescript
function requireInt(val: string | undefined, def: number, name: string): number {
  const n = parseInt(val ?? String(def), 10);
  if (isNaN(n)) throw new Error(`Invalid env var ${name}: "${val}"`);
  return n;
}
// Apply to port, intervalMs, packetsPerSession
```

---

## Info

### IN-01: AVL Fixed Block Comment Says 28 Bytes But Allocation Is 29 Bytes

**File:** `packages/simulator/src/index.ts:246`

**Issue:** Comment reads "Fixed AVL record (28 bytes)" but `Buffer.alloc(29)` is correct (8+1+4+4+2+2+2+2+2+2 = 29). Misleading comment.

**Fix:** Update comment to "Fixed AVL record header (29 bytes)".

---

### IN-02: `dotenv.config()` Called Before Any Validation or Module Imports That Need Env Vars

**File:** `packages/simulator/src/index.ts:1-2`

**Issue:** `dotenv.config()` is called before imports, which is the correct pattern for ensuring env vars are present before use. However, there is no check for whether the `.env` file was found (dotenv returns `{ parsed, error }`). If the file is missing, values silently fall back to hardcoded defaults with no warning. This is acceptable for a simulator but worth noting for operational clarity.

**Fix (optional):**
```typescript
const result = dotenv.config();
if (result.error) {
  console.warn('[config] No .env file found, using defaults / environment variables');
}
```

---

_Reviewed: 2026-05-12_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
