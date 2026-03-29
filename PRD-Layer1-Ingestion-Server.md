# VitalsDrive — Layer 1: TCP Ingestion Server
**Version:** 1.0  
**Status:** Draft  
**Last Updated:** March 2026  
**Layer Owner:** Backend Infrastructure  
**Parent Document:** VitalsDrive PRD (Main)

---

## 1. Executive Summary

**Component Name:** TCP Ingestion Server  
**Alternative Names:** Packet Parser, Protocol Decoder, Device Listener  
**Type:** Network Service (Node.js)  
**Hosting:** Railway.app  
**Port:** 5050 TCP

The TCP Ingestion Server is the entry point for all telemetry data in the VitalsDrive platform. It receives raw hex packets from 4G OBD2 devices over TCP/IP, parses the manufacturer-specific protocols, decodes the binary payloads into normalized JSON, and pushes clean records to Supabase for downstream consumption by the Angular dashboard.

This layer is intentionally decoupled from the hardware — it must handle malformed packets, protocol variations, and device disconnections gracefully while producing hardware-agnostic output. The Ghost Fleet simulator exercises this layer extensively before real hardware arrives.

**Key Responsibilities:**
- Accept and maintain long-lived TCP connections from OBD2 devices
- Parse login/handshake packets and send acknowledgments
- Decode data packets per device manufacturer protocol
- Normalize output to a hardware-agnostic JSON schema
- Push decoded telemetry to Supabase via REST API
- Handle device reconnection and session continuity

---

## 2. Functional Requirements

### 2.1 TCP Connection Management

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| F-2.1.1 | Accept incoming TCP connections on port 5050 | Critical | Railway exposes port; Node.js `net` server |
| F-2.1.2 | Support concurrent connections from up to 50 devices | High | MVP scale; design for 500 |
| F-2.1.3 | Maintain persistent connections (devices send data continuously) | Critical | 4G devices maintain long-lived TCP sessions |
| F-2.1.4 | Detect and handle device disconnection gracefully | Critical | Clean up resources; log disconnect events |
| F-2.1.5 | Support connection keep-alive to detect stale connections | Medium | TCP keep-alive probe every 60s |
| F-2.1.6 | Implement connection timeout (inactive devices after 5 min) | Medium | Remove stale sessions; devices may reconnect |

### 2.2 Protocol Parsing — Login Packet

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| F-2.2.1 | Receive and parse device login/handshake packet | Critical | First packet after TCP connect |
| F-2.2.2 | Extract device IMEI/identifier from login packet | Critical | Used for vehicle_id mapping in Supabase |
| F-2.2.3 | Send login acknowledgment response to device | Critical | Protocol-specific; must be correct or device disconnects |
| F-2.2.4 | Validate protocol version in login packet | Medium | Log warning if version mismatch |
| F-2.2.5 | Store session context (device_id, connect_time, last_seen) | High | In-memory Map; keyed by socket |

### 2.3 Protocol Parsing — Data Packet

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| F-2.3.1 | Parse incoming data packets after login | Critical | Continuous stream during vehicle operation |
| F-2.3.2 | Extract RPM from Byte 14 (0x0C) per Alibaba protocol | Critical | Core MVP metric |
| F-2.3.3 | Extract GPS coordinates (latitude, longitude) | Critical | For fleet map |
| F-2.3.4 | Extract battery voltage | Critical | For battery health monitoring |
| F-2.3.5 | Extract coolant temperature | Critical | For temperature alerts |
| F-2.3.6 | Extract speed (if available in protocol) | Medium | For fleet map velocity display |
| F-2.3.7 | Extract DTC codes (if available in protocol) | Medium | For alert system |
| F-2.3.8 | Validate packet checksum/CRC | Critical | Discard invalid packets; log error |
| F-2.3.9 | Handle packet framing (start/stop bytes: 0x78 0x78 ... 0x0D 0x0A) | Critical | Per Alibaba protocol |
| F-2.3.10 | Handle multi-packet bursts (device sends rapid succession) | Medium | Buffer and process sequentially |

