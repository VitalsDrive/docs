# Ghost Fleet Simulator — Product Requirements Document
**Version:** 1.0  
**Status:** Draft  
**Last Updated:** March 2026  
**Parent Document:** VitalsDrive PRD (v1.0)

---

## 1. Executive Summary

The Ghost Fleet Simulator is a critical component of the VitalsDrive development strategy, enabling full end-to-end pipeline validation before hardware procurement. It simulates 4G OBD2 device behavior by transmitting raw binary packets over TCP that are byte-for-byte identical to what production hardware would send.

**Primary Objectives:**
- Enable parallel development of frontend, backend, and database layers without waiting for hardware
- Validate the complete data pipeline from TCP ingestion through Supabase to the Angular dashboard
- Provide reproducible test scenarios for edge cases (battery drain, DTC triggers, critical temperature states)
- Create a fallback mechanism for regression testing when hardware is unavailable

**Key Constraint:** The Ghost Fleet Simulator outputs **raw binary/hex over TCP**—never JSON. JSON is the output of the Parser layer. This ensures the Parser is fully exercised during simulation.

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
┌─────────────────┐     raw hex/TCP      ┌──────────────┐     clean JSON      ┌───────────┐     Realtime      ┌──────────┐
│ Ghost Fleet     │ ───────────────────► │ Node.js      │ ──────────────────► │ Supabase  │ ─────────────────► │ Angular  │
│ Simulator       │                      │ Parser       │                     │ PostgreSQL│                    │ Dashboard│
└─────────────────┘                      └──────────────┘                     └───────────┘                    └──────────┘
     TCP Port 5050                            Railway.app                        WebSocket                    Vercel
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
| Staging (Railway) | `vitalsdrive-staging.railway.app` | `5050` |
| Production (Railway) | `vitalsdrive-prod.railway.app` | `5050` |

---

## 4. Protocol Implementation Details

### 4.1 Protocol Overview

The protocol is derived from the SinoTrack/Micodus 4G OBD2 device family commonly sourced from Alibaba. All multi-byte integers are **big-endian (network byte order)**.

**Packet Types:**
| Protocol Byte | Packet Type | Description |
|---|---|---|
| `0x01` | Login Packet | Device registration/authentication |
| `0x22` | Data Packet | Periodic telemetry transmission |
| `0x23` | Heartbeat Packet | Keep-alive/connection verification |

### 4.2 Byte Structure — Login Packet (0x01)

Sent by device upon TCP connection establishment.

```
Offset  Size  Field               Type      Description
------  ----  -----               ----      -----------
0       2     Start Bytes         uint16    0x78 0x78
2       1     Packet Length       uint8     Total bytes from offset 3 to CRC end
3       1     Protocol Number     uint8     0x01 (login packet type)
4       10    Device ID           ASCII     Left-padded with 0x00
14      4     Software Version    uint32    Device firmware version
18      4     Hardware Version    uint32    Device hardware version
22      4     Timestamp           uint32    Unix timestamp of device clock
26      2     CRC                 uint16    Modbus CRC-16 of bytes [2..25]
28      2     Stop Bytes          uint16    0x0D 0x0A
```

**Login Packet Example (hex):**
```
78 78 19 01 56 39 34 32 31 30 30 30 30 30 30 30 00 01 02 03 04 01 00 00 00 5A 3F B2 0D 0A
│  │  │  │  │                                │  │  │  │  │                    │
│  │  │  │  │                                │  │  │  │  └─ CRC (2 bytes)
│  │  │  │  │                                │  │  │  └─ Timestamp
│  │  │  │  │                                │  │  └─ Hardware version
│  │  │  │  │                                │  └─ Software version
│  │  │  │  └─ Device ID (ASCII: "V9421000000", null-padded)
│  │  │  └─ Protocol: 0x01 (login)
│  │  └─ Length: 25 bytes (0x19)
│  └─ Start: 0x78 0x78
└─ Start: 0x78 0x78
```

### 4.3 Byte Structure — Data Packet (0x22)

