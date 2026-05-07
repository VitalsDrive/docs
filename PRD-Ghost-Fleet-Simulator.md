# Ghost Fleet Simulator — Product Requirements Document
**Version:** 2.0
**Status:** Draft
**Last Updated:** March 2026
**Parent Document:** VitalsDrive PRD (v1.0)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Purpose and Objectives](#2-purpose-and-objectives)
3. [Architecture Integration](#3-architecture-integration)
4. [Protocol Specification](#4-protocol-specification)
5. [Simulator Features](#5-simulator-features)
6. [Configuration](#6-configuration)
7. [Output Format](#7-output-format)
8. [Testing Scenarios](#8-testing-scenarios)
9. [Hardware Transition](#9-hardware-transition)
10. [Open Questions](#10-open-questions)

---

## 1. Executive Summary

The Ghost Fleet Simulator is a critical component of the VitalsDrive development strategy, enabling full end-to-end pipeline validation before hardware procurement. It simulates 4G OBD2 device behavior by transmitting raw binary packets over TCP that are byte-for-byte identical to what production hardware would send.

**Primary Objectives:**
- Enable parallel development of frontend, backend, and database layers without waiting for hardware
- Validate the complete data pipeline from TCP ingestion through Supabase to the Angular dashboard
- Provide reproducible test scenarios for edge cases (battery drain, DTC triggers, critical temperature states)
- Create a fallback mechanism for regression testing when hardware is unavailable

**Key Constraint:** The Ghost Fleet Simulator outputs **raw binary/hex over TCP** — never JSON. JSON is the output of the Parser layer. This ensures the Parser is fully exercised during simulation.

---

## 2. Purpose and Objectives

### 2.1 Development Acceleration

The simulator allows the team to build against the production architecture immediately, without dependency on physical hardware lead times (typically 2–4 weeks from Alibaba suppliers).

| Without Simulator | With Simulator |
|---|---|
| Wait 2–4 weeks for hardware | Start Day 1 |
| Unverified pipeline until hardware arrives | Fully validated pipeline |
| Late discovery of protocol mismatches | Protocol validated in simulation |
| Hardware-dependent testing | Automated CI/CD testable |

### 2.2 Regression and CI/CD Integration

Once the simulator is stable, it serves as a permanent fixture in the test suite. Any code changes to the Parser can be validated against known packet sequences without physical devices.

### 2.3 Edge Case Coverage

Physical testing cannot reliably reproduce edge cases like:
- Rapid battery voltage decay
- Sudden DTC code triggers
- Boundary values (extreme temperatures, max RPM)
- Network interruption/reconnection scenarios

The simulator generates these scenarios deterministically.

---

## 3. Architecture Integration

### 3.1 Position in Data Pipeline

```
Ghost Fleet Simulator → (raw hex/TCP) → Node.js Parser → (clean JSON) → Supabase → (Realtime) → Angular Dashboard
TCP Port 5050              Railway.app                      PostgreSQL        WebSocket          Vercel
```

### 3.2 Responsibility Boundaries

| Layer | Component | Responsibility |
|---|---|---|
| **L0** | Ghost Fleet Simulator | Generates well-formed binary packets per protocol spec |
| **L1** | Node.js TCP Parser | Receives raw bytes, decodes protocol, outputs normalized JSON |
| **L2** | Supabase | Stores JSON telemetry records, pushes via WebSocket |
| **L3** | Angular Dashboard | Subscribes to Realtime, renders vehicle health |

**Critical:** The Ghost Fleet Simulator operates at L0 only. It has no knowledge of Supabase, Angular, or the Parser internals. It only knows the protocol specification.

### 3.3 Network Topology

| Environment | Host | Port |
|---|---|---|
| Development (local) | `localhost` | `5050` |
| Staging (Railway) | `staging.railway.app` | `5050` |
| Production (Railway) | `ingestion.vitalsdrive.com` | `5050` |

---

## 4. Protocol Specification

### 4.1 Protocol Overview

The protocol is derived from the SinoTrack/Micodus 4G OBD2 device family commonly sourced from Alibaba. All multi-byte integers are **big-endian (network byte order)**.

**Packet Types:**

| Protocol Byte | Packet Type | Description |
|---|---|---|
| `0x01` | Login Packet | Device registration/authentication |
| `0x22` | Data Packet | Periodic telemetry transmission |
| `0x23` | Heartbeat Packet | Keep-alive/connection verification |

### 4.2 Login Packet (0x01)

Sent by device upon TCP connection establishment.

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0 | 2 | Start Bytes | uint16 | `0x78 0x78` |
| 2 | 1 | Packet Length | uint8 | Total bytes from offset 3 to CRC end |
| 3 | 1 | Protocol Number | uint8 | `0x01` |
| 4 | 10 | Device ID | ASCII | Left-padded with `0x00` |
| 14 | 4 | Software Version | uint32 | Device firmware version |
| 18 | 4 | Hardware Version | uint32 | Device hardware version |
| 22 | 4 | Timestamp | uint32 | Unix timestamp of device clock |
| 26 | 2 | CRC | uint16 | Modbus CRC-16 of bytes [2..25] |
| 28 | 2 | Stop Bytes | uint16 | `0x0D 0x0A` |

**Login ACK (server → device):**
- Protocol byte: `0x81` (login ack)
- Result byte: `0x00` (success)
- Format: `0x78 0x78 0x05 0x81 0x00 [CRC] 0x0D 0x0A`

### 4.3 Data Packet (0x22)

Primary telemetry packet sent at configurable intervals (default: 5 seconds).

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0 | 2 | Start Bytes | uint16 | `0x78 0x78` |
| 2 | 1 | Packet Length | uint8 | Total bytes from offset 3 to CRC end |
| 3 | 1 | Protocol Number | uint8 | `0x22` |
| 4 | 4 | Latitude | int32 | Degrees × 1,000,000 (signed) |
| 8 | 4 | Longitude | int32 | Degrees × 1,000,000 (signed) |
| 12 | 1 | Speed | uint8 | km/h (0-255) |
| 13 | 2 | Voltage | uint16 | Millivolts (e.g., 12600 = 12.6V) |
| 15 | 1 | Coolant Temp | uint8 | Celsius (-40 to +125) |
| 16 | 2 | RPM | uint16 | Engine RPM (0-9999) |
| 18 | 1 | DTC Count | uint8 | Number of active DTC codes (0-8) |
| 19 | N | DTC Codes | N×2 | 2 bytes per DTC code (see §4.6) |
| 19+N | 2 | CRC | uint16 | Modbus CRC-16 of bytes [2..18+N] |
| 21+N | 2 | Stop Bytes | uint16 | `0x0D 0x0A` |

**Constraints:**
- Minimum size: 22 bytes (no DTC codes)
- Maximum size: 40 bytes (8 DTC codes)

### 4.4 Heartbeat Packet (0x23)

Lightweight keep-alive sent between data packets to maintain TCP connection.

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0 | 2 | Start Bytes | uint16 | `0x78 0x78` |
| 2 | 1 | Packet Length | uint8 | `0x05` |
| 3 | 1 | Protocol Number | uint8 | `0x23` |
| 4 | 1 | Status | uint8 | `0x00` = normal, `0x01` = low battery |
| 5 | 2 | CRC | uint16 | Modbus CRC-16 of bytes [2..4] |
| 7 | 2 | Stop Bytes | uint16 | `0x0D 0x0A` |

### 4.5 CRC Calculation

The protocol uses **Modbus CRC-16**:
- Polynomial: `0x8005` (reflected: `0xA001`)
- Initial value: `0xFFFF`
- No XORout
- Calculated over bytes `[2..(length-3)]` (everything after length byte, before CRC bytes)

### 4.6 DTC Code Format

DTCs follow the standard OBD-II format:
- First byte: Code type (`0x00` = Generic, `0x01` = Manufacturer-specific)
- Second byte: High nibble = digit 1, Low nibble = digit 2
- Third byte: High nibble = digit 3, Low nibble = digit 4

**Example DTC: P0301**
- Byte 1: `0x00` (Powertrain, Generic)
- Byte 2: `0x03` → P, 0
- Byte 3: `0x01` → 3, 0 → Combined: P0301

**Common DTCs for Simulation:**

| Code | Description |
|---|---|
| P0300 | Random/Multiple Cylinder Misfire Detected |
| P0301 | Cylinder 1 Misfire Detected |
| P0420 | Catalyst System Efficiency Below Threshold |
| P0171 | System Too Lean (Bank 1) |
| P0128 | Coolant Thermostat Range/Performance |
| P0562 | System Voltage Low |
| P0563 | System Voltage High |

---

## 5. Simulator Features

### 5.1 Single Vehicle Mode

Basic mode simulating one vehicle transmitting telemetry continuously.

**Default configuration:** Single TCP connection sending data packets at configurable interval from a fixed or slowly moving GPS position.

### 5.2 Multi-Vehicle Mode

Extends single vehicle to simulate N vehicles simultaneously, each with a unique TCP connection and device ID.

**Configuration parameters:**
- Vehicle count
- Base GPS coordinates
- Position spread (randomize within radius)
- Connection stagger (avoid simultaneous connect storms)

### 5.3 Scenario Modes

The simulator supports deterministic scenarios for testing edge cases:

| Scenario | Behavior | Purpose |
|---|---|---|
| Normal | Stable readings within normal ranges | Baseline pipeline validation |
| Warning | Temperature approaching threshold (100-104°C) | Verify warning state display |
| Critical | Temperature > 105°C, voltage < 12.0V | Verify critical alert generation |
| DTC Trigger | Inject DTC codes mid-stream | Verify DTC alert system |
| Battery Drain | Gradual voltage decay from 12.8V to 11.5V | Verify predictive alert logic |
| GPS Loss | Periodically set GPS valid = 0 | Verify map handling |
| Disconnect | Simulate TCP disconnect + reconnect | Verify session recovery |

---

## 6. Configuration

### 6.1 Configuration File

The simulator uses YAML configuration files for vehicle and fleet settings.

**Vehicle-level settings:**
- `id`: Unique vehicle identifier
- `start_lat`, `start_lng`: Initial GPS position
- `route`: Route pattern (stationary, circle, predefined)
- `imei`: Device IMEI for parser lookup

**Telemetry settings:**
- `interval_seconds`: Time between data packets (default: 5)
- `jitter_ms`: Random variation in interval (default: 500)
- `heartbeat_interval_seconds`: Time between heartbeat packets (default: 30)

**Fleet settings (multi-vehicle):**
- `count`: Number of simulated vehicles
- `base_lat`, `base_lng`: Center GPS coordinates
- `spread_meters`: Randomize positions within radius
- `stagger_seconds`: Stagger connection start times

**Connection settings:**
- `host`: Parser host (default: localhost)
- `port`: Parser port (default: 5050)
- `reconnect_delay_seconds`: Delay after disconnect (default: 5)

### 6.2 CLI Flags

Command-line flags override configuration file values:
- `--config`: Path to YAML config file
- `--host`: Override parser host
- `--port`: Override parser port
- `--vehicles`: Override vehicle count
- `--scenario`: Select scenario mode
- `--log-level`: Set logging verbosity

### 6.3 Environment Variables

| Variable | Description | Default |
|---|---|---|
| `GHOST_FLEET_HOST` | Parser host | localhost |
| `GHOST_FLEET_PORT` | Parser port | 5050 |
| `GHOST_FLEET_VEHICLES` | Vehicle count | 1 |
| `GHOST_FLEET_INTERVAL` | Telemetry interval (seconds) | 5 |

---

## 7. Output Format

### 7.1 Console Output

Each packet sent is logged with vehicle ID, packet type, and key telemetry values for debugging:

```
[vehicle-01] Data: lat=37.7749 lng=-122.4194 speed=65 temp=70 voltage=12.6 rpm=1450 dtcs=[]
[vehicle-01] Heartbeat: status=normal
[vehicle-01] Login: imei=123456789012345
```

### 7.2 Structured Logging

When `--log-level=json` is set, output is structured JSON suitable for CI/CD pipelines:

```json
{
  "vehicle_id": "vehicle-01",
  "packet_type": "data",
  "lat": 37.7749,
  "lng": -122.4194,
  "speed": 65,
  "temp": 70,
  "voltage": 12.6,
  "rpm": 1450,
  "dtc_codes": [],
  "timestamp": "2026-03-29T14:32:00.123Z"
}
```

### 7.3 Statistics Output

At end of run, the simulator outputs summary statistics:
- Total packets sent per vehicle
- Connection uptime
- Reconnection count
- Parse error count (if parser reports back)

---

## 8. Testing Scenarios

### 8.1 Protocol Compliance Tests

Verify the simulator produces packets that the parser accepts:
- Login packet is valid and receives ack
- Data packet CRC is correct
- Heartbeat packet maintains connection
- All packet types follow the byte layout specification

### 8.2 Pipeline Integration Tests

End-to-end validation:
- Ghost Fleet → Parser → Supabase → verify row inserted
- Multi-vehicle: all N vehicles insert correctly
- Scenario mode: critical scenario triggers alert generation

### 8.3 Edge Cases

| Test | Expected |
|---|---|
| Max DTC count (8) | Parser extracts all 8 codes |
| Voltage at boundary (12.4V) | Warning alert triggered |
| Temperature at boundary (105°C) | Critical alert triggered |
| GPS coordinates at extremes | Parser validates range |
| Rapid connect/disconnect | Parser cleans up sessions |

### 8.4 Stress Tests

| Test | Target |
|---|---|
| 50 simultaneous vehicles | Parser handles all connections |
| 1 packet/second per vehicle | Parser throughput > 50 pps |
| 1 hour continuous run | No memory leaks in parser |

---

## 9. Hardware Transition

### 9.1 Protocol Verification Checklist

Before transitioning to real hardware:
- [ ] Protocol specification verified against supplier documentation
- [ ] CRC algorithm confirmed (Modbus CRC-16)
- [ ] Byte offsets validated with real device capture
- [ ] Login ACK format matches device expectations
- [ ] DTC code encoding verified

### 9.2 Coexistence

When real devices are available, the simulator should be able to run alongside them. Real devices and ghost vehicles are distinguished by their IMEI prefixes (e.g., ghost devices use `GHOST-*` IMEI prefix).

---

## 10. Open Questions

| # | Question | Impact | Status |
|---|---|---|---|
| O-1 | Has the official SinoTrack protocol specification been received from the supplier? | Critical | Pending |
| O-2 | Does the device wait for login ACK before sending data? | High | Unknown |
| O-3 | Is DTC data included in every data packet or on separate request? | Medium | Unknown |
| O-4 | Is the voltage field raw ADC reading or already calibrated (mV)? | Medium | Unknown |
| O-5 | Is the RPM encoding confirmed (2-byte value or split high/low)? | Critical | Pending |

---

*Document Owner: Backend Infrastructure
Next Review: After hardware arrival and protocol verification*