### 2.4 Device Protocol Support

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| F-2.4.1 | Support Alibaba SinoTrack protocol (primary) | Critical | First device batch from Alibaba |
| F-2.4.2 | Support Micodus protocol (if hardware varies) | Medium | Secondary supplier |
| F-2.4.3 | Detect device manufacturer from login packet | High | Switch parser logic accordingly |
| F-2.4.4 | Log unsupported protocol types; do not crash | Medium | Alert on new protocol detection |
| F-2.4.5 | Document protocol variations for future extensibility | Low | ASCII protocol notes in codebase |

### 2.5 Data Output — Supabase Integration

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| F-2.5.1 | Push decoded JSON to Supabase REST API | Critical | POST to `telemetry_logs` table |
| F-2.5.2 | Include vehicle_id (mapped from device IMEI) | Critical | Lookup in Supabase `devices` table |
| F-2.5.3 | Include all telemetry fields (lat, lng, temp, voltage, rpm, etc.) | Critical | Per schema |
| F-2.5.4 | Include timestamp (server-side generated) | High | `TIMESTAMPTZ DEFAULT NOW()` |
| F-2.5.5 | Handle Supabase API failures gracefully | Critical | Retry with backoff; do not drop data |
| F-2.5.6 | Queue outgoing messages during Supabase outage | High | In-memory queue; bounded size |
| F-2.5.7 | Log all Supabase push success/failure | High | For debugging and monitoring |

### 2.6 Normalized JSON Output

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| F-2.6.1 | Output hardware-agnostic JSON regardless of source device | Critical | Key architectural principle |
| F-2.6.2 | Normalize units (°C for temp, V for voltage, RPM for engine) | Critical | Per main PRD spec |
| F-2.6.3 | Include raw packet hex in debug mode (for validation) | Low | Feature flag: `DEBUG_RAW_PACKETS` |
| F-2.6.4 | Include parser version in output (for debugging) | Medium | Field: `parser_version` |

---

## 3. Non-Functional Requirements

### 3.1 Performance

| ID | Requirement | Target | Notes |
|---|---|---|---|
| N-3.1.1 | Packet processing latency (receive → Supabase push) | < 100ms p95 | Excludes network transit |
| N-3.1.2 | Throughput capacity | 100 packets/second | 50 devices × 2 packets/sec average |
| N-3.1.3 | Concurrent connection capacity | 500 devices | Design for MVP + 10x headroom |
| N-3.1.4 | Memory footprint | < 256MB | Railway container limit |
| N-3.1.5 | CPU utilization | < 40% sustained | Node.js single-thread; adequate for I/O-bound |

### 3.2 Reliability

| ID | Requirement | Target | Notes |
|---|---|---|---|
| N-3.2.1 | Uptime SLA | 99.5% | Railway + Node.js resilience |
| N-3.2.2 | Data loss tolerance | 0.1% max | Acceptable during Supabase outage |
| N-3.2.3 | Graceful degradation | Yes | Queue data; alert; continue serving |
| N-3.2.4 | Automatic restart on crash | Yes | Railway health check + restart policy |
| N-3.2.5 | Connection recovery after network blip | Yes | Devices reconnect automatically |

### 3.3 Scalability

| ID | Requirement | Target | Notes |
|---|---|---|---|
| N-3.3.1 | Horizontal scaling readiness | Yes | Stateless design; connection state in memory |
| N-3.3.2 | Stateless packet processing | Yes | Each packet independent; no cross-session state |
| N-3.3.3 | Load balancing compatibility | Yes | TCP sticky sessions required OR shared state store |
| N-3.3.4 | Geographic distribution (future) | Yes | Multi-region Railway deployment |

### 3.4 Observability

| ID | Requirement | Target | Notes |
|---|---|---|---|
| N-3.4.1 | Structured JSON logging | Required | Pino logger |
| N-3.4.2 | Log levels: error, warn, info, debug | Required | Configurable via env |
| N-3.4.3 | Request tracing (packet → Supabase) | Medium | Correlation ID per packet |
| N-3.4.4 | Metrics export (Prometheus-compatible) | Medium | Railway native or custom |

---

## 4. Technical Design

### 4.1 Node.js net Module Usage