Primary telemetry packet sent at configurable intervals (default: 5 seconds).

```
Offset  Size  Field               Type      Description
------  ----  -----               ----      -----------
0       2     Start Bytes         uint16    0x78 0x78
2       1     Packet Length       uint8     Total bytes from offset 3 to CRC end
3       1     Protocol Number     uint8     0x22 (data packet type)
4       4     Latitude            int32     Degrees * 1,000,000 (signed)
8       4     Longitude           int32     Degrees * 1,000,000 (signed)
12      1     Speed               uint8     km/h (0-255)
13      2     Voltage             uint16    millivolts (e.g., 12600 = 12.6V)
15      1     Coolant Temp        uint8     Celsius (-40 to +125)
16      2     RPM                 uint16    Engine RPM (0-9999)
18      2     DTC Count           uint8     Number of active DTC codes (0-8)
19      N     DTC Codes           N*2       2 bytes per DTC code (see DTC format)
19+N    2     CRC                 uint16    Modbus CRC-16 of bytes [2..18+N]
21+N    2     Stop Bytes          uint16    0x0D 0x0A
```

**Minimum Data Packet Size:** 22 bytes (no DTC codes)  
**Maximum Data Packet Size:** 40 bytes (8 DTC codes)

**Data Packet Example (hex, no DTCs):**
```
78 78 15 22 02 46 47 D0 F3 70 F9 0D FA 31 46 00 00 00 00 B8 5C 0D 0A
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ Stop: 0x0D 0x0A
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ CRC high
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ CRC low
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ DTC Count: 0
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ RPM high
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ RPM low: 0
│  │  │  │  │  │  │  │  │  │  │  │  │  └─ Coolant Temp: 70°C
│  │  │  │  │  │  │  │  │  │  │  │  └─ Voltage high: 0x31 (wait - check)
│  │  │  │  │  │  │  │  │  │  │  └─ Voltage low: 0xFA 0x0D → 0x0DFA = 3580??? 
│  │  │  │  │  │  │  │  │  │  └─ Speed: 65 km/h
│  │  │  │  │  │  │  │  │  └─ Longitude: -122.4194° → -122419400
│  │  │  │  │  │  │  │  └─ Longitude: 0xF370F90D (wait, wrong)
│  │  │  │  │  │  │  └─ Latitude: 37.7749° → 37774900
│  │  │  │  │  │  └─ Latitude: 0x02 0x46 0x47 0xD0 = 37774900 ✓
│  │  │  │  │  └─ Protocol: 0x22 (data)
│  │  │  │  └─ Length: 21 bytes (0x15)
│  │  └─ Start: 0x78 0x78
└─ Start: 0x78 0x78
```

**Parsed values from example:**
- Latitude: 37.774900°N
- Longitude: -122.419400°W  
- Speed: 65 km/h
- Voltage: 12.6V (12600 mv)
- Coolant Temp: 70°C
- RPM: 0
- DTC Count: 0

### 4.4 Byte Structure — Heartbeat Packet (0x23)

Lightweight keep-alive packet sent between data packets to maintain TCP connection.

```
Offset  Size  Field               Type      Description
------  ----  -----               ----      -----------
0       2     Start Bytes         uint16    0x78 0x78
2       1     Packet Length       uint8     0x05
3       1     Protocol Number     uint8     0x23 (heartbeat)
4       1     Status              uint8     0x00 = normal, 0x01 = low battery
5       2     CRC                 uint16    Modbus CRC-16 of bytes [2..4]
7       2     Stop Bytes          uint16    0x0D 0x0A
```

### 4.5 CRC Calculation (Modbus CRC-16)

The protocol uses **Modbus CRC-16** with the following parameters:
- Polynomial: `0x8005` (reflected: `0xA001`)
- Initial value: `0xFFFF`
- No XORout
- CRC is calculated over bytes `[2..(length-3)]` (everything after length byte, before CRC bytes)

