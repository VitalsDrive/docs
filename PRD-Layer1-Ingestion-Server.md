# VitalsDrive — Layer 1: TCP Ingestion Server
**Version:** 1.0
**Status:** Draft
**Last Updated:** March 2026
**Layer Owner:** Backend Infrastructure
**Parent Document:** VitalsDrive PRD (Main)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Technical Design](#4-technical-design)
5. [API/Interface Contracts](#5-apiinterface-contracts)
6. [Configuration](#6-configuration)
7. [Monitoring](#7-monitoring)
8. [Security](#8-security)
9. [Testing Strategy](#9-testing-strategy)
10. [Open Questions and Risks](#10-open-questions-and-risks)

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
| F-2.3.9 | Handle packet framing (0x00000000 preamble, length, CRC-16) | Critical | Per Teltonika Codec 8 Extended |
| F-2.3.10 | Handle multi-packet bursts (device sends rapid succession) | Medium | Buffer and process sequentially |

### 2.4 Device Protocol Support

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| F-2.4.1 | Support Teltonika Codec 8 Extended protocol (primary) | Critical | Teltonika FMC003 device |
| F-2.4.2 | Support additional Teltonika codecs if hardware varies | Medium | Codec 8 (non-Extended) fallback |
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

### 4.1 Server Architecture

The server uses Node.js `net` module for TCP socket management. It maintains an in-memory session map keyed by socket ID and tracks connected device state (IMEI, protocol type, connection time).

**Session Lifecycle:**
1. Device connects → assign session ID, enable TCP keep-alive (60s probe), set timeout (5 min inactive)
2. Device sends login packet → validate IMEI, send ACK, mark session as authenticated
3. Device sends data packets → parse, validate, normalize, push to Supabase
4. Device disconnects → clean up session, log event

### 4.2 Packet Parsing Requirements — Teltonika Codec 8 Extended

The server must implement a state machine per connection:

```
CONNECTED → WAIT_LOGIN → AUTHENTICATED → DATA_ACTIVE
             │              │                    │
         recv IMEI      send 0x01 ACK      recv 0x8E packets
```

**Handshake:** Device sends IMEI as raw 15-digit ASCII bytes. Server responds with single byte `0x01`. Device then sends AVL data packets.

**Packet Framing — Codec 8 Extended:**

| Offset | Field | Size | Notes |
|---|---|---|---|
| 0..3 | Preamble | 4 | `0x00 0x00 0x00 0x00` |
| 4..7 | Data length | 4 | uint32 big-endian, payload size |
| 8 | Codec ID | 1 | `0x8E` (Codec 8 Extended) |
| 9..10 | Num records | 2 | uint16 big-endian |
| ... | AVL data records | variable | See record layout below |
| N..N+1 | Num records | 2 | Repeated (uint16 BE) |
| N+2..N+3 | CRC-16 | 2 | CRC from codec ID through second num records |

**AVL Data Record Layout:**

| Offset | Field | Size | Type | Notes |
|---|---|---|---|---|
| 0..7 | Timestamp | 8 | uint64 BE | Milliseconds since epoch |
| 8 | Priority | 1 | uint8 | Event priority |
| 9..12 | Longitude | 4 | int32 BE | Degrees × 10^7 |
| 13..16 | Latitude | 4 | int32 BE | Degrees × 10^7 |
| 17..18 | Altitude | 2 | int16 BE | Meters |
| 19..20 | Angle | 2 | uint16 BE | Heading 0-360 |
| 21..22 | Satellites | 2 | uint16 BE | GPS satellite count |
| 23..24 | Speed | 2 | uint16 BE | km/h × 10 |
| 25..26 | Event ID | 2 | uint16 BE | Event identifier |
| 27..28 | IO element count | 2 | uint16 BE | Number of IO elements |
| 29..n | IO Elements | variable | See IO element layout below |

**IO Element Layout:**

| Field | Size | Notes |
|---|---|---|
| IO ID | 2 | uint16 BE, element identifier |
| IO Type | 1 | uint8: 1=8-bit, 2=16-bit, 3=32-bit, 4=64-bit, 5=float, 6=double |
| IO Value | 1/2/4/8 | Variable length based on type, big-endian |

**Required IO Elements for MVP:**

| IO ID | Name | Type | Unit | Notes |
|---|---|---|---|---|
| 5 | Speed | 1 (uint8) | km/h | Redundant with AVL speed field |
| 67 | Battery Voltage | 2 (uint16 BE) | mV | Divide by 1000 for volts |
| 128 | Engine Temperature | 2 (uint16 BE) | °C×10 | Divide by 10 for °C |
| 179 | Engine RPM | 3 (uint32 BE) | RPM | Raw RPM value |

**Server ACK (after each packet):** Send 4-byte big-endian record count confirming receipt. Example: `0x00000001` for 1 record.

**CRC-16 Calculation:** CRC-16-IBM (polynomial 0x1021, initial value 0x0000) computed over bytes from codec ID (offset 8) through the second num records field. Reject packets with CRC mismatch.

**RPM:** Read directly from IO ID 179 (uint32 BE).

**Voltage:** Read from IO ID 67 (uint16 BE, mV) — divide by 1000 for display.

**Temperature:** Read from IO ID 128 (uint16 BE, °C×10) — divide by 10 for °C.

### 4.3 Error Handling Strategy

| Scenario | Response |
|---|---|
| Malformed packet (invalid start/stop bytes) | Discard invalid bytes, log error, continue processing |
| CRC mismatch | Log warning, discard packet, continue |
| Supabase unreachable | Queue messages in-memory (bounded, default 1000), retry with exponential backoff, alert after 3 consecutive failures |
| Device disconnect | Clean up session from Map, log disconnect event |
| Memory pressure | Alert at >200MB, restart at >256MB |
| Duplicate IMEI login | Clean up previous session, log "duplicate login" |

### 4.4 Health Checks

| Check | Interval | Failure Action |
|---|---|---|
| TCP port 5050 listening | 30s | Railway auto-restart |
| Supabase API reachable | 60s | Alert + continue queuing |
| Queue depth | 60s | Alert if > 500 |
| Memory usage | 60s | Alert if > 200MB |

**Health endpoint:** `GET /health` returns status, active connections, queue depth, and last Supabase push timestamp. Returns 5xx when degraded to trigger Railway restart.

---

## 5. API/Interface Contracts

### 5.1 Normalized JSON Output Schema

The parser writes to Supabase via REST API (`POST /rest/v1/telemetry_logs`):

| Field | Type | Source | Description |
|---|---|---|---|
| vehicle_id | UUID | Supabase lookup | Mapped from device IMEI |
| lat | decimal(10,7) | Packet bytes 4-7 | Latitude |
| lng | decimal(10,7) | Packet bytes 8-11 | Longitude |
| speed | decimal(6,2) | Packet byte 12 | km/h |
| temp | integer | Packet byte 18 | °C coolant temperature |
| voltage | decimal(5,2) | Packet bytes 16-17 | Battery voltage (V) |
| rpm | integer | Packet bytes 19-20 | Calculated RPM |
| dtc_codes | text[] | Packet bytes 22+ | Active DTC codes |
| gps_valid | boolean | Packet byte 14 | GPS signal validity |
| direction | integer | Packet byte 13 | 0-360 degrees |
| gsm_signal | integer | Packet byte 15 | 0-31 |
| parser_version | text | Server config | Parser version for debugging |
| device_imei | varchar(20) | Login packet | Device identifier |
| timestamp | timestamptz | Server-generated | Ingestion time |

### 5.2 IMEI to Vehicle ID Mapping

The parser queries `GET /rest/v1/devices?imei=eq.{imei}` to map device IMEI to vehicle UUID. If the device is not found, the packet is rejected and logged.

### 5.3 Health Check Endpoint

`GET /health` returns 200 OK with status, active connection count, queue depth, and last successful Supabase push timestamp. Returns 5xx when the server is degraded.

---

## 6. Configuration

### 6.1 Environment Variables

| Variable | Type | Default | Required | Description |
|---|---|---|---|---|
| `PORT` | number | 5050 | Yes | TCP listen port |
| `SUPABASE_URL` | string | - | Yes | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | string | - | Yes | Service role key for writes |
| `LOG_LEVEL` | string | info | No | debug, info, warn, error |
| `CONNECTION_TIMEOUT_MS` | number | 300000 | No | Inactive session timeout |
| `KEEPALIVE_INTERVAL_MS` | number | 60000 | No | TCP keep-alive probe interval |
| `SUPABASE_QUEUE_MAX_SIZE` | number | 1000 | No | Max queued writes during outage |
| `DEBUG_RAW_PACKETS` | boolean | false | No | Log raw hex input |

### 6.2 Feature Flags

| Flag | Default | Description |
|---|---|---|
| `DEBUG_RAW_PACKETS` | false | Log raw hex input (verbose) |
| `STRICT_PROTOCOL_VALIDATION` | true | Reject packets with invalid CRC |
| `ALLOW_UNREGISTERED_DEVICES` | false | Accept data from unknown IMEIs |
| `QUEUE_ON_SUPABASE_FAILURE` | true | Enable offline queuing |

### 6.3 Deployment Configuration

Deployed on Railway using nixpacks builder. TCP port 5050 exposed. Health check path: `/health`. Restart policy: ON_FAILURE with max 10 retries.

---

## 7. Monitoring

### 7.1 Logging

Structured JSON logging (Pino). Key events: server start, device connect/disconnect, login received, data received, parse errors, CRC failures, Supabase push results, queue depth warnings, unknown protocol detection.

### 7.2 Key Metrics

| Metric | Type | Alert Threshold |
|---|---|---|
| `tcp_connections_active` | gauge | > 450 |
| `packets_parse_errors_total` | counter | > 1% of total |
| `supabase_push_duration_ms` | histogram | p95 > 500ms |
| `supabase_push_failures_total` | counter | > 0 |
| `queue_depth` | gauge | > 800 |

### 7.3 Alerting

| Alert | Condition | Severity |
|---|---|---|
| Server down | /health returns 5xx | P1 |
| Supabase unreachable | 3 consecutive failures | P2 |
| Queue depth critical | > 900 for 5min | P2 |
| Parse error spike | > 5% error rate | P3 |

---

## 8. Security

| Concern | Mitigation |
|---|---|
| Unauthorized device access | Valid login packet with registered IMEI required |
| IMEI spoofing | Supabase lookup validates IMEI exists |
| Supabase credentials | Railway encrypted env vars, never in code |
| Raw packet data | `DEBUG_RAW_PACKETS` disabled in production |

---

## 9. Testing Strategy

### 9.1 Unit Tests

**Packet Parsing:** Login parsing (valid, truncated, corrupted), data field extraction, CRC validation, edge cases (empty buffer, partial packet, multi-packet burst).

**Normalization:** Unit conversion correctness, null handling, JSON schema validation.

**Supabase Integration:** Successful insert, network failure simulation, queue overflow, retry logic.

**Framework:** Jest

### 9.2 Integration Tests

- Ghost Fleet client connects, sends login, receives ack
- Ghost Fleet client sends data packets, verifies Supabase insert
- Multiple simultaneous connections
- Graceful disconnect handling
- Full round-trip: ghost script → parser → Supabase → verify row

### 9.3 Validation Checklist

- [ ] Login packet: parser sends ack, no errors
- [ ] Data packets: parser extracts all fields correctly
- [ ] GPS coordinates: within valid range (-90 to 90, -180 to 180)
- [ ] RPM: within valid range (0 to 10000)
- [ ] Voltage: within valid range (0 to 30V)
- [ ] Temperature: within valid range (-40 to 150°C)
- [ ] Supabase row: all fields match ghost script input

### 9.4 Performance Tests

| Test | Target |
|---|---|
| Connection churn (100/sec) | No memory leaks |
| Packet burst (1000 rapid) | Buffer handling correct |
| Sustained load (50 devices × 5 pps, 1 hour) | < 100ms p95 latency |
| Memory stability (24h) | No growth |

### 9.5 Chaos Testing

| Scenario | Expected Behavior |
|---|---|
| Supabase returns 500 | Queue fills, retries, alert fired |
| Network interruption (5s) | Devices reconnect, queue drains |
| Malformed packet flood | Invalid packets logged, valid packets continue |
| Server restart mid-session | Device reconnects, new session created |

---

## 10. Open Questions and Risks

### 10.1 Open Questions

| # | Question | Impact | Owner | Status |
|---|---|---|---|---|
| O-1 | Protocol document verification: Has the Teltonika Codec 8 Extended spec been validated against real FMC003 device? | Critical | Hardware | Pending |
| O-2 | SMS command format: What is the exact SMS syntax to configure the device's server IP and port? | Critical | Hardware | Pending |
| O-3 | Login packet timing: Does the device wait for ACK before sending data packets? | High | Protocol | Unknown |
| O-4 | Multi-device authentication: Can the same IMEI connect from multiple sockets simultaneously? | Medium | Protocol | Unknown |
| O-5 | GPS cold start behavior: How does the device behave when GPS signal is lost? | Medium | Behavior | Unknown |
| O-6 | DTC extraction: Is DTC data included in the data packet, or does it require a separate request? | Medium | Protocol | Unknown |
| O-7 | Battery voltage offset: Is the voltage field raw ADC or already calibrated (mV)? | Medium | Calibration | Unknown |
| O-8 | RPM formula confirmation: Is RPM = (high*256 + low) / 4 correct for this device? | Critical | Protocol | Pending |

### 10.2 Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Protocol mismatch between ghost script and real device | Medium | High | Verify protocol doc before Phase 3; adjust parser incrementally |
| Supabase rate limiting at scale | Low | Medium | Monitor API usage; upgrade tier or add Redis caching |
| Railway IPv4 changes on redeploy (devices lose IP) | Low | High | Use Railway static IP addon for production |
| Memory leak from session Map not cleaning up | Medium | High | Unit test + 24h stress test before Phase 4 |
| CRC algorithm mismatch (device uses different CRC variant) | Low | Critical | Request protocol spec; include multiple CRC implementations |

### 10.3 Dependencies

| Dependency | Owner | Due Date | Blocker |
|---|---|---|---|
| Alibaba protocol specification | Hardware team | Before Phase 3 | Parser finalization |
| Teltonika FMC003 device (1 unit for testing) | Hardware team | Before Phase 3 | Real device testing |
| Supabase project created | Backend | Week 1 | Deployment |
| Railway project configured | Backend | Week 1 | Deployment |

---

## 11. Glossary

| Term | Definition |
|---|---|
| **CRC** | Cyclic Redundancy Check — error detection code used in protocol packets |
| **DTC** | Diagnostic Trouble Code — OBD2 fault codes (e.g., P0301 misfire) |
| **Ghost Fleet** | Simulation of device traffic using a local script before real hardware arrives |
| **IMEI** | International Mobile Equipment Identity — unique 15-digit device identifier |
| **Ingestion Server** | Layer 1 component that receives raw device data and normalizes it |
| **Pino** | Node.js structured JSON logger |
| **Railway** | Cloud platform for deploying Node.js services |
| **Supabase** | PostgreSQL + Realtime backend-as-a-service |
| **Telemetry** | Vehicle diagnostic data (RPM, voltage, GPS, temperature, etc.) |

---

*Layer Owner: Backend Infrastructure
Next Review: End of Phase 1 (Week 1 of development)*