```javascript
// Server skeleton (pseudocode)
const net = require('net');

const server = net.createServer({
  highWaterMark: 64 * 1024,  // 64KB read buffer
  pauseOnConnect: false      // Do not pause; handle immediately
});

server.on('connection', (socket) => {
  // Assign unique session ID
  const sessionId = uuidv4();
  
  // Enable TCP keep-alive
  socket.setKeepAlive(true, 60000); // probe after 60s idle
  
  // Set timeout (5 min inactive)
  socket.setTimeout(300000);
  
  // Handle per-session context
  const session = {
    id: sessionId,
    deviceId: null,
    protocol: null,
    connectedAt: new Date(),
    lastPacketAt: null,
    socket: socket
  };
  
  sessionMap.set(sessionId, session);
  
  // Buffer for incomplete packets
  let buffer = Buffer.alloc(0);
  
  socket.on('data', (chunk) => {
    buffer = Buffer.concat([buffer, chunk]);
    processPackets(buffer, session, socket);
  });
  
  socket.on('close', () => handleDisconnect(sessionId));
  socket.on('error', (err) => handleError(sessionId, err));
  socket.on('timeout', () => handleTimeout(sessionId, socket));
});

server.listen(5050, '0.0.0.0', () => {
  logger.info({ port: 5050 }, 'TCP Ingestion Server started');
});
```

### 4.2 Packet Parsing Logic

#### 4.2.1 Alibaba Protocol — Packet Structure

| Offset | Field | Size | Type | Description |
|---|---|---|---|---|
| 0 | Start Byte 1 | 1 | uint8 | 0x78 |
| 1 | Start Byte 2 | 1 | uint8 | 0x78 |
| 2 | Length | 1 | uint8 | Packet length (including header) |
| 3 | Protocol | 1 | uint8 | 0x01=Login, 0x22=Data |
| 4..n | Payload | Variable | bytes | Protocol-dependent |
| n-2 | CRC | 2 | uint16 | Modbus CRC16 of bytes 2..n-3 |
| n | Stop Byte 1 | 1 | uint8 | 0x0D |
| n+1 | Stop Byte 2 | 1 | uint8 | 0x0A |

#### 4.2.2 Login Packet (Protocol 0x01) — Alibaba

```
Offset  Field           Size    Notes
0       Start           2       0x78 0x78
2       Length          1       Typically 0x19 (25)
3       Protocol        1       0x01 (login)
4..15   IMEI            15      ASCII string
16..17  Software Ver    2       e.g. "01"
18..19  Hardware Ver    2       e.g. "01"
20..21  CRC             2       CRC16
22      Stop            2       0x0D 0x0A
```

**Login ACK Response (server → device):**
```
0x78 0x78 0x05 0x81 0x00 [CRC2] 0x0D 0x0A
```
- 0x81 = login ack protocol number
- 0x00 = login result (0=success)

#### 4.2.3 Data Packet (Protocol 0x22) — Alibaba

```
Offset  Field           Size    Type        Notes
0       Start           2       0x78 0x78    -
2       Length          1       uint8       Total packet length
3       Protocol        1       0x22        Data packet
4..7     Latitude        4       int32       Degrees * 1,000,000
8..11    Longitude       4       int32       Degrees * 1,000,000
12       Speed           1       uint8       km/h
13       Direction       1       uint8       0-360 degrees
14       GPS Valid       1       uint8       0=invalid, 1=valid
15       GSM Signal      1       uint8       0-31
16       Battery Voltage 2       uint16      mV
17       External Voltage 2     uint16      mV (optional)
18       Temp            1       int8        °C (signed)
19       RPM High        1       uint8       RPM / 256
20       RPM Low         1       uint8       RPM % 256
21       DTC Count       1       uint8       Number of DTCs
22..n    DTC Codes       n       bytes       (if count > 0)
n-2     CRC             2       uint16      CRC16
n       Stop            2       0x0D 0x0A    -
```

**RPM Calculation:**
```
rpm = (rpmHigh * 256 + rpmLow) / 4
```

### 4.3 Protocol Handling — State Machine

```
                    ┌─────────────┐
                    │   CONNECTED  │
                    └──────┬──────┘
                           │ socket 'data'
                           ▼
                    ┌─────────────┐
              ┌────▶│  WAIT_LOGIN  │◀───────┐
              │     └──────┬──────┘        │
              │            │ recv login    │ timeout (30s)
              │            ▼               │ reconnect
              │     ┌─────────────┐        │
              │     │ LOGIN_RCVD  │─────────┘
              │     └──────┬──────┘
              │            │ send ack
              │            ▼
              │     ┌─────────────┐
              │     │  AUTHENTICATED │ (send ack success)
              │     └──────┬──────┘
              │            │ recv data packet
              │            ▼
              │     ┌─────────────┐
              └─────│  DATA_ACTIVE │──────┐
                    └─────────────┘      │
                           │             │ recv data
                           ▼             │
                    [Process Packet]──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  PUSH_TO_SUPABASE │
                    └─────────────┘
```