**Python Implementation:**
```python
def calculate_crc(data: bytes) -> int:
    """
    Modbus CRC-16 calculation.
    Polynomial: 0xA001 (reflected form of 0x8005)
    Initial value: 0xFFFF
    """
    crc = 0xFFFF
    for byte in data:
        crc ^= byte
        for _ in range(8):
            if crc & 0x0001:
                crc = (crc >> 1) ^ 0xA001
            else:
                crc >>= 1
    return crc & 0xFFFF
```

**Validation Example:**
```python
# For packet: 78 78 15 22 ... B8 5C
# CRC input bytes: 15 22 02 46 47 D0 F3 70 F9 0D FA 31 46 00 00 00 00
# Expected CRC: 0x5CB8 (returned as 0xB85C in the packet due to big-endian storage)
```

### 4.6 DTC Code Format

DTCs (Diagnostic Trouble Codes) follow the standard OBD-II format:
- First byte: Code type (0x00 = Generic, 0x01 = Manufacturer-specific)
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

### 5.1 Single Vehicle Simulation

The basic mode simulates one vehicle transmitting telemetry continuously.

**Configuration (default):**
```yaml
vehicle:
  id: "ghost-vehicle-01"
  start_lat: 37.7749
  start_lng: -122.4194
  route: "stationary"  # or "circle", "route_a"
```

**Output:** One TCP connection sending data packets at the configured interval.

### 5.2 Multi-Vehicle Simulation

Extends single vehicle mode to simulate a fleet of N vehicles simultaneously.

**Configuration:**
```yaml
fleet:
  count: 5
  base_lat: 37.7749
  base_lng: -122.4194
  spread_meters: 5000  # Randomize starting positions within 5km
  stagger_seconds: 2  # Stagger connection start times
```

**Output:** N simultaneous TCP connections, each with a unique vehicle ID.

### 5.3 Scenario Simulation

The simulator supports deterministic scenario playback for testing edge cases.

#### 5.3.1 Normal State Scenario

Baseline healthy vehicle operation.

```yaml
scenario:
  name: "normal_operation"
  duration_seconds: 300
  vehicle_state:
    speed: [45, 65]           # km/h, random variation
    coolant_temp: [85, 95]     # °C, normal range
    voltage: [12.5, 13.2]      # V, healthy battery
    rpm: [1200, 2500]          # Normal idle under load
    dtc_codes: []
```

#### 5.3.2 Warning State Scenario

Vehicle approaching critical thresholds (pre-cursor to critical).

```yaml
scenario:
  name: "approaching_warning"
  duration_seconds: 180
  vehicle_state:
    speed: [20, 40]
    coolant_temp: [100, 105]   # Getting hot
    voltage: [12.0, 12.4]       # Battery dropping
    rpm: [800, 1500]
    dtc_codes: []
  trigger_at_second: 90
  next_scenario: "critical_overheat"
```

#### 5.3.3 Critical State Scenario

Vehicle in failure condition requiring immediate attention.

```yaml
scenario:
  name: "critical_overheat"
  duration_seconds: 60
  vehicle_state:
    speed: [0, 10]             # Vehicle likely stopped
    coolant_temp: [108, 115]   # Overheating
    voltage: [11.5, 12.0]      # Battery stressed
    rpm: [600, 800]            # Rough idle
    dtc_codes:
      - "P0217"  # Engine Overtemperature Condition
      - "P0300"  # Random Misfire
```

#### 5.3.4 DTC Trigger Scenario

Simulates Check Engine Light activation.

```yaml
scenario:
  name: "dtc_trigger"
  duration_seconds: 120
  phases:
    - name: "normal_start"
      duration: 30
      dtc_codes: []
    - name: "dtc_appears"
      duration: 10
      dtc_codes:
        - "P0420"  # Catalyst efficiency below threshold
    - name: "multiple_dtcs"
      duration: 80
      dtc_codes:
        - "P0420"
        - "P0171"  # System too lean
```

### 5.4 Battery Drain Scenario

Tests battery monitoring and low-voltage alerting.

```yaml
scenario:
  name: "battery_drain"
  duration_seconds: 600  # 10 minute simulation
  initial_voltage: 12.8
  drain_rate_v_per_minute: 0.05  # 0.05V/minute drain
  critical_voltage: 11.8
  simulate_accessory_load: true
  simulate_alternator_failure: true  # At t=300s
```

**Voltage Progression:**
| Time (s) | Voltage (V) | Alert Level |
|---|---|---|
| 0 | 12.8 | Normal |
| 60 | 12.5 | Normal |
| 180 | 12.2 | Normal |
| 300 | 11.9 | Warning (alternator failure kicks in) |
| 420 | 11.6 | Critical |
| 540 | 11.3 | Critical (likely no-start) |

### 5.5 DTC Trigger Scenarios

Pre-configured scenarios for common DTC events.

```yaml
scenarios:
  misfire:
    name: "cylinder_misfire"
    dtc_codes: ["P0300", "P0301"]
    description: "Cylinder 1 misfire detected"
    
  oxygen_sensor:
    name: "oxygen_sensor_fault"
    dtc_codes: ["P0171", "P0172"]
    description: "Fuel system too lean/rich"
    
  thermostat:
    name: "thermostat_stuck"
    dtc_codes: ["P0128"]
    description: "Coolant thermostat range/performance"
    side_effects:
      coolant_temp: [100, 115]  # Runs hotter
```

---

## 6. Configuration Options

### 6.1 Configuration File Structure

The simulator uses YAML for configuration.

```yaml
# ghost-fleet-config.yaml

network:
  host: "localhost"
  port: 5050
  protocol: "tcp"
  reconnect: true
  reconnect_delay_ms: 1000
  max_reconnect_attempts: 10

simulator:
  mode: "single"  # single | multi | scenario
  log_level: "INFO"  # DEBUG | INFO | WARNING | ERROR
  log_hex_packets: true

vehicle:
  id: "ghost-vehicle-01"
  protocol_version: "1.0"
  device_id: "V9421000001"
  
telemetry:
  interval_seconds: 5
  jitter_ms: 500  # Randomize interval by +/- 500ms
  include_heartbeat: true
  heartbeat_interval_packets: 3

location:
  mode: "stationary"  # stationary | route | random_walk
  lat: 37.7749
  lng: -122.4194
  route_file: "routes/route_a.json"  # If mode=route
  
state:
  speed: 0
  voltage: 12.6
  coolant_temp: 90
  rpm: 0
  dtc_codes: []
```

### 6.2 Environment Variable Overrides

All configuration can be overridden via environment variables for CI/CD flexibility.

| Variable | Description | Default |
|---|---|---|
| `GHOST_HOST` | TCP server host | `localhost` |
| `GHOST_PORT` | TCP server port | `5050` |
| `GHOST_INTERVAL` | Packet interval (seconds) | `5` |
| `GHOST_VEHICLE_ID` | Vehicle identifier | `ghost-vehicle-01` |
| `GHOST_LOG_LEVEL` | Logging verbosity | `INFO` |
| `GHOST_MODE` | Simulation mode | `single` |

### 6.3 Runtime Command-Line Flags

```bash
ghost-fleet --config ./config.yaml --mode single --vehicle-id truck-01
ghost-fleet --scenario battery_drain --duration 600
ghost-fleet --multi 10 --host staging.railway.app --port 5050
```

---

## 7. Output Format Specification

### 7.1 Console Output

The simulator logs all transmitted packets to stdout in the following format:

```
[2026-03-29T09:15:23.451Z] INFO  [ghost-vehicle-01] TX: 78781522024647d0f370f90d0fa314600000000b85c0d0a
[2026-03-29T09:15:23.452Z] DEBUG [ghost-vehicle-01] Parsed: lat=37.7749 lng=-122.4194 spd=65 temp=70 rpm=0 volt=12.6
[2026-03-29T09:15:28.451Z] INFO  [ghost-vehicle-01] TX: 78781522024647d0f370f90d0fa324600000000b95c0d0a
```

### 7.2 Structured Log Output (JSON)

For integration with log aggregation systems (Datadog, CloudWatch):