### 4.4 Error Handling & Reconnection Logic

#### 4.4.1 Malformed Packet Handling

```javascript
function processPackets(buffer, session, socket) {
  while (buffer.length >= 4) { // Minimum packet size
    // Find packet start (0x78 0x78)
    const startIdx = buffer.indexOf(0x78);
    if (startIdx === -1) {
      buffer = Buffer.alloc(0); // No start byte; discard
      break;
    }
    
    if (startIdx > 0) {
      logger.warn({ sessionId: session.id, dropped: startIdx }, 
        'Discarded bytes before packet start');
      buffer = buffer.slice(startIdx);
    }
    
    if (buffer.length < 4) break; // Insufficient for length byte
    
    const length = buffer[2];
    const totalPacketLen = length + 4; // start(2) + length(1) + data(length) + crc(2) + stop(2) - 1
    
    if (buffer.length < totalPacketLen) {
      break; // Wait for more data
    }
    
    const packet = buffer.slice(0, totalPacketLen);
    buffer = buffer.slice(totalPacketLen);
    
    try {
      const result = parsePacket(packet, session);
      if (result.type === 'login') {
        handleLogin(session, result.data, socket);
      } else if (result.type === 'data') {
        handleData(session, result.data);
      }
    } catch (err) {
      logger.error({ sessionId: session.id, error: err.message }, 
        'Packet parse error');
      // Continue processing next packet (do not crash)
    }
  }
}
```

#### 4.4.2 Device Reconnection Handling

- Devices may disconnect and reconnect at any time
- Session Map uses device IMEI as secondary key for deduplication
- If same IMEI reconnects before timeout, previous session is cleaned up
- Log "duplicate login" if IMEI already authenticated

#### 4.4.3 Supabase Failure Handling

```javascript
const supabaseQueue = {
  queue: [],
  maxSize: 1000,
  processing: false,
  retryDelay: 1000, // ms
  maxRetries: 3
};

async function pushToSupabase(telemetry) {
  supabaseQueue.queue.push({ data: telemetry, retries: 0 });
  if (!supabaseQueue.processing) {
    processQueue();
  }
}

async function processQueue() {
  supabaseQueue.processing = true;
  
  while (supabaseQueue.queue.length > 0) {
    const item = supabaseQueue.queue[0];
    
    try {
      await supabaseClient.from('telemetry_logs').insert(item.data);
      supabaseQueue.queue.shift();
    } catch (err) {
      if (item.retries >= supabaseQueue.maxRetries) {
        logger.error({ item }, 'Supabase push failed after max retries; dropping');
        supabaseQueue.queue.shift();
      } else {
        item.retries++;
        await sleep(supabaseQueue.retryDelay * item.retries);
      }
    }
  }
  
  supabaseQueue.processing = false;
}
```

### 4.5 Health Checks

| Check | Interval | Failure Threshold | Action |
|---|---|---|---|
| TCP port 5050 listening | 30s | 3 consecutive | Railway restarts container |
| Supabase API reachable | 60s | 3 consecutive | Alert + continue queuing |
| Active connections > 0 | 5min | N/A (info only) | Log connection count |
| Queue depth > 500 | 60s | 3 consecutive | Alert |
| Memory usage > 200MB | 60s | 2 consecutive | Alert |

Railway native health check:
```
GET /health → 200 OK { "status": "healthy", "connections": N, "queueDepth": M }
```

---

## 5. API/Interface Contracts

### 5.1 Supabase Input Schema — telemetry_logs

The ingestion server writes to Supabase via REST API (not direct Postgres connection).

**Endpoint:** `POST /rest/v1/telemetry_logs`

**Request Headers:**
```
apikey: {SUPABASE_SERVICE_ROLE_KEY}
Authorization: Bearer {SUPABASE_SERVICE_ROLE_KEY}
Content-Type: application/json
Prefer: return=minimal
```