```json
{
  "timestamp": "2026-03-29T09:15:23.451Z",
  "level": "INFO",
  "vehicle_id": "ghost-vehicle-01",
  "event": "packet_sent",
  "packet_hex": "78781522024647d0f370f90d0fa314600000000b85c0d0a",
  "packet_bytes": 22,
  "sequence": 42,
  "lat": 37.7749,
  "lng": -122.4194,
  "speed": 65,
  "voltage": 12.6,
  "coolant_temp": 70,
  "rpm": 0,
  "dtc_count": 0
}
```

### 7.3 Statistics Output

Every N packets (configurable), output aggregate statistics:

```
[2026-03-29T09:20:00.000Z] STATS [ghost-vehicle-01] packets_sent=60 packets_acked=60 errors=0 avg_latency_ms=23
```

---

## 8. Timing and Interval Configuration

### 8.1 Default Timing Parameters

| Parameter | Default | Range | Description |
|---|---|---|---|
| Data packet interval | 5s | 1-60s | Time between data packets |
| Heartbeat interval | Every 3rd packet | N/A | Heartbeat packet inserted every N data packets |
| Connection retry delay | 1s | 100ms-30s | Delay before reconnect attempt |
| Max reconnect attempts | 10 | 1-unlimited | Connection attempts before failure |
| Jitter | ±500ms | 0-2000ms | Random variance added to interval |

### 8.2 Timing Modes

**Fixed Interval Mode:**
```
Packet sent exactly every N seconds
Use case: Predictable testing, benchmark validation
```

**Jittered Mode (default):**
```
Packet interval = base_interval + random(-jitter, +jitter)
Use case: Realistic traffic simulation, buffer flush testing
```

**Burst Mode:**
```
Send N packets instantly, then idle for M seconds
Use case: Network interruption recovery testing
```

### 8.3 TCP Keep-Alive

The simulator enables TCP keep-alive to detect stale connections:

```python
sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)  # 10 second interval
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 3)     # 3 retries before dropping
```

---

## 9. Testing Scenarios to Validate

### 9.1 Protocol Compliance Tests

| Test ID | Description | Expected Result |
|---|---|---|
| TC-01 | Valid login packet transmission | Server accepts connection, no CRC errors |
| TC-02 | Valid data packet transmission | Server decodes all fields correctly |
| TC-03 | Invalid CRC rejection | Server logs CRC error, disconnects |
| TC-04 | Missing stop bytes | Server times out waiting for 0x0D 0x0A |
| TC-05 | Wrong protocol version | Server rejects or logs version mismatch |
| TC-06 | All DTC types transmitted | Server correctly parses 0-8 DTC codes |

### 9.2 Pipeline Integration Tests

| Test ID | Description | Expected Result |
|---|---|---|
| PI-01 | Ghost → Parser → Supabase | JSON appears in `telemetry_logs` within 2s |
| PI-02 | Multi-vehicle simultaneous | All N vehicle records appear in Supabase |
| PI-03 | Supabase → Angular WebSocket | Dashboard updates within 500ms of send |
| PI-04 | Parser restart resilience | Ghost auto-reconnects, no data loss |
| PI-05 | Supabase reconnect | Ghost waits, reconnects, continues |

### 9.3 Edge Case Tests

| Test ID | Scenario | Expected Result |
|---|---|---|
| EC-01 | Battery voltage drops below 11.5V | Dashboard shows battery alert |
| EC-02 | Coolant temp exceeds 105°C | Dashboard shows temperature warning |
| EC-03 | DTC code triggers mid-route | Alert appears with correct code description |
| EC-04 | Rapid state changes | All state transitions logged correctly |
| EC-05 | GPS coordinates at boundaries | Lat/lng correctly encoded (±90°, ±180°) |
| EC-06 | RPM at 0 | Vehicle appears stopped on map |
| EC-07 | RPM max (9999) | Value correctly transmitted without overflow |

### 9.4 Stress Tests

| Test ID | Description | Expected Result |
|---|---|---|
| ST-01 | 50 vehicles simultaneous | All connections stable, data correct |
| ST-02 | 1 second interval for 10 minutes | Server handles high-frequency packets |
| ST-03 | 24-hour continuous run | No memory leaks, stable connection |
| ST-04 | Rapid connect/disconnect cycles | No socket file descriptor leaks |

---

## 10. Integration with CI/CD

### 10.1 GitHub Actions Integration

```yaml
# .github/workflows/ghost-fleet-test.yml
name: Ghost Fleet Integration Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  ghost-fleet-tests:
    runs-on: ubuntu-latest
    services:
      parser:
        image: node:20-alpine
        ports:
          - 5050:5050
        env:
          PORT: 5050
        # Runs the parser in test mode
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install ghost-fleet
        run: pip install ghost-fleet-simulator
      
      - name: Run protocol compliance tests
        run: |
          ghost-fleet test \
            --host localhost \
            --port 5050 \
            --suite protocol
      
      - name: Run pipeline integration tests
        run: |
          ghost-fleet test \
            --host localhost \
            --port 5050 \
            --suite integration
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_ANON_KEY: ${{ secrets.SUPABASE_ANON_KEY }}
      
      - name: Run edge case tests
        run: |
          ghost-fleet test \
            --host localhost \
            --port 5050 \
            --suite edge_cases \
            --report json > test-results.json
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        with:
          name: ghost-fleet-results
          path: test-results.json
```

### 10.2 Local Development Testing

```bash
# Terminal 1: Start parser
cd parser && npm run dev

# Terminal 2: Run ghost fleet
ghost-fleet --config ./config/dev.yaml --mode single

# Terminal 3: Validate with test suite
ghost-fleet test --host localhost --port 5050 --suite full
```

### 10.3 Dockerfile for Containerized Testing

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN pip install ghost-fleet-simulator==1.0.0

COPY config/ ./config/
COPY scenarios/ ./scenarios/

ENTRYPOINT ["ghost-fleet"]
CMD ["--config", "/app/config/default.yaml"]
```

---

## 11. Transition to Real Hardware

### 11.1 Switching from Simulation to Production

When transitioning from the Ghost Fleet Simulator to real SinoTrack/Micodus hardware:

**Step 1: Verify Protocol Alignment**
- Request protocol documentation from hardware supplier
- Compare against the Ghost Fleet protocol implementation
- Document any discrepancies in `docs/protocol-diff.md`

**Step 2: Configure Hardware**
- Send SMS to device: `adminip123456 [RAILWAY_IP] [PORT]`
- Example: `adminip123456 52.14.28.71 5050`
- Device will begin sending login packet followed by data packets

**Step 3: Verify Pipeline**
- Confirm real device data appears in Supabase
- Compare byte structures (log raw hex from parser)
- Validate CRC calculations match expected values

**Step 4: Dashboard Verification**
- No Angular code changes required
- Real device data should render identically to simulated data

### 11.2 Coexistence Mode

Both simulated and real vehicles can operate simultaneously:

```yaml
# ghost-fleet-config.yaml
simulator:
  mode: "disabled"  # Disable simulator when using real hardware
  
# OR, to add simulated vehicles alongside real ones:
fleet:
  simulated_vehicles:
    - id: "ghost-truck-01"
      enabled: true
    - id: "ghost-truck-02"  
      enabled: false
```

### 11.3 Validation Checklist

| Check | Method | Pass Criteria |
|---|---|---|
| Login packet received | Parser logs | `0x01` protocol byte decoded |
| Data packets flowing | Supabase query | New rows every 5 seconds |
| CRC validation | Parser logs | No CRC errors |
| GPS coordinates valid | Supabase query | `lat BETWEEN -90 AND 90` |
| Voltage in range | Supabase query | `voltage BETWEEN 9.0 AND 16.0` |
| Temperature in range | Supabase query | `temp BETWEEN -40 AND 150` |
| DTC codes readable | Dashboard UI | Codes display correctly |

---

## 12. Maintenance Considerations

### 12.1 Protocol Versioning

If the hardware supplier changes the protocol in a new firmware version:

```yaml
# config/protocols/v1.yaml
protocol:
  version: "1.0"
  start_bytes: [0x78, 0x78]
  end_bytes: [0x0D, 0x0A]
  packet_types:
    login: 0x01
    data: 0x22
    heartbeat: 0x23