**Request Body:**
```json
{
  "vehicle_id": "550e8400-e29b-41d4-a716-446655440000",
  "lat": 37.7749,
  "lng": -122.4194,
  "speed": 35.5,
  "temp": 92,
  "voltage": 12.6,
  "rpm": 1450,
  "dtc_codes": ["P0301", "P0420"],
  "gps_valid": true,
  "direction": 180,
  "gsm_signal": 25,
  "parser_version": "1.0.0",
  "device_imei": "861234567890123",
  "timestamp": "2026-03-29T14:32:00.123Z"
}
```

**Supabase Table Definition:**
```sql
CREATE TABLE telemetry_logs (
  id          BIGSERIAL PRIMARY KEY,
  vehicle_id  UUID NOT NULL REFERENCES vehicles(id),
  lat         DECIMAL(10, 7),
  lng         DECIMAL(10, 7),
  speed       DECIMAL(6, 2),
  temp        INTEGER,
  voltage     DECIMAL(5, 2),
  rpm         INTEGER,
  dtc_codes   TEXT[],
  gps_valid   BOOLEAN DEFAULT true,
  direction   INTEGER,
  gsm_signal  INTEGER,
  parser_version TEXT,
  device_imei VARCHAR(20),
  timestamp   TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_telemetry_logs_vehicle_id ON telemetry_logs(vehicle_id);
CREATE INDEX idx_telemetry_logs_timestamp ON telemetry_logs(timestamp DESC);
```

### 5.2 Device IMEI to Vehicle ID Mapping

**Endpoint:** `GET /rest/v1/devices?imei=eq.{imei}&select=id,vehicle_id`

```json
{
  "id": "device-uuid",
  "vehicle_id": "vehicle-uuid"
}
```

If device not found, insert into `unregistered_devices` table for alerting.

### 5.3 Internal Health Check Endpoint

**Endpoint:** `GET /health`

```json
{
  "status": "healthy",
  "uptime": 86400,
  "connections": {
    "active": 12,
    "total": 150
  },
  "queueDepth": 3,
  "lastSupabasePush": "2026-03-29T14:31:58Z",
  "version": "1.0.0"
}
```

**Non-Healthy Response (5xx):**
```json
{
  "status": "degraded",
  "issue": "supabase_unreachable",
  "queueDepth": 523,
  "connections": { "active": 8 }
}
```

---

## 6. Configuration Management

### 6.1 Environment Variables

| Variable | Type | Default | Required | Description |
|---|---|---|---|---|
| `PORT` | number | 5050 | Yes | TCP listen port |
| `SUPABASE_URL` | string | - | Yes | e.g. `https://xxx.supabase.co` |
| `SUPABASE_SERVICE_ROLE_KEY` | string | - | Yes | Service role key for writes |
| `LOG_LEVEL` | string | info | No | debug, info, warn, error |
| `CONNECTION_TIMEOUT_MS` | number | 300000 | No | 5 minutes |
| `KEEPALIVE_INTERVAL_MS` | number | 60000 | No | TCP keep-alive probe |
| `SUPABASE_QUEUE_MAX_SIZE` | number | 1000 | No | Max queued writes |
| `SUPABASE_RETRY_DELAY_MS` | number | 1000 | No | Initial retry delay |
| `SUPABASE_MAX_RETRIES` | number | 3 | No | Max push retries |
| `DEBUG_RAW_PACKETS` | boolean | false | No | Log hex of each packet |
| `MAX_PACKET_SIZE` | number | 1024 | No | Max bytes per packet |

### 6.2 Railway Configuration (railway.toml)

```toml
[build]
  builder = "nixpacks"
  builderVersion = "1"

[deploy]
  numReplicas = 1
  restartPolicyType = "ON_FAILURE"
  restartPolicyMaxRetries = 10
  healthCheckPath = "/health"

[[ports]]
  port = 5050
  protocol = "TCP"

[environment]
  PORT = "5050"
  NODE_ENV = "production"
```

### 6.3 Feature Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `DEBUG_RAW_PACKETS` | boolean | false | Log raw hex input (verbose) |
| `STRICT_PROTOCOL_VALIDATION` | boolean | true | Reject packets with invalid CRC |
| `ALLOW_UNREGISTERED_DEVICES` | boolean | false | Accept data from unknown IMEIs |
| `QUEUE_ON_SUPABASE_FAILURE` | boolean | true | Enable offline queuing |

---

## 7. Deployment Considerations (Railway)

### 7.1 Deployment Steps

1. **Connect Repository:** Link GitHub repo to Railway project
2. **Configure Build:** `npm install && npm start` (Node.js Dockerfile or nixpacks)
3. **Set Environment Variables:** SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, LOG_LEVEL
4. **Deploy:** Railway builds and deploys container
5. **Verify:** Check `/health` endpoint responds 200

### 7.2 Networking

- Railway assigns random public IPv4 at deploy
- Device SMS configuration: `adminip123456 [RAILWAY_PUBLIC_IP] 5050`
- For production: Consider Railway "Variables" for static IP or dedicated TCP proxy
- Domain: `ingestion.vitalsdrive.com` (optional CNAME)

### 7.3 Scaling

| Phase | Replicas | Notes |
|---|---|---|
| MVP/Pilot (1-10 vehicles) | 1 | Single instance sufficient |
| Growth (10-100 vehicles) | 1-2 | Single TCP connection per device; connection count is the limit |
| Scale (100-500 vehicles) | 2+ | Requires TCP sticky sessions or centralized session store (Redis) |

### 7.4 Cost

| Resource | MVP Cost | Notes |
|---|---|---|
| Railway Compute | $0-5/mo | Usage-based; ~$3/mo for 1 replica |
| TCP Port | Included | Railway includes TCP ports |
| Data Transfer | ~$0 | Minimal; Supabase egress separate |

### 7.5 Disaster Recovery

| Scenario | Recovery | RTO |
|---|---|---|
| Container crash | Railway auto-restarts | < 30s |
| Host failure | Railway migrates to new host | < 2min |
| Full region outage | Deploy to new region; SMS devices with new IP | < 30min |
| Supabase outage | Local queue; alert; retry on restore | Depends on Supabase |

---

## 8. Monitoring and Logging Strategy

### 8.1 Logging (Pino)

**Format:** JSON (machine-readable)

```json
{
  "level": "info",
  "time": 1743245520123,
  "pid": 1,
  "hostname": "container-abc123",
  "msg": "Packet processed",
  "sessionId": "sess-uuid",
  "deviceImei": "861234567890123",
  "packetType": "data",
  "parserMs": 2,
  "supabasePushMs": 45
}
```

### 8.2 Key Log Events

| Event | Level | Fields |
|---|---|---|
| Server started | info | port, version |
| Device connected | info | sessionId, remoteIp, deviceImei |
| Device disconnected | info | sessionId, reason, duration |
| Login packet received | debug | sessionId, imei, protocol |
| Login ack sent | debug | sessionId |
| Data packet received | debug | sessionId, packetLen |
| Packet parse error | error | sessionId, error, rawHex |
| CRC validation failed | warn | sessionId, expectedCrc, actualCrc |
| Supabase push success | debug | sessionId, telemetryId |
| Supabase push failure | error | sessionId, error, retries |
| Queue depth warning | warn | queueDepth |
| Unknown protocol | warn | sessionId, protocolByte |

### 8.3 Metrics to Track

| Metric | Type | Source | Alert Threshold |
|---|---|---|---|
| `tcp_connections_active` | gauge | Node.js | > 450 (90% capacity) |
| `tcp_connections_total` | counter | Node.js | N/A |
| `packets_processed_total` | counter | Node.js | N/A |
| `packets_parse_errors_total` | counter | Node.js | > 1% of total |
| `supabase_push_duration_ms` | histogram | Node.js | p95 > 500ms |
| `supabase_push_failures_total` | counter | Node.js | > 0 |
| `queue_depth` | gauge | Node.js | > 800 |
| `uptime_seconds` | gauge | Node.js | < 60 (restart detected) |

### 8.4 Alerting

| Alert | Condition | Severity | Action |
|---|---|---|---|
| Server down | /health returns 5xx | P1 | Railway auto-restart + alert |
| Supabase unreachable | 3 consecutive failures | P2 | Slack alert |
| Queue depth critical | > 900 for 5min | P2 | Slack alert |
| Parse error spike | > 5% error rate | P3 | Log review |
| Connection count high | > 450 | P3 | Capacity planning |

---

## 9. Security Considerations

### 9.1 Network Security