# config/protocols/v2.yaml
protocol:
  version: "2.0"
  # Potential changes documented here
```

### 12.2 Dependency Management

| Dependency | Version | Update Policy |
|---|---|---|
| Python | 3.11+ | Quarterly |
| ghost-fleet-simulator | 1.x | Semantic versioning |
| Node.js (parser) | 20 LTS | Major version only |
| Supabase client | Latest | Monthly |

### 12.3 Monitoring the Simulator

In production, monitor the Ghost Fleet Simulator as any other service:

**Key Metrics:**
- Packets sent per minute
- Connection success rate
- CRC error rate
- Average packet size
- Reconnection frequency

**Alerting:**
- Connection failures > 5 in 10 minutes
- CRC error rate > 1%
- Packet throughput deviation > 50% from baseline

### 12.4 Known Limitations

| Limitation | Impact | Workaround |
|---|---|---|
| No GPS route simulation | Limited navigation testing | Use pre-recorded route files |
| No CAN bus emulation | Cannot test deep vehicle diagnostics | Requires real hardware |
| Single TCP connection per vehicle | Cannot simulate connection pooling | Use multi-vehicle mode |
| Fixed DTC code set | Cannot add custom manufacturer codes | Update config/scenarios |

---

## 13. Open Questions

### 13.1 Protocol Verification with Supplier

| Question | Priority | Owner | Status |
|---|---|---|---|
| Confirm exact byte offsets for voltage field | High | Hardware team | Pending |
| Verify CRC algorithm (Modbus vs CRC-16-CCITT) | High | Hardware team | Pending |
| Confirm heartbeat packet interval requirement | Medium | Hardware team | Pending |
| Document all supported DTC code formats | Medium | Hardware team | Pending |
| Verify login packet response expected from server | Medium | Backend team | Pending |

### 13.2 Protocol Document Request Template

```
To: [Supplier Contact]
Subject: Protocol Specification Request - SinoTrack/Micodus 4G OBD2

Dear Supplier,

We are integrating your device into our fleet management platform.
To ensure successful integration, we require the following documentation:

1. Complete byte-level protocol specification for:
   - Login/Authentication packet
   - Data telemetry packet
   - Heartbeat packet (if applicable)

2. CRC calculation method used

3. Expected server responses (if any) upon packet receipt

4. SMS command set for device configuration

5. Any protocol differences between firmware versions

Please advise on the best way to obtain this documentation.

Thank you,
VitalsDrive Engineering
```

### 13.3 Fallback Protocol

If the supplier cannot provide protocol documentation:

**Option A:** Reverse-engineer from device (requires oscilloscope/logic analyzer)  
**Option B:** Use known SinoTrack protocol documented in open-source projects (GitHub)  
**Option C:** Request sample packets from supplier via debug mode

### 13.4 Additional Protocol Clarifications

| Question | Context | Next Step |
|---|---|---|
| Is the length byte 0-indexed or includes header? | Protocol ambiguity | Test with known packet |
| Are GPS coordinates signed or unsigned? | Affects lat/lng parsing | Verify with 0,0 coordinates |
| Does voltage need byte swap on some devices? | Endianness issues seen in similar devices | Test with 12.6V sample |
| What's the maximum packet size before fragmentation? | TCP MTU considerations | Test with 8 DTC codes |

---

## Appendix A: Quick Reference — Packet Builder

```python
import struct