| Concern | Mitigation | Notes |
|---|---|---|
| Unauthorized device access | Devices must send valid login packet with registered IMEI | First line of defense |
| IMEI spoofing | Supabase lookup validates IMEI exists in `devices` table | Non-registered devices rejected |
| Man-in-the-middle | Railway uses TLS for all outbound; inbound is raw TCP | 4G device encryption handles transport |
| Port scanning | Railway provides network isolation; only 5050 exposed | Cloud provider responsibility |

### 9.2 Data Security

| Concern | Mitigation | Notes |
|---|---|---|
| Supabase credentials | Stored as Railway environment variables (encrypted at rest) | Never in code |
| Device IMEI in logs | Log IMEI only for active sessions; redact in slow queries | PII consideration |
| Raw packet data | `DEBUG_RAW_PACKETS` only enabled in dev/staging | Never production |
| SQL injection | Using Supabase JS client with parameterized queries | N/A (no direct SQL) |

### 9.3 Rate Limiting

- No per-device rate limiting in MVP (devices naturally throttled by hardware)
- Supabase free tier has built-in rate limiting (~60 requests/min)
- Consider adding Redis-based rate limiting if Supabase errors appear

### 9.4 Input Validation

```javascript
function validatePacket(buffer) {
  if (buffer.length < 4) throw new Error('Packet too short');
  if (buffer[0] !== 0x78 || buffer[1] !== 0x78) throw new Error('Invalid start bytes');
  
  const length = buffer[2];
  if (length > 255) throw new Error('Invalid length byte');
  if (buffer.length < length + 4) throw new Error('Incomplete packet');
  
  // Validate stop bytes
  const stopIdx = 2 + length;
  if (buffer[stopIdx - 1] !== 0x0D || buffer[stopIdx] !== 0x0A) {
    throw new Error('Invalid stop bytes');
  }
  
  return true;
}
```

---

## 10. Testing Strategy

### 10.1 Unit Tests

**Test Packet Parsing:**
- Login packet parsing (valid, truncated, corrupted)
- Data packet parsing (all field extractions)
- CRC validation (valid, invalid)
- Edge cases: empty buffer, partial packet, multi-packet buffer

**Test Normalization:**
- Unit conversion correctness (mV → V, raw RPM → actual RPM)
- Null/undefined field handling
- JSON output schema validation

**Test Supabase Integration:**
- Successful insert
- Network failure simulation
- Queue overflow handling
- Retry logic

**Framework:** Jest

```javascript
describe('Alibaba Protocol Parser', () => {
  test('parses valid login packet', () => {
    const packet = Buffer.from('7878150123456789012345678901230112010D0A');
    const result = parseLoginPacket(packet);
    expect(result.imei).toBe('123456789012345');
    expect(result.protocolVersion).toBe('01');
  });
  
  test('throws on invalid CRC', () => {
    const packet = Buffer.from('787815012345678901234567890123011201FF0A');
    expect(() => parseLoginPacket(packet)).toThrow('CRC mismatch');
  });
});
```

### 10.2 Integration Tests

**Test TCP Server:**
- Ghost Fleet client connects, sends login, receives ack
- Ghost Fleet client sends data packets, verifies Supabase insert
- Multiple simultaneous connections
- Graceful disconnect handling

**Test with Supabase:**
- Full round-trip: ghost script → parser → Supabase → verify row exists
- Concurrent writes from multiple simulated devices

### 10.3 Ghost Fleet Simulation

Primary integration test method. The Ghost Fleet script (see main PRD Section 9) is the reference implementation for packet generation. It must produce packets that the parser accepts.

**Validation Checklist:**
- [ ] Login packet: parser sends ack, no errors
- [ ] Data packets: parser extracts all fields correctly
- [ ] GPS coordinates: within valid range (-90 to 90, -180 to 180)
- [ ] RPM: within valid range (0 to 10000)
- [ ] Voltage: within valid range (0 to 30V)
- [ ] Temperature: within valid range (-40 to 150°C)
- [ ] Supabase row: all fields match ghost script input

### 10.4 Performance Tests

| Test | Method | Target |
|---|---|---|
| Connection churn | Rapid connect/disconnect (100/sec) | No memory leaks |
| Packet burst | 1000 packets sent rapidly | Buffer handling correct |
| Sustained load | 50 devices × 5 packets/sec for 1 hour | < 100ms p95 latency |
| Memory leak | Monitor heap over 24h | Stable memory usage |

### 10.5 Chaos Testing

| Scenario | Expected Behavior |
|---|---|
| Supabase API returns 500 | Queue fills, retries, alert fired |
| Network interruption (5s) | Devices reconnect, queue drains |
| Malformed packet flood | Invalid packets logged, valid packets continue |
| Server restart mid-session | Device reconnects, new session created |

---

## 11. Open Questions and Risks

### 11.1 Open Questions

| # | Question | Impact | Owner | Status |
|---|---|---|---|---|
| O-1 | **Protocol document verification:** Has the SinoTrack/Micodus protocol spec been received from the Alibaba supplier? | Critical | Hardware | Pending |
| O-2 | **SMS command format:** What is the exact SMS syntax to configure the device's server IP and port? | Critical | Hardware | Pending |
| O-3 | **Login packet timing:** Does the device wait for ACK before sending data packets? | High | Protocol | Unknown |
| O-4 | **Multi-device authentication:** Can the same IMEI connect from multiple sockets simultaneously? | Medium | Protocol | Unknown |
| O-5 | **GPS cold start behavior:** How does the device behave when GPS signal is lost (e.g., underground garage)? | Medium | Behavior | Unknown |
| O-6 | **DTC extraction:** Is DTC data included in the data packet, or does it require a separate request? | Medium | Protocol | Unknown |
| O-7 | **Battery voltage offset:** Is the voltage field raw ADC or already calibrated (mV)? | Medium | Calibration | Unknown |
| O-8 | **RPM formula confirmation:** Is RPM = (high*256 + low) / 4 correct for this device? | Critical | Protocol | Pending |

### 11.2 Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **R-1:** Protocol mismatch between ghost script and real device | Medium | High | Verify protocol doc before Phase 3; adjust parser incrementally |
| **R-2:** Device sends packets before TCP connection fully established | Low | Medium | Buffer handling in parser; test with real device ASAP |
| **R-3:** Supabase rate limiting at scale | Low | Medium | Monitor API usage; upgrade tier or add Redis caching |
| **R-4:** Railway IPv4 changes on redeploy (devices lose IP) | Low | High | Use Railway static IP addon for production; SMS update script for devices |
| **R-5:** Memory leak from session Map not cleaning up | Medium | High | Unit test + 24h stress test before Phase 4 |
| **R-6:** CRC algorithm mismatch (device uses different CRC variant) | Low | Critical | Request protocol spec; include multiple CRC implementations; log failures for analysis |
| **R-7:** Device firmware update changes protocol | Low | High | Pin firmware version; log protocol version on login; alert on new version |
| **R-8:** Multiple devices behind same carrier NAT (connection deduplication) | Low | Medium | Use IMEI as primary identifier; do not rely on IP |

### 11.3 Dependencies

| Dependency | Owner | Due Date | Blocker |
|---|---|---|---|
| Alibaba protocol specification | Hardware team | Before Phase 3 | Parser finalization |
| SinoTrack device (1 unit for testing) | Hardware team | Before Phase 3 | Real device testing |
| Supabase project created | Backend | Week 1 | Deployment |
| Railway project configured | Backend | Week 1 | Deployment |
| Telnyx/Monogoto SIM activated | Hardware team | Before Phase 3 | Live device testing |

---

## 12. Glossary

| Term | Definition |
|---|---|
| **CRC** | Cyclic Redundancy Check — error detection code used in protocol packets |
| **DTC** | Diagnostic Trouble Code — OBD2 fault codes (e.g., P0301 misfire) |
| **Ghost Fleet** | Simulation of device traffic using a local script before real hardware arrives |
| **IMEI** | International Mobile Equipment Identity — unique 15-digit device identifier |
| **Ingestion Server** | Layer 1 component that receives raw device data and normalizes it |
| **Net module** | Node.js built-in module for TCP server implementation |
| **Pino** | Node.js structured JSON logger (pinojs.github.io) |
| **Railway** | Cloud platform for deploying Node.js services (railway.app) |
| **Supabase** | PostgreSQL + Realtime backend-as-a-service (supabase.com) |
| **Telemetry** | Vehicle diagnostic data (RPM, voltage, GPS, temperature, etc.) |

---

## 13. Document History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | March 2026 | VitalsDrive Product | Initial draft |

---

*Layer Owner: Backend Infrastructure  
Next Review: End of Phase 1 (Week 1 of development)*