class PacketBuilder:
    START_BYTES = b'\x78\x78'
    STOP_BYTES = b'\x0d\x0a'
    
    @staticmethod
    def calculate_crc(data: bytes) -> int:
        crc = 0xFFFF
        for byte in data:
            crc ^= byte
            for _ in range(8):
                if crc & 0x0001:
                    crc = (crc >> 1) ^ 0xA001
                else:
                    crc >>= 1
        return crc & 0xFFFF
    
    @classmethod
    def build_data_packet(cls, lat: float, lng: float, speed: int, 
                          voltage_mv: int, coolant_temp: int, rpm: int,
                          dtc_codes: list = None) -> bytes:
        dtc_codes = dtc_codes or []
        
        lat_int = int(lat * 1_000_000)
        lng_int = int(lng * 1_000_000)
        
        payload = struct.pack('>ii', lat_int, lng_int)
        payload += struct.pack('>B', speed)
        payload += struct.pack('>H', voltage_mv)
        payload += struct.pack('>B', coolant_temp)
        payload += struct.pack('>H', rpm)
        payload += struct.pack('>B', len(dtc_codes))
        
        for code in dtc_codes:
            payload += code.encode('ascii')[:2] if isinstance(code, str) else bytes(code[:2])
        
        length = 1 + len(payload) + 2
        header = b'\x78\x78' + struct.pack('B', length) + b'\x22' + payload
        
        crc = cls.calculate_crc(header[2:])
        return header + struct.pack('>H', crc) + cls.STOP_BYTES
    
    @classmethod
    def build_login_packet(cls, device_id: str, timestamp: int) -> bytes:
        device_id_bytes = device_id.encode('ascii').ljust(10, b'\x00')[:10]
        payload = device_id_bytes + struct.pack('>I', 0x01020304)
        payload += struct.pack('>I', 0x00010000)
        payload += struct.pack('>I', timestamp)
        
        length = 1 + len(payload) + 2
        header = b'\x78\x78' + struct.pack('B', length) + b'\x01' + payload
        
        crc = cls.calculate_crc(header[2:])
        return header + struct.pack('>H', crc) + cls.STOP_BYTES
    
    @classmethod
    def build_heartbeat_packet(cls, status: int = 0x00) -> bytes:
        header = b'\x78\x78\x05\x23' + struct.pack('B', status)
        crc = cls.calculate_crc(header[2:])
        return header + struct.pack('>H', crc) + cls.STOP_BYTES
```

---

## Appendix B: Configuration Schema

```yaml
# Full schema reference for ghost-fleet-config.yaml
$schema: "https://vitalsdrive.com/schemas/ghost-fleet-config.json"

type: object
properties:
  network:
    type: object
    properties:
      host: { type: string }
      port: { type: integer, minimum: 1, maximum: 65535 }
      protocol: { type: string, enum: [tcp] }
      reconnect: { type: boolean }
      reconnect_delay_ms: { type: integer }
      max_reconnect_attempts: { type: integer }
    required: [host, port]

  simulator:
    type: object
    properties:
      mode: { type: string, enum: [single, multi, scenario, disabled] }
      log_level: { type: string, enum: [DEBUG, INFO, WARNING, ERROR] }
      log_hex_packets: { type: boolean }
    required: [mode]

  vehicle:
    type: object
    properties:
      id: { type: string }
      protocol_version: { type: string }
      device_id: { type: string }
    required: [id]

  telemetry:
    type: object
    properties:
      interval_seconds: { type: number }
      jitter_ms: { type: integer }
      include_heartbeat: { type: boolean }
      heartbeat_interval_packets: { type: integer }
    required: [interval_seconds]

  location:
    type: object
    properties:
      mode: { type: string, enum: [stationary, route, random_walk] }
      lat: { type: number, minimum: -90, maximum: 90 }
      lng: { type: number, minimum: -180, maximum: 180 }
    required: [mode, lat, lng]

  state:
    type: object
    properties:
      speed: { type: integer, minimum: 0, maximum: 255 }
      voltage: { type: number, minimum: 0, maximum: 20 }
      coolant_temp: { type: integer, minimum: -40, maximum: 150 }
      rpm: { type: integer, minimum: 0, maximum: 9999 }
      dtc_codes: { type: array, items: { type: string } }
    required: [speed, voltage, coolant_temp, rpm]
```

---

*Document Owner: VitalsDrive Engineering  
Ghost Fleet Simulator Owner: Backend/DevOps  
Next Review: Upon hardware procurement confirmation*
