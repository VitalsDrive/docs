# VitalsDrive — Architecture Overview
**Version:** 1.0  
**Status:** Draft  
**Last Updated:** March 2026  
**Document Owner:** VitalsDrive Engineering  
**Parent Documents:**
- `VitalsDrive_PRD.md` (Main PRD)
- `PRD-Layer1-Ingestion-Server.md`
- `PRD-Layer2-Data-Storage.md`
- `PRD-Layer3-Angular-Dashboard.md`
- `PRD-Ghost-Fleet-Simulator.md`

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Overview](#2-system-overview)
3. [Data Flow Architecture](#3-data-flow-architecture)
4. [Component Architecture](#4-component-architecture)
5. [Technology Stack Summary](#5-technology-stack-summary)
6. [Infrastructure Topology](#6-infrastructure-topology)
7. [Security Architecture](#7-security-architecture)
8. [Deployment Architecture](#8-deployment-architecture)
9. [Development Workflow](#9-development-workflow)
10. [Environment Configuration](#10-environment-configuration)
11. [Monitoring and Observability](#11-monitoring-and-observability)
12. [Disaster Recovery](#12-disaster-recovery)
13. [Scaling Strategy](#13-scaling-strategy)
14. [Integration Points](#14-integration-points)
15. [Development Roadmap Dependencies](#15-development-roadmap-dependencies)

---

## 1. Executive Summary

### 1.1 Purpose

This document provides a comprehensive architectural overview of the VitalsDrive platform, a "health-first" vehicle diagnostic and telematics system. It defines how all layers interact to deliver real-time fleet health monitoring to small fleet operators (2–15 vehicles).

### 1.2 Product Overview

| Attribute | Detail |
|---|---|
| **Product** | VitalsDrive — Fleet Health Monitoring Platform |
| **Tagline** | *Fleet Health, Simplified.* |
| **Target Market** | Small fleet operators (2–15 vehicles) |
| **Core Value** | Affordable, diagnostics-forward monitoring with actionable alerts |
| **MVP Features** | DTC Alert, Battery Health Monitor, Coolant Temperature Alert, Live Fleet Map |

### 1.3 Architecture Highlights

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         VitalsDrive Architecture                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐     raw hex/TCP      ┌──────────────┐     clean JSON     │
│   │   Ghost      │────────────────────▶│   Node.js     │───────────────────▶│
│   │   Fleet      │                      │   Parser      │                    │
│   │   Simulator  │                      │   (Railway)   │                    │
│   └──────────────┘                      └──────────────┘                    │
│                                                │                              │
│   ┌──────────────┐     raw hex/TCP           │         ┌──────────────┐     │
│   │   Real OBD2   │──────────────────────────┘────────▶│   Supabase    │     │
│   │   Devices     │                                ┌────▶│   PostgreSQL  │─────┘
│   └──────────────┘                                │     └──────────────┘     │
│                                                   │            │              │
│                                                   │   WebSocket (Realtime)   │
│                                                   │            ▼              │
│                                                   │     ┌──────────────┐       │
│                                                   └─────│   Angular    │       │
│                                                         │   Dashboard   │       │
│                                                         │   (Vercel)    │       │
│                                                         └──────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.4 Key Architectural Principles

1. **Hardware-Agnostic Output:** The Parser normalizes all device data to a common JSON schema regardless of manufacturer
2. **Real-Time First:** Supabase Realtime (WebSocket) enables instant dashboard updates
3. **Simulation-First Development:** Ghost Fleet enables full pipeline validation before hardware arrives
4. **Scale-to-Zero Infrastructure:** Free tiers utilized during MVP/pilot phases

---

## 2. System Overview

### 2.1 Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    VitalsDrive Platform                               │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                              Layer 0: Simulation                                 │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │ │
│  │  │                      Ghost Fleet Simulator                                │   │ │
│  │  │                                                                          │   │ │
│  │  │   • Generates raw binary/hex packets per SinoTrack protocol             │   │ │
│  │  │   • Sends Login + Data packets over TCP                                  │   │ │
│  │  │   • Supports multi-vehicle scenarios (up to 50 simultaneous)            │   │ │
│  │  │   • Enables deterministic edge case testing                               │   │ │
│  │  │   • Configuration: YAML-based + environment variable overrides           │   │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                           │                                             │
│                                           │ raw hex (TCP port 5050)                       │
│                                           ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                            Layer 1: TCP Ingestion Server                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │ │
│  │  │                     Node.js TCP Server (Railway)                        │   │ │
│  │  │                                                                          │   │ │
│  │  │   Responsibilities:                                                      │   │ │
│  │  │   • Accept TCP connections on port 5050                                 │   │ │
│  │  │   • Parse Login packets (0x01) — validate, acknowledge                  │   │ │
│  │  │   • Parse Data packets (0x22) — extract telemetry fields               │   │ │
│  │  │   • Parse Heartbeat packets (0x23) — maintain connection               │   │ │
│  │  │   • Validate CRC-16 checksums                                           │   │ │
│  │  │   • Map device IMEI to vehicle_id via Supabase lookup                   │   │ │
│  │  │   • Push decoded JSON to Supabase REST API                              │   │ │
│  │  │   • Queue outgoing messages during Supabase outage                      │   │ │
│  │  │                                                                          │   │ │
│  │  │   Protocols Supported:                                                   │   │ │
│  │  │   • Alibaba SinoTrack (primary)                                         │   │ │
│  │  │   • Micodus (secondary)                                                  │   │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                           │                                             │
│                                           │ decoded JSON (REST API)                      │
│                                           ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                          Layer 2: Data Storage (Supabase)                       │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │ │
│  │  │                      PostgreSQL Database                                  │   │ │
│  │  │                                                                          │   │ │
│  │  │   Core Tables:                                                           │   │ │
│  │  │   ├── telemetry_logs     — Vehicle telemetry data points                 │   │ │
│  │  │   ├── vehicles            — Registered vehicles                            │   │ │
│  │  │   ├── fleets              — Fleet groupings                                │   │ │
│  │  │   ├── users              — Application users with roles                   │   │ │
│  │  │   ├── fleet_members      — User-fleet many-to-many relationship          │   │ │
│  │  │   ├── alerts             — Generated alerts from threshold violations     │   │ │
│  │  │   └── telemetry_rules    — User-defined threshold rules                   │   │ │
│  │  │                                                                          │   │ │
│  │  │   Features:                                                               │   │ │
│  │  │   • Row Level Security (RLS) for tenant isolation                         │   │ │
│  │  │   • Postgres Realtime (WebSocket) for live updates                        │   │ │
│  │  │   • Automated backups (daily + point-in-time)                            │   │ │
│  │  │   • Database triggers for alert generation                               │   │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                           │                                             │
│                                           │ WebSocket (Supabase Realtime)                │
│                                           ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                       Layer 3: Angular Dashboard (Vercel)                       │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │ │
│  │  │                     Angular 19/20 SPA                                    │   │ │
│  │  │                                                                          │   │ │
│  │  │   State Management:                                                      │   │ │
│  │  │   • Angular Signals for synchronous state                                │   │ │
│  │  │   • RxJS for async operations and Supabase Realtime                     │   │ │
│  │  │   • bufferTime(500ms) to batch high-frequency telemetry updates         │   │ │
│  │  │                                                                          │   │ │
│  │  │   Core Features:                                                          │   │ │
│  │  │   ├── DTC Alert System    — Real-time Check Engine notifications        │   │ │
│  │  │   ├── Battery Monitor     — Voltage tracking + predictive alerts        │   │ │
│  │  │   ├── Coolant Temp Alert  — Overheating warnings (>100°C/105°C)         │   │ │
│  │  │   └── Live Fleet Map      — Leaflet.js with real-time vehicle markers   │   │ │
│  │  │                                                                          │   │ │
│  │  │   Key Components:                                                         │   │ │
│  │  │   ├── TelemetryService     — Supabase Realtime subscription              │   │ │
│  │  │   ├── AlertService         — Alert creation, management, notifications   │   │ │
│  │  │   ├── VehicleService       — Vehicle registry and derived state         │   │ │
│  │  │   ├── HealthGaugeComponent — SVG arc gauge for coolant temp             │   │ │
│  │  │   ├── BatteryStatusComponent— Voltage display with trend line           │   │ │
│  │  │   └── FleetMapComponent    — Leaflet.js map with vehicle markers        │   │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Layer Responsibility Summary

| Layer | Component | Hosting | Protocol | Output | Document Reference |
|---|---|---|---|---|---|
| **L0** | Ghost Fleet Simulator | Local/Dev | TCP (raw hex) | Binary packets to Parser | `PRD-Ghost-Fleet-Simulator.md` |
| **L1** | TCP Ingestion Server | Railway | TCP → REST | Normalized JSON to Supabase | `PRD-Layer1-Ingestion-Server.md` |
| **L2** | Data Storage | Supabase | REST/WebSocket | Telemetry records to Dashboard | `PRD-Layer2-Data-Storage.md` |
| **L3** | Angular Dashboard | Vercel | WebSocket | Real-time UI updates | `PRD-Layer3-Angular-Dashboard.md` |

---

## 3. Data Flow Architecture

### 3.1 End-to-End Telemetry Path

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Vehicle   │────▶│  OBD2 GPS   │────▶│   4G TCP     │────▶│    Node.js  │────▶│   Supabase  │
│   Engine    │     │   Device    │     │   Network    │     │   Parser    │     │  PostgreSQL │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
     │                   │                   │                   │                   │
     │ CAN bus           │ 0x78 0x78...      │                   │                   │
     │ telemetry         │ raw hex            │                   │                   │
     │                   │                    │                   │                   │
     │                   │                   │                   │                   │
     │                   │                   │                   │                   │
     ▼                   ▼                   ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Vehicle    │     │  Device     │     │   Railway   │     │   Parse +   │     │   Store +   │
│  sensors    │     │  firmware   │     │   network   │     │   Validate  │     │   Publish   │
│  generate   │────▶│  encodes    │────▶│   routing   │────▶│   packets   │────▶│   via WS    │
│  raw data   │     │  to binary  │     │             │     │   to JSON   │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                                                                  │
                                                                                  │
                                                                                  ▼
                                                                          ┌─────────────┐
                                                                          │   Angular   │
                                                                          │  Dashboard  │
                                                                          └─────────────┘
                                                                                  │
                                                                                  ▼
                                                                          ┌─────────────┐
                                                                          │  Real-time  │
                                                                          │     UI      │
                                                                          └─────────────┘
```

### 3.2 Packet Lifecycle

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           Packet Lifecycle (Ghost Fleet Path)                         │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│  Phase 1: Generation (Ghost Fleet Simulator)                                          │
│  ─────────────────────────────────────────────────────                                │
│  ┌────────────────────────────────────────────────────────────────────────────────┐   │
│  │ Python/Node script generates binary packet per SinoTrack protocol:              │   │
│  │                                                                                 │   │
│  │   78 78 15 22 02 46 47 D0 F3 70 F9 0D FA 31 46 00 00 00 00 B8 5C 0D 0A       │   │
│  │   │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │             │   │
│  │   │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ Stop      │   │
│  │   │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ CRC         │   │
│  │   │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ DTC Count     │   │
│  │   │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ RPM             │   │
│  │   │  │  │  │  │  │  │  │  │  │  │  │  │  │  └─ Coolant Temp        │   │
│  │   │  │  │  │  │  │  │  │  │  │  │  │  └─ Voltage                │   │
│  │   │  │  │  │  │  │  │  │  │  │  └─ Speed                     │   │
│  │   │  │  │  │  │  │  │  │  └─ Longitude                    │   │
│  │   │  │  │  │  │  │  └─ Latitude                       │   │
│  │   │  │  │  │  └─ Protocol: 0x22 (Data)               │   │
│  │   │  │  │  └─ Length: 21 bytes                       │   │
│  │   │  └─ Start: 0x78 0x78                            │   │
│  └────────────────────────────────────────────────────────────────────────────────┘   │
│                                           │                                             │
│                                           │ TCP write()                                 │
│                                           ▼                                             │
│  Phase 2: Ingestion (Node.js Parser)                                                      │
│  ─────────────────────────────────────────────────────                                │
│  ┌────────────────────────────────────────────────────────────────────────────────┐   │
│  │ Node.js net.Server receives chunk, buffers until complete packet:               │   │
│  │                                                                                 │   │
│  │   1. Find start bytes (0x78 0x78)                                              │   │
│  │   2. Read length byte                                                          │   │
│  │   3. Wait for length + 6 bytes (header + CRC + stop)                           │   │
│  │   4. Validate CRC-16                                                           │   │
│  │   5. Extract fields: lat, lng, speed, voltage, temp, rpm, dtc_codes           │   │
│  │   6. Look up vehicle_id by device IMEI in Supabase                             │   │
│  │   7. Push normalized JSON to Supabase /rest/v1/telemetry_logs                 │   │
│  └────────────────────────────────────────────────────────────────────────────────┘   │
│                                           │                                             │
│                                           │ POST /rest/v1/telemetry_logs               │
│                                           ▼                                             │
│  Phase 3: Storage (Supabase PostgreSQL)                                                 │
│  ─────────────────────────────────────────────────────                                │
│  ┌────────────────────────────────────────────────────────────────────────────────┐   │
│  │ PostgreSQL receives INSERT:                                                     │   │
│  │                                                                                 │   │
│  │   INSERT INTO telemetry_logs (                                                │   │
│  │     vehicle_id, lat, lng, speed, temp, voltage, rpm, dtc_codes, timestamp     │   │
│  │   ) VALUES (                                                                   │   │
│  │     'uuid-xxx', 37.7749, -122.4194, 65, 70, 12.6, 0, '{}', NOW()              │   │
│  │   )                                                                            │   │
│  │                                                                                 │   │
│  │   ──▶ Trigger: generate_alert_if_needed()                                     │   │
│  │   ──▶ Publish to Supabase Realtime channel                                    │   │
│  └────────────────────────────────────────────────────────────────────────────────┘   │
│                                           │                                             │
│                                           │ WebSocket (postgres_changes)               │
│                                           ▼                                             │
│  Phase 4: Presentation (Angular Dashboard)                                             │
│  ─────────────────────────────────────────────────────                                │
│  ┌────────────────────────────────────────────────────────────────────────────────┐   │
│  │ Angular TelemetryService receives payload:                                     │   │
│  │                                                                                 │   │
│  │   1. Supabase SDK `.on('postgres_changes', ...)` callback                     │   │
│  │   2. bufferTime(500ms) batches rapid updates                                   │   │
│  │   3. Update vehicle Signal (telemetryMap)                                      │   │
│  │   4. AlertService checks thresholds:                                           │   │
│  │      • Coolant > 105°C? Create critical alert                                  │   │
│  │      • Voltage < 12.0V? Create battery alert                                  │   │
│  │      • New DTC codes? Create DTC alert with translation                       │   │
│  │   5. UI components react to Signal changes (OnPush change detection)          │   │
│  └────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Data Transformation at Each Layer

| Stage | Input | Output | Transformation |
|---|---|---|---|
| **Vehicle Sensors** | Analog/digital signals | CAN bus data | Vehicle ECUs generate OBD-II PIDs |
| **OBD2 Device** | CAN bus frames | Binary protocol packet | Device firmware encodes to SinoTrack format |
| **Network** | Binary packet | TCP stream | 4G module transmits over cellular |
| **Parser (L1)** | Raw hex bytes | Normalized JSON | Protocol decoding, unit conversion, IMEI→vehicle_id mapping |
| **Supabase (L2)** | JSON record | Database row + WebSocket event | INSERT trigger fires alert generation |
| **Angular (L3)** | WebSocket payload | UI state update | Signal updates, threshold checks, alert creation |

---

## 4. Component Architecture

### 4.1 Layer 0 — Ghost Fleet Simulator

**Reference:** `PRD-Ghost-Fleet-Simulator.md`

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                           Ghost Fleet Simulator Architecture                        │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                              ghost-fleet-simulator                            │  │
│  │                                                                              │  │
│  │  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐              │  │
│  │  │  Config        │───▶│  Packet         │───▶│  TCP           │              │  │
│  │  │  Manager       │    │  Builder        │    │  Client       │              │  │
│  │  │                │    │                 │    │               │              │  │
│  │  │  • YAML config │    │  • Login (0x01) │    │  • connect()  │              │  │
│  │  │  • Env vars    │    │  • Data (0x22)  │    │  • send()     │              │  │
│  │  │  • CLI flags   │    │  • Heartbeat    │    │  • reconnect  │              │  │
│  │  │  • Scenarios   │    │    (0x23)       │    │               │              │  │
│  │  └────────────────┘    └────────────────┘    └───────┬────────┘              │  │
│  │                                                        │                       │  │
│  │                                                        │ TCP Port 5050         │  │
│  │                                                        ▼                       │  │
│  │  ┌────────────────────────────────────────────────────────────────────────┐    │  │
│  │  │                         Network Output                                 │    │  │
│  │  │   Host: localhost (dev) | staging.railway.app (staging)              │    │  │
│  │  │   Port: 5050                                                           │    │  │
│  │  │   Protocol: TCP (raw binary, no TLS on device side)                   │    │  │
│  │  └────────────────────────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                           Simulation Modes                                    │  │
│  │                                                                              │  │
│  │   single        Multi-vehicle      Scenario-based      Scenario playback    │  │
│  │   ────────      ──────────────      ───────────────      ─────────────────   │  │
│  │   1 vehicle     N simultaneous      Deterministic       Replay recorded     │  │
│  │   TCP conn      TCP connections      state transitions     packet sequences    │  │
│  │                                                                              │  │
│  │   Config:                                                                  │  │
│  │   ──────                                                                  │  │
│  │   vehicle:                                                                 │  │
│  │     id: "ghost-vehicle-01"                                                │  │
│  │     lat: 37.7749                                                          │  │
│  │     lng: -122.4194                                                        │  │
│  │   telemetry:                                                              │  │
│  │     interval_seconds: 5                                                   │  │
│  │     jitter_ms: 500                                                        │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Key Interfaces:**
| Interface | Direction | Format | Description |
|---|---|---|---|
| TCP to Parser | Outbound | Raw hex (binary) | Login + Data + Heartbeat packets |
| Config File | Inbound | YAML | Simulation parameters |
| CLI Args | Inbound | String | Override config values |

### 4.2 Layer 1 — TCP Ingestion Server

**Reference:** `PRD-Layer1-Ingestion-Server.md`

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                         Layer 1: TCP Ingestion Server                              │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                          Node.js Process (Railway)                            │  │
│  │                                                                              │  │
│  │  ┌──────────────────────────────────────────────────────────────────────┐    │  │
│  │  │                        net.Server                                     │    │  │
│  │  │                                                                      │    │  │
│  │  │   ┌────────────────────────────────────────────────────────────┐    │    │  │
│  │  │   │                    Connection Handler                       │    │    │  │
│  │  │   │                                                             │    │    │  │
│  │  │   │   Session Map (in-memory)                                   │    │    │  │
│  │  │   │   ┌─────────────────────────────────────────────────────┐  │    │    │  │
│  │  │   │   │  sessionId → { deviceId, protocol, socket,         │  │    │    │  │
│  │  │   │   │                   connectedAt, lastPacketAt }       │  │    │    │  │
│  │  │   │   └─────────────────────────────────────────────────────┘  │    │    │  │
│  │  │   │                                                             │    │    │  │
│  │  │   │   State Machine per connection:                            │    │    │  │
│  │  │   │   ┌───────────┐                                             │    │    │  │
│  │  │   │   │ CONNECTED │                                             │    │    │  │
│  │  │   │   └─────┬─────┘                                             │    │    │  │
│  │  │   │         │ socket 'data'                                     │    │    │  │
│  │  │   │         ▼                                                   │    │    │  │
│  │  │   │   ┌───────────┐                                             │    │    │  │
│  │  │   │   │WAIT_LOGIN │                                             │    │    │  │
│  │  │   │   └─────┬─────┘                                             │    │    │  │
│  │  │   │         │ recv login (0x01)                                 │    │    │  │
│  │  │   │         ▼                                                   │    │    │  │
│  │  │   │   ┌───────────┐                                             │    │    │  │
│  │  │   │   │AUTHENTICATED│ ──── send ack (0x81)                      │    │    │  │
│  │  │   │   └─────┬─────┘                                             │    │    │  │
│  │  │   │         │ recv data (0x22)                                  │    │    │  │
│  │  │   │         ▼                                                   │    │    │  │
│  │  │   │   ┌───────────┐                                             │    │    │  │
│  │  │   │   │DATA_ACTIVE│ ──────────────┐                             │    │    │  │
│  │  │   │   └───────────┘               │                             │    │    │  │
│  │  │   │                                │                             │    │    │  │
│  │  │   │   Process each packet:         │                             │    │    │  │
│  │  │   │                                ▼                             │    │    │  │
│  │  │   │   ┌─────────────────────────────────────────────────────┐    │    │    │  │
│  │  │   │   │              Packet Parser                          │    │    │    │  │
│  │  │   │   │                                                     │    │    │    │  │
│  │  │   │   │   1. Find start bytes (0x78 0x78)                   │    │    │    │  │
│  │  │   │   │   2. Validate length + checksum                      │    │    │    │  │
│  │  │   │   │   3. Extract fields per protocol                    │    │    │    │  │
│  │  │   │   │   4. Normalize units (°C, V, RPM)                   │    │    │    │  │
│  │  │   │   │   5. Output: { lat, lng, speed, temp, voltage,      │    │    │    │  │
│  │  │   │   │          rpm, dtc_codes, device_imei, timestamp }   │    │    │    │  │
│  │  │   │   └─────────────────────────────────────────────────────┘    │    │    │  │
│  │  │   │                                │                             │    │    │  │
│  │  │   │                                │                             │    │    │  │
│  │  │   │                                ▼                             │    │    │  │
│  │  │   │   ┌─────────────────────────────────────────────────────┐    │    │    │  │
│  │  │   │   │              Supabase Client                         │    │    │    │  │
│  │  │   │   │                                                     │    │    │    │  │
│  │  │   │   │   POST /rest/v1/telemetry_logs                      │    │    │    │  │
│  │  │   │   │   Headers: apikey, Authorization (service role)     │    │    │    │  │
│  │  │   │   │   Body: { vehicle_id, lat, lng, ... }               │    │    │    │  │
│  │  │   │   │                                                     │    │    │    │  │
│  │  │   │   │   On failure:                                       │    │    │    │  │
│  │  │   │   │   • Retry with exponential backoff                  │    │    │    │  │
│  │  │   │   │   • Queue in-memory (max 1000 items)                │    │    │    │  │
│  │  │   │   │   • Alert after 3 consecutive failures              │    │    │    │  │
│  │  │   │   └─────────────────────────────────────────────────────┘    │    │    │  │
│  │  └──────────────────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                              Protocol Support                                 │  │
│  │                                                                              │  │
│  │   Alibaba SinoTrack                    Micodus Protocol                      │  │
│  │   ─────────────────                    ──────────────────                     │  │
│  │   • Login: 0x01                       • Similar structure                    │  │
│  │   • Data: 0x22                       • Same packet types                     │  │
│  │   • Heartbeat: 0x23                  • May differ in field offsets          │  │
│  │                                                                              │  │
│  │   Detection: Protocol version byte in login packet determines parser        │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                           Error Handling Strategy                             │  │
│  │                                                                              │  │
│  │   Malformed Packet:        │  Discard invalid bytes, log error, continue    │  │
│  │   CRC Failure:             │  Log warning, discard packet                   │  │
│  │   Supabase Down:           │  Queue messages, retry with backoff             │  │
│  │   Device Disconnect:       │  Clean up session, log disconnect              │  │
│  │   Memory Pressure:         │  Alert at >200MB, restart at >256MB           │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Key Interfaces:**
| Interface | Direction | Format | Description |
|---|---|---|---|
| TCP from Devices | Inbound | Raw hex | Login, Data, Heartbeat packets |
| Supabase REST API | Outbound | JSON | INSERT telemetry records |
| /health | Outbound | JSON | Railway health check endpoint |
| IMEI Lookup | Outbound | REST | GET /rest/v1/devices?imei=eq.xxx |

### 4.3 Layer 2 — Data Storage

**Reference:** `PRD-Layer2-Data-Storage.md`

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                         Layer 2: Data Storage (Supabase)                           │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                          PostgreSQL Schema                                     │  │
│  │                                                                              │  │
│  │   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                  │  │
│  │   │   fleets     │────▶│  vehicles    │◀────│ fleet_members│                  │  │
│  │   │              │     │              │     │              │                  │  │
│  │   │  id (UUID)   │     │  id (UUID)   │     │  fleet_id    │                  │  │
│  │   │  name        │     │  fleet_id    │     │  user_id     │                  │  │
│  │   │  owner_id    │     │  vin         │     │  role        │                  │  │
│  │   │  settings    │     │  make/model  │     └──────────────┘                  │  │
│  │   └──────────────┘     │  year/status │            │                          │  │
│  │                        └──────┬───────┘            │                          │  │
│  │                               │                    │                          │  │
│  │                               │ references         │ references               │  │
│  │                               ▼                    ▼                          │  │
│  │   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                  │  │
│  │   │   alerts     │◀────│telemetry_logs│     │    users     │                  │  │
│  │   │              │     │              │     │              │                  │  │
│  │   │  id          │     │  id          │     │  id          │                  │  │
│  │   │  vehicle_id  │     │  vehicle_id  │     │  email       │                  │  │
│  │   │  severity    │     │  lat/lng     │     │  display_name│                  │  │
│  │   │  code        │     │  temp/voltage│     │  role        │                  │  │
│  │   │  message     │     │  rpm         │     │  preferences │                  │  │
│  │   │  acknowledged│     │  dtc_codes  │     └──────────────┘                  │  │
│  │   └──────────────┘     │  timestamp   │                                    │  │
│  │                        └──────────────┘                                    │  │
│  │                               ▲                                              │  │
│  │                               │                                              │  │
│  │                               │ trigger: generate_alert_if_needed            │  │
│  │                        ┌──────────────┐                                       │  │
│  │                        │telemetry_rules│                                      │  │
│  │                        │              │                                       │  │
│  │                        │  fleet_id    │                                       │  │
│  │                        │  metric      │                                       │  │
│  │                        │  operator    │                                       │  │
│  │                        │  threshold   │                                       │  │
│  │                        │  severity    │                                       │  │
│  │                        └──────────────┘                                       │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Realtime Configuration                                │  │
│  │                                                                              │  │
│  │   ALTER PUBLICATION supabase_realtime ADD TABLE telemetry_logs;              │  │
│  │   ALTER PUBLICATION supabase_realtime ADD TABLE alerts;                      │  │
│  │                                                                              │  │
│  │   ┌──────────────────────────────────────────────────────────────────────┐    │  │
│  │   │                     WebSocket Channel                                │    │  │
│  │   │                                                                      │    │  │
│  │   │   supabase.channel('telemetry-live')                                 │    │  │
│  │   │     .on('postgres_changes', {                                        │    │  │
│  │   │       event: 'INSERT',                                               │    │  │
│  │   │       schema: 'public',                                               │    │  │
│  │   │       table: 'telemetry_logs'                                        │    │  │
│  │   │     }, callback)                                                      │    │  │
│  │   │     .subscribe()                                                      │    │  │
│  │   └──────────────────────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Row Level Security (RLS)                              │  │
│  │                                                                              │  │
│  │   get_user_fleet_ids() ──▶ Returns fleet UUIDs where user is member          │  │
│  │                                                                              │  │
│  │   telemetry_logs:    SELECT ──▶ vehicle.fleet_id IN get_user_fleet_ids()      │  │
│  │   vehicles:         SELECT ──▶ fleet_id IN get_user_fleet_ids()             │  │
│  │   alerts:           SELECT ──▶ fleet_id IN get_user_fleet_ids()             │  │
│  │                                                                              │  │
│  │   Service role key bypasses RLS for ingestion server writes                 │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Key Interfaces:**
| Interface | Direction | Format | Description |
|---|---|---|---|
| REST API (telemetry_logs) | Inbound | JSON | INSERT from Parser |
| REST API (vehicles/devices) | Outbound | JSON | IMEI → vehicle_id lookup |
| WebSocket (Realtime) | Outbound | JSON | postgres_changes events |
| Service Role Key | Inbound | Header | Auth for INSERT operations |

### 4.4 Layer 3 — Angular Dashboard

**Reference:** `PRD-Layer3-Angular-Dashboard.md`

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                         Layer 3: Angular Dashboard                                  │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                          Angular Application                                  │  │
│  │                                                                              │  │
│  │   ┌──────────────────────────────────────────────────────────────────────┐   │  │
│  │   │                         AppComponent (Root)                           │   │  │
│  │   │   └── ShellComponent                                                  │   │  │
│  │   │       ├── HeaderComponent (logo, nav, user menu)                       │   │  │
│  │   │       ├── SidebarComponent (fleet list navigation)                    │   │  │
│  │   │       └── MainContentComponent (router outlet)                         │   │  │
│  │   │               │                                                       │   │  │
│  │   │               ├── /dashboard ──▶ DashboardComponent                    │   │  │
│  │   │               │       ├── VehicleGridComponent                        │   │  │
│  │   │               │       │       └── VehicleCardComponent                 │   │  │
│  │   │               │       │               ├── HealthGaugeComponent          │   │  │
│  │   │               │       │               ├── BatteryStatusComponent       │   │  │
│  │   │               │       │               └── DtcIndicatorComponent         │   │  │
│  │   │               │       └── AlertBannerComponent                         │   │  │
│  │   │               │                                                       │   │  │
│  │   │               ├── /map ────────▶ FleetMapComponent                     │   │  │
│  │   │               │       ├── LeafletMapComponent                           │   │  │
│  │   │               │       └── VehicleDetailPanelComponent                  │   │  │
│  │   │               │                                                       │   │  │
│  │   │               ├── /vehicle/:id ──▶ VehicleDetailComponent              │   │  │
│  │   │               │       ├── TelemetryChartComponent                      │   │  │
│  │   │               │       ├── DtcListComponent                              │   │  │
│  │   │               │       └── BatteryAnalysisComponent                      │   │  │
│  │   │               │                                                       │   │  │
│  │   │               └── /alerts ─────▶ AlertsComponent                       │   │  │
│  │   │                                                                       │   │  │
│  │   └──────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Core Services (Injectable Root)                        │  │
│  │                                                                              │  │
│  │   ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐        │  │
│  │   │  TelemetryService │    │  AlertService     │    │  VehicleService  │        │  │
│  │   │                  │    │                  │    │                  │        │  │
│  │   │  • subscribe()   │    │  • checkAndCreate │    │  • loadVehicles()│        │  │
│  │   │  • bufferTime()  │    │  • dismissAlert() │    │  • selectVehicle │        │  │
│  │   │  • toSignal()    │    │  • activeAlerts   │    │  • vehicles      │        │  │
│  │   │                  │    │                  │    │                  │        │  │
│  │   │  Uses:           │    │  Checks:         │    │  Uses:            │        │  │
│  │   │  • Supabase SDK │    │  • temp > 105°C  │    │  • TelemetrySvc  │        │  │
│  │   │  • RxJS buffer   │    │  • voltage < 12V │    │  • Supabase SDK │        │  │
│  │   │    Time(500)     │    │  • new DTC codes │    │                  │        │  │
│  │   └──────────────────┘    └──────────────────┘    └──────────────────┘        │  │
│  │                                                                              │  │
│  │   ┌──────────────────┐    ┌──────────────────┐                               │  │
│  │   │  FleetService    │    │ DtcTranslationSvc │                               │  │
│  │   │                  │    │                  │                               │  │
│  │   │  • getFleets()   │    │  • translate()  │                               │  │
│  │   │  • getVehicles() │    │  • getSeverity()│                               │  │
│  │   │  • addVehicle()  │    │                  │                               │  │
│  │   └──────────────────┘    └──────────────────┘                               │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Signal-Based State Management                         │  │
│  │                                                                              │  │
│  │   vehicleSignal:        Signal<Vehicle[]>                                    │  │
│  │   telemetrySignal:      Signal<Map<string, TelemetryRecord[]>>               │  │
│  │   alertsSignal:         Signal<Alert[]>                                      │  │
│  │   selectedVehicleSignal: Signal<string | null>                              │  │
│  │   connectionStatusSignal: Signal<'connected' | 'disconnected'>              │  │
│  │                                                                              │  │
│  │   Computed Signals:                                                           │  │
│  │   ─────────────────                                                           │  │
│  │   vehicleHealthScore = computed(() => { ... })  // Derives from telemetry    │  │
│  │   activeAlertCount   = computed(() => alerts().filter(a => a.status === 'active'))  │  │
│  │   vehiclesWithHealth = computed(() => { ... })  // Merges vehicles + telemetry │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Key Interfaces:**
| Interface | Direction | Format | Description |
|---|---|---|---|
| Supabase WebSocket | Inbound | JSON | Realtime telemetry subscriptions |
| REST API (Supabase) | Outbound | JSON | Vehicle/fleet CRUD operations |
| Router | Internal | Angular Router | Page navigation |
| Toast Notifications | Outbound | UI Component | Alert display |
| LocalStorage | Both | JSON | Session persistence |

---

## 5. Technology Stack Summary

### 5.1 Layer-by-Layer Stack

| Layer | Technology | Version | Provider | Purpose |
|---|---|---|---|---|
| **L0: Simulation** | Python/Node.js | 3.11+/20 | Local | Ghost Fleet packet generation |
| **L1: Ingestion** | Node.js (net module) | 20 LTS | Railway | TCP server, protocol parsing |
| **L1: Logging** | Pino | latest | npm | Structured JSON logging |
| **L2: Database** | PostgreSQL | 15 | Supabase | Primary data store |
| **L2: Realtime** | Postgres Changes | — | Supabase | WebSocket pub/sub |
| **L2: Auth** | Supabase Auth | — | Supabase | User authentication |
| **L3: Frontend** | Angular | 19/20 | Vercel | SPA framework |
| **L3: State** | Angular Signals | — | Angular | Reactive state management |
| **L3: HTTP** | RxJS + Supabase SDK | latest | npm | Async operations |
| **L3: UI** | Angular Material | 19 | npm | Component library |
| **L3: Maps** | Leaflet.js | 1.9 | npm | Fleet map visualization |

### 5.2 Infrastructure Stack

| Component | Technology | Tier | Monthly Cost (MVP) |
|---|---|---|---|
| Frontend Hosting | Vercel | Hobby | $0 |
| Backend Hosting | Railway | Usage-based | $0-5 |
| Database | Supabase | Free | $0 |
| Domain/DNS | Vercel | — | $0 (via Vercel) |
| SIM Connectivity | Telnyx/Monogoto | Pay-as-you-go | ~$1.50/SIM |
| **Total Infrastructure** | | | **$0-7** |

### 5.3 Development Tool Stack

| Tool | Purpose |
|---|---|
| GitHub | Source control, CI/CD |
| GitHub Actions | Automation (tests, deployments) |
| Supabase CLI | Local database development |
| Angular CLI | Frontend scaffolding and builds |
| Node.js 20 | Local parser development |
| Docker (optional) | Containerized development |
| VS Code | IDE (recommended) |

---

## 6. Infrastructure Topology

### 6.1 Development Environment

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              Development Environment                                  │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌────────────────────────────────────────────────────────────────────────────┐   │
│   │                            Developer's Machine                              │   │
│   │                                                                             │   │
│   │   ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐   │   │
│   │   │   Ghost Fleet    │     │   Angular Dev    │     │   Supabase CLI   │   │   │
│   │   │   Simulator     │     │   Server         │     │   (Local DB)     │   │   │
│   │   │                  │     │                  │     │                  │   │   │
│   │   │  localhost:5050  │     │  localhost:4200  │     │  localhost:54321 │   │   │
│   │   │       │          │     │       │          │     │       │          │   │   │
│   │   │       │          │     │       │          │     │       │          │   │   │
│   │   │       │          │     │       │          │     │       │          │   │   │
│   │   │       ▼          │     │       │          │     │       │          │   │   │
│   │   │  TCP connect ────┼─────┼───────┘          │     │       │          │   │   │
│   │   │                 │     │                  │     │       │          │   │   │
│   │   │                 │     │                  │     │       │          │   │   │
│   │   └─────────────────┘     └──────────────────┘     └───────┼──────────┘   │   │
│   │                                                           │              │   │
│   │                                                           │ REST/WS      │   │
│   │                                                           ▼              │   │
│   │                                                   ┌──────────────────┐   │   │
│   │                                                   │  Supabase Local  │   │   │
│   │                                                   │  (Docker)        │   │   │
│   │                                                   └──────────────────┘   │   │
│   │                                                                             │   │
│   └────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│   Ports:                                                                            │
│   • 4200: Angular dev server (ng serve)                                            │
│   • 5050: TCP Ingestion server (Node.js)                                           │
│   • 54321: Supabase Studio (supabase start)                                       │
│   • 54322: PostgreSQL (supabase start)                                             │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Staging Environment

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                                Staging Environment                                 │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌────────────────────────────────────────────────────────────────────────────┐   │
│   │                              Railway (Staging)                               │   │
│   │                                                                             │   │
│   │   ┌────────────────────────────────────────────────────────────────────┐  │   │
│   │   │                      Railway Container                               │  │   │
│   │   │                                                                     │  │   │
│   │   │   Node.js TCP Ingestion Server                                      │  │   │
│   │   │   • PORT: 5050 (TCP exposed)                                        │  │   │
│   │   │   • Memory: 256MB limit                                            │  │   │
│   │   │   • Replicas: 1                                                     │  │   │
│   │   │   • Health check: GET /health                                      │  │   │
│   │   │                                                                     │  │   │
│   │   │   Environment Variables:                                            │  │   │
│   │   │   • SUPABASE_URL: https://xxx.supabase.co                         │  │   │
│   │   │   • SUPABASE_SERVICE_ROLE_KEY: [staging key]                       │  │   │
│   │   │   • LOG_LEVEL: debug                                                │  │   │
│   │   │                                                                     │  │   │
│   │   └────────────────────────────────────────────────────────────────────┘  │   │
│   │                                                                             │   │
│   │   Railway Public URL: vitalsdrive-staging.railway.app:5050                │   │
│   └────────────────────────────────────────────────────────────────────────────┘   │
│                                           │                                          │
│                                           │ REST API                                 │
│                                           ▼                                          │
│   ┌────────────────────────────────────────────────────────────────────────────┐   │
│   │                        Supabase (Staging Project)                            │   │
│   │                                                                             │   │
│   │   Project URL: https://[staging-ref].supabase.co                          │   │
│   │   Database Size: < 500MB (Free tier)                                       │   │
│   │   Realtime: Enabled on telemetry_logs, alerts                               │   │
│   │   RLS: Enforced                                                            │   │
│   │                                                                             │   │
│   └────────────────────────────────────────────────────────────────────────────┘   │
│                                           │                                          │
│                                           │ WebSocket                                │
│                                           ▼                                          │
│   ┌────────────────────────────────────────────────────────────────────────────┐   │
│   │                            Vercel (Staging Deploy)                            │   │
│   │                                                                             │   │
│   │   Project: vitalsdrive-dashboard-staging                                   │   │
│   │   Branch: develop                                                           │   │
│   │   URL: vitalsdrive-staging.vercel.app                                      │   │
│   │   Environment: staging                                                      │   │
│   │                                                                             │   │
│   └────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│   ┌────────────────────────────────────────────────────────────────────────────┐   │
│   │                          Ghost Fleet (CI/CD)                                 │   │
│   │                                                                             │   │
│   │   GitHub Actions runner sends test packets to Railway staging endpoint      │   │
│   │   Validates: Parser → Supabase → Angular pipeline                           │   │
│   │                                                                             │   │
│   └────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Production Environment

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                               Production Environment                                │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌────────────────────────────────────────────────────────────────────────────┐   │
│   │                              Railway (Production)                             │   │
│   │                                                                             │   │
│   │   ┌────────────────────────────────────────────────────────────────────┐  │   │
│   │   │                      Railway Container                               │  │   │
│   │   │                                                                     │  │   │
│   │   │   Node.js TCP Ingestion Server                                      │  │   │
│   │   │   • PORT: 5050 (TCP exposed)                                        │  │   │
│   │   │   • Memory: 256MB limit                                            │  │   │
│   │   │   • Replicas: 1 (MVP), 2+ (Scale)                                  │  │   │
│   │   │   • Health check: GET /health                                      │  │   │
│   │   │   • Restart policy: ON_FAILURE (max 10 retries)                    │  │   │
│   │   │                                                                     │  │   │
│   │   │   DNS: ingestion.vitalsdrive.com → Railway IPv4                    │  │   │
│   │   │                                                                     │  │   │
│   │   └────────────────────────────────────────────────────────────────────┘  │   │
│   └────────────────────────────────────────────────────────────────────────────┘   │
│                                           │                                          │
│   ┌───────────────────────────────────────┼───────────────────────────────────────┐
│   │                                       │ REST API                                 │
│   │                                       ▼                                          │
│   │   ┌────────────────────────────────────────────────────────────────────────┐ │
│   │   │                        Supabase (Production)                            │ │
│   │   │                                                                        │ │
│   │   │   Project: vitalsdrive-prod                                           │ │
│   │   │   Tier: Pro ($25/mo) when scaling beyond free tier                     │ │
│   │   │   Database: 8GB (Pro tier)                                             │ │
│   │   │   Bandwidth: 50GB (Pro tier)                                           │ │
│   │   │   Connections: 200 (Pro tier)                                          │ │
│   │   │   Point-in-time Recovery: 30 days                                      │ │
│   │   │   Daily Backups: 7 days retention                                       │ │
│   │   │                                                                        │ │
│   │   │   RLS: Strictly enforced                                              │ │
│   │   │   Realtime: Enabled on telemetry_logs, alerts                           │ │
│   │   │                                                                        │ │
│   │   └────────────────────────────────────────────────────────────────────────┘ │
│   │                                       │                                        │
│   │                                       │ WebSocket                               │
│   │                                       ▼                                        │
│   │   ┌────────────────────────────────────────────────────────────────────────┐ │
│   │   │                          Vercel (Production)                           │ │
│   │   │                                                                        │ │
│   │   │   Project: vitalsdrive-dashboard                                       │ │
│   │   │   Branch: main                                                         │ │
│   │   │   URL: vitalsdrive.com (configured)                                    │ │
│   │   │   Tier: Pro ($20/mo) when traffic increases                            │ │
│   │   │                                                                        │ │
│   │   └────────────────────────────────────────────────────────────────────────┘ │
│   │                                                                             │
│   └─────────────────────────────────────────────────────────────────────────────┘
│                                                                                     │
│   ┌────────────────────────────────────────────────────────────────────────────┐   │
│   │                         OBD2 Devices (Internet)                              │   │
│   │                                                                             │   │
│   │   SMS Configured Devices:                                                  │   │
│   │   adminip123456 [Railway_IP] 5050                                          │   │
│   │                                                                             │   │
│   │   Device Types: SinoTrack/Micodus 4G OBD2                                  │   │
│   │   Connectivity: 4G LTE Cat-1                                               │   │
│   │   Protocol: SinoTrack binary over TCP                                       │   │
│   │                                                                             │   │
│   └────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Security Architecture

### 7.1 Network Security

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                               Network Security Layers                               │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌────────────────────────────────────────────────────────────────────────────┐   │
│   │                           OBD2 Device → Railway                             │   │
│   │                                                                             │   │
│   │   4G Cellular Network                                                       │   │
│   │   ├── Device encrypts before transmission (hardware-dependent)             │   │
│   │   ├── VitalsDrive cannot control device-side encryption                    │   │
│   │   └── Railway receives raw TCP (no TLS termination for device protocol)    │   │
│   │                                                                             │   │
│   │   Railway Network                                                           │   │
│   │   ├── Railway provides network isolation                                   │   │
│   │   ├── Only TCP port 5050 exposed to internet                                │   │
│   │   └── Container-to-container traffic is internal                            │   │
│   │                                                                             │   │
│   │   Railway → Supabase                                                        │   │
│   │   ├── All outbound calls use HTTPS (TLS 1.2+)                             │   │
│   │   └── Supabase validates service role key                                   │   │
│   │                                                                             │   │
│   └────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│   ┌────────────────────────────────────────────────────────────────────────────┐   │
│   │                           Browser → Vercel → Supabase                        │   │
│   │                                                                             │   │
│   │   Browser → Vercel                                                           │   │
│   │   ├── HTTPS enforced (TLS 1.3)                                              │   │
│   │   ├── HSTS headers configured                                              │   │
│   │   └── Security headers: X-Frame-Options, X-Content-Type-Options, etc.     │   │
│   │                                                                             │   │
│   │   Vercel → Supabase (via Browser)                                          │   │
│   │   ├── Supabase WebSocket (wss://)                                          │   │
│   │   ├── Anon key sent with each request                                     │   │
│   │   └── RLS enforces row-level access based on user context                 │   │
│   │                                                                             │   │
│   └────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Data Security

| Data Category | At Rest | In Transit | Access Control |
|---|---|---|---|
| **Telemetry Logs** | Supabase (encrypted) | HTTPS/WSS | RLS (fleet membership) |
| **Vehicle Data** | Supabase (encrypted) | HTTPS/WSS | RLS (fleet membership) |
| **User Credentials** | Supabase Auth (bcrypt) | HTTPS | Supabase Auth only |
| **Supabase Keys** | Railway env vars (encrypted) | N/A | Service role only |
| **Device IMEI** | Supabase | Raw TCP | Parser only (service role) |
| **Raw Packets (debug)** | Memory only | N/A | Debug flag only |

### 7.3 Authentication Architecture

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                         Authentication Flow (Supabase Auth)                        │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌──────────────┐                      ┌──────────────┐                           │
│   │   Browser    │                      │   Supabase   │                           │
│   │              │                      │     Auth      │                           │
│   └──────┬───────┘                      └──────┬───────┘                           │
│          │                                     │                                   │
│          │  1. Sign up / Sign in               │                                   │
│          │─────────────────────────────────────▶                                   │
│          │                                     │                                   │
│          │  2. Verify email / OTP               │                                   │
│          │◀─────────────────────────────────────                                   │
│          │                                     │                                   │
│          │  3. Receive JWT (access token)      │                                   │
│          │◀─────────────────────────────────────                                   │
│          │                                     │                                   │
│          │                                     │                                   │
│   ┌──────┴───────┐                      ┌──────┴───────┐                           │
│   │   Browser    │                      │   Supabase   │                           │
│   │   Storage    │                      │   Database   │                           │
│   │              │                      │              │                           │
│   │  access_token │                      │  auth.users  │                           │
│   │  refresh_token│                      │  users table │                           │
│   └──────────────┘                      └──────────────┘                           │
│                                                                                     │
│   Dashboard Request Flow:                                                          │
│   ───────────────────────                                                          │
│   1. Browser includes JWT in Authorization header                                 │
│   2. Supabase validates JWT signature                                              │
│   3. RLS policies use auth.uid() to filter rows                                   │
│   4. Anon key used for initial load; JWT used for user-specific data               │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.4 Input Validation

| Layer | Validation | Location |
|---|---|---|
| **Parser (L1)** | CRC checksum validation | `validatePacket()` in `parser/src/` |
| **Parser (L1)** | Packet length bounds | `if (buffer.length < length + 4)` |
| **Parser (L1)** | GPS coordinate range (-90 to 90, -180 to 180) | Supabase CHECK constraint |
| **Parser (L1)** | Protocol byte validation | State machine transitions |
| **Supabase (L2)** | CHECK constraints on tables | `lat CHECK (lat >= -90 AND lat <= 90)` |
| **Supabase (L2)** | NOT NULL constraints | Table definitions |
| **Supabase (L2)** | UUID validation | Foreign key constraints |
| **Angular (L3)** | TypeScript interfaces | `Vehicle`, `TelemetryRecord`, `Alert` |
| **Angular (L3)** | Input sanitization | Angular DomSanitizer where needed |

---

## 8. Deployment Architecture

### 8.1 Deployment Pipeline

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                               CI/CD Deployment Pipeline                             │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                           GitHub Repository                                   │  │
│  │                                                                              │  │
│  │   main ◀─── develop                                                          │  │
│  │    │           │                                                              │  │
│  │    │           │ PR merge                                                     │  │
│  │    │           │                                                              │  │
│  │    ▼           ▼                                                              │  │
│  │  Production   Staging Deploy                                                  │  │
│  │  Pipeline     Pipeline                                                        │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Staging Deployment Flow                                │  │
│  │                                                                              │  │
│  │   push to develop ──▶ GitHub Actions ──▶ Vercel Preview ──▶ Dashboard Test   │  │
│  │                                  │                                             │  │
│  │                                  ├──▶ Railway Staging ──▶ Ghost Fleet Test   │  │
│  │                                  │                                             │  │
│  │                                  └──▶ Supabase Staging (auto)                  │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                        Production Deployment Flow                              │  │
│  │                                                                              │  │
│  │   merge to main ──▶ GitHub Actions ──▶ Vercel Production ──▶ Smoke Tests      │  │
│  │                            │                                                     │  │
│  │                            ├──▶ Railway Production ──▶ Health Check            │  │
│  │                            │                                                     │  │
│  │                            └──▶ Supabase Production (auto)                      │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Railway Deployment

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              Railway Deployment                                     │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   railway.toml (Parser)                                                            │
│   ────────────────────                                                             │
│   [build]                                                                           │
│     builder = "nixpacks"                                                            │
│     builderVersion = "1"                                                            │
│                                                                                      │
│   [deploy]                                                                          │
│     numReplicas = 1                                                                 │
│     restartPolicyType = "ON_FAILURE"                                                │
│     restartPolicyMaxRetries = 10                                                    │
│     healthCheckPath = "/health"                                                     │
│                                                                                      │
│   [[ports]]                                                                         │
│     port = 5050                                                                      │
│     protocol = "TCP"                                                                │
│                                                                                      │
│   [environment]                                                                      │
│     PORT = "5050"                                                                   │
│     NODE_ENV = "production"                                                         │
│                                                                                      │
│   Deployment Steps:                                                                  │
│   ────────────────                                                                   │
│   1. Connect GitHub repo to Railway project                                         │
│   2. Configure build command: npm install && npm start                              │
│   3. Set environment variables:                                                     │
│      • SUPABASE_URL                                                                 │
│      • SUPABASE_SERVICE_ROLE_KEY                                                    │
│      • LOG_LEVEL                                                                    │
│   4. Railway auto-deploys on main branch push                                        │
│   5. Health check verifies /health endpoint                                         │
│                                                                                      │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 8.3 Vercel Deployment

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                               Vercel Deployment                                     │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   vercel.json                                                                          │
│   ──────────                                                                           │
│   {                                                                                   │
│     "framework": "angular",                                                          │
│     "buildCommand": "ng build",                                                       │
│     "outputDirectory": "dist/vitalsdrive-dashboard/browser",                          │
│     "devCommand": "ng serve",                                                         │
│     "installCommand": "npm ci",                                                       │
│     "regions": ["iad1"]                                                               │
│   }                                                                                   │
│                                                                                      │
│   Environment Variables (Vercel Dashboard):                                           │
│   ─────────────────────────────────────────                                          │
│   • SUPABASE_URL                                                                     │
│   • SUPABASE_ANON_KEY                                                                │
│   • NG_APP_VERSION (auto-populated)                                                   │
│                                                                                      │
│   Deployment Steps:                                                                   │
│   ────────────────                                                                    │
│   1. Connect GitHub repo to Vercel project                                          │
│   2. Configure build settings (framework: Angular)                                  │
│   3. Set environment variables                                                       │
│   4. Vercel auto-deploys on main branch push                                         │
│   5. Preview deployments for PRs                                                     │
│                                                                                      │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Development Workflow

### 9.1 Local Development Setup

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              Local Development Setup                                │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   Prerequisites:                                                                      │
│   ───────────────                                                                     │
│   • Node.js 20 LTS                                                                  │
│   • Python 3.11+ (for Ghost Fleet)                                                  │
│   • Supabase CLI: npm install -g supabase                                            │
│   • Angular CLI: npm install -g @angular/cli                                        │
│   • Git                                                                           │
│                                                                                     │
│   Setup Steps:                                                                       │
│   ────────────                                                                       │
│   1. Clone repository:                                                               │
│      git clone https://github.com/vitalsdrive/vitalsdrive.git                        │
│      cd vitalsdrive                                                                 │
│                                                                                      │
│   2. Start Supabase local:                                                          │
│      supabase init                                                                  │
│      supabase start                                                                 │
│      (Provides local PostgreSQL on port 54322, Studio on 54321)                    │
│                                                                                      │
│   3. Apply migrations:                                                              │
│      supabase db reset  # Applies migrations from supabase/migrations/             │
│                                                                                      │
│   4. Configure environment:                                                         │
│      cp parser/.env.example parser/.env                                             │
│      cp dashboard/src/environments/environment.ts.example \\                                                │
│        dashboard/src/environments/environment.ts                                     │
│      (Update SUPABASE_URL and keys for local)                                       │
│                                                                                      │
│   5. Start parser:                                                                  │
│      cd parser                                                                       │
│      npm install                                                                     │
│      npm run dev          # Starts on localhost:5050                                │
│                                                                                      │
│   6. Start dashboard:                                                                │
│      cd dashboard                                                                     │
│      npm install                                                                     │
│      npm start          # Starts on localhost:4200                                  │
│                                                                                      │
│   7. Run Ghost Fleet:                                                                │
│      cd simulator                                                                     │
│      pip install ghost-fleet-simulator                                               │
│      ghost-fleet --config ./config/dev.yaml                                         │
│                                                                                      │
│   8. Verify:                                                                       │
│      • Dashboard: http://localhost:4200                                              │
│      • Ghost packets flowing: Check parser terminal output                           │
│      • Data in Supabase: http://localhost:54321                                     │
│                                                                                      │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Testing Strategy

| Test Type | Location | Framework | CI Integration |
|---|---|---|---|
| **Parser Unit Tests** | `parser/__tests__/` | Jest | Required |
| **Parser Integration Tests** | `parser/__tests__/integration/` | Jest + Supertest | Required |
| **Ghost Fleet Validation** | `simulator/tests/` | Python pytest | Required |
| **Angular Unit Tests** | `dashboard/src/__tests__/` | Jasmine/Karma | Required |
| **Angular Component Tests** | `dashboard/src/__tests__/` | Angular TestBed | Required |
| **Angular E2E Tests** | `dashboard/e2e/` | Playwright | Required (main only) |
| **Supabase DB Tests** | `supabase/tests/` | SQL + pgTAP | Optional |

### 9.3 Code Quality

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                                Code Quality Gates                                   │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   Pre-commit Hooks:                                                                 │
│   ───────────────                                                                     │
│   • ESLint (dashboard) - Angular TypeScript                                          │
│   • Prettier (dashboard) - Code formatting                                          │
│   • Rustfmt (parser) - Rust formatting (if applicable)                             │
│   • Python Black (simulator) - Python formatting                                   │
│                                                                                     │
│   Lint Commands:                                                                     │
│   ──────────────                                                                     │
│   Parser:   npm run lint                                                             │
│   Dashboard: npm run lint                                                           │
│                                                                                     │
│   Typecheck Commands:                                                                │
│   ───────────────────                                                                 │
│   Parser:   npm run typecheck (if TypeScript)                                        │
│   Dashboard: npm run typecheck                                                      │
│                                                                                     │
│   Bundle Size Budgets (Dashboard):                                                   │
│   ────────────────────────────────                                                   │
│   Initial bundle: 500kb warning, 1mb error                                           │
│   Component bundles: 50kb warning, 100kb error                                     │
│                                                                                      │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Environment Configuration

### 10.1 Environment Variables

#### Parser (Layer 1)

| Variable | Type | Default | Required | Description |
|---|---|---|---|---|
| `PORT` | number | 5050 | Yes | TCP listen port |
| `SUPABASE_URL` | string | — | Yes | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | string | — | Yes | Service role key for writes |
| `LOG_LEVEL` | string | info | No | debug, info, warn, error |
| `CONNECTION_TIMEOUT_MS` | number | 300000 | No | 5 minutes |
| `KEEPALIVE_INTERVAL_MS` | number | 60000 | No | TCP keep-alive probe |
| `SUPABASE_QUEUE_MAX_SIZE` | number | 1000 | No | Max queued writes |
| `DEBUG_RAW_PACKETS` | boolean | false | No | Log hex of each packet |

#### Dashboard (Layer 3)

| Variable | Type | Default | Required | Description |
|---|---|---|---|---|
| `SUPABASE_URL` | string | — | Yes | Supabase project URL |
| `SUPABASE_ANON_KEY` | string | — | Yes | Anonymous key for client |
| `NG_APP_VERSION` | string | — | No | Auto-populated build version |

### 10.2 Environment Matrix

| Environment | Parser Host | Supabase Project | Vercel Project | Purpose |
|---|---|---|---|---|
| **Local** | localhost:5050 | localhost:54322 | localhost:4200 | Individual dev |
| **Staging** | vitalsdrive-staging.railway.app:5050 | [staging-ref].supabase.co | vitalsdrive-staging.vercel.app | Integration testing |
| **Production** | ingestion.vitalsdrive.com | [prod-ref].supabase.co | vitalsdrive.com | Live system |

---

## 11. Monitoring and Observability

### 11.1 Logging Strategy

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                                 Logging Architecture                                 │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   Parser (L1) — Pino JSON Logger                                                     │
│   ─────────────────────────────                                                     │
│   {                                                                            │
│     "level": "info",                                                             │
│     "time": 1743245520123,                                                       │
│     "pid": 1,                                                                    │
│     "hostname": "container-abc123",                                              │
│     "msg": "Packet processed",                                                   │
│     "sessionId": "sess-uuid",                                                    │
│     "deviceImei": "861234567890123",                                             │
│     "packetType": "data",                                                        │
│     "parserMs": 2,                                                               │
│     "supabasePushMs": 45                                                         │
│   }                                                                            │
│                                                                                      │
│   Key Log Events:                                                                  │
│   ───────────────                                                                   │
│   • Server started (info)                                                          │
│   • Device connected/disconnected (info)                                           │
│   • Login packet received (debug)                                                  │
│   • Data packet received (debug)                                                   │
│   • CRC validation failed (warn)                                                   │
│   • Supabase push success/failure (debug/error)                                    │
│   • Queue depth warning (warn)                                                      │
│                                                                                      │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Metrics to Track

| Layer | Metric | Type | Alert Threshold |
|---|---|---|---|
| **Parser** | `tcp_connections_active` | gauge | > 450 (90% capacity) |
| **Parser** | `packets_parse_errors_total` | counter | > 1% of total |
| **Parser** | `supabase_push_duration_ms` | histogram | p95 > 500ms |
| **Parser** | `supabase_push_failures_total` | counter | > 0 |
| **Parser** | `queue_depth` | gauge | > 800 |
| **Supabase** | `database_size` | gauge | > 400MB (80%) |
| **Supabase** | `connection_count` | gauge | > 40 |
| **Supabase** | `queries_per_second` | gauge | > 50 |

### 11.3 Alerting

| Alert | Condition | Severity | Action |
|---|---|---|---|
| Server down | /health returns 5xx | P1 | Railway auto-restart + alert |
| Supabase unreachable | 3 consecutive failures | P2 | Slack alert |
| Queue depth critical | > 900 for 5min | P2 | Slack alert |
| Parse error spike | > 5% error rate | P3 | Log review |
| Connection count high | > 450 | P3 | Capacity planning |
| Database size warning | > 80% | P2 | Review data retention |

---

## 12. Disaster Recovery

### 12.1 Backup Strategy

| Backup Type | Frequency | Retention | RTO | RPO |
|---|---|---|---|---|
| Supabase Automated Daily | Daily | 7 days | < 1 hour | 24 hours |
| Supabase Point-in-time | Continuous | 30 days | < 1 hour | 1 hour |
| Supabase Manual | On-demand | Until deleted | Depends | Depends |
| Railway Container | On redeploy | N/A | < 30s | N/A |

### 12.2 Recovery Procedures

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              Recovery Procedures                                    │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   Supabase Point-in-Time Recovery:                                                  │
│   ────────────────────────────────                                                  │
│   1. Navigate to Supabase Dashboard > Database > Backups                          │
│   2. Select Point-in-time recovery                                                │
│   3. Choose recovery timestamp                                                    │
│   4. Create new project or restore to existing                                    │
│   5. Update Railway/Dashboard environment variables if new project                │
│                                                                                     │
│   Railway Container Recovery:                                                       │
│   ───────────────────────────                                                       │
│   1. Railway auto-restarts on crash                                               │
│   2. For full outage, Railway migrates to new host (< 2min)                       │
│   3. Public IP may change — update device SMS if needed                          │
│                                                                                     │
│   Full Region Outage (Railway):                                                    │
│   ─────────────────────────────                                                     │
│   1. Deploy to new region                                                          │
│   2. Update DNS/SMS for new IP                                                    │
│   3. Devices reconnect automatically                                              │
│   4. Expected RTO: < 30 minutes                                                   │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 12.3 Data Retention

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                               Data Retention Policy                                 │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   telemetry_logs:                                                                   │
│   ──────────────                                                                    │
│   • Default retention: 90 days                                                     │
│   • Automatic cleanup: Monthly via cleanup_old_telemetry() function               │
│   • Archive before delete: Optional export to Supabase Storage (CSV)               │
│                                                                                     │
│   alerts:                                                                            │
│   ──────                                                                             │
│   • Retained indefinitely until manually resolved                                  │
│   • Annual review to archive old resolved alerts                                   │
│                                                                                     │
│   audit_logs (if enabled):                                                          │
│   ──────────────────────                                                            │
│   • Retained for compliance as needed                                              │
│   • Typically 1-2 years for SOC2/HIPAA compliance                                  │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. Scaling Strategy

### 13.1 Scaling Triggers

| Metric | Threshold to Investigate | Threshold to Act |
|---|---|---|
| Database size | > 300MB | > 400MB (80% of free tier) |
| Monthly bandwidth | > 1GB | > 1.5GB |
| Connection errors | > 2/month | > 5/month |
| Query latency (p99) | > 300ms | > 500ms |
| Active vehicles | — | > 10 (single parser) |

### 13.2 Scaling Timeline

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              Scaling Timeline                                       │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   MVP Phase (1-10 vehicles):                                                       │
│   ──────────────────────────                                                       │
│   • Railway: 1 replica (~$0-3/mo)                                                  │
│   • Supabase: Free tier (500MB, 2GB bandwidth)                                    │
│   • Vercel: Hobby tier ($0)                                                        │
│   • Total: ~$0/mo infrastructure                                                   │
│                                                                                     │
│   Growth Phase (10-50 vehicles):                                                   │
│   ──────────────────────────────                                                   │
│   • Railway: 1-2 replicas (~$5/mo)                                                 │
│   • Supabase: Free → Pro ($25/mo when limits approached)                         │
│   • Vercel: Hobby → Pro ($20/mo)                                                    │
│   • Total: ~$25-50/mo infrastructure                                               │
│                                                                                     │
│   Scale Phase (50-200 vehicles):                                                   │
│   ──────────────────────────────                                                    │
│   • Railway: 2 replicas + vertical scaling                                         │
│   • Supabase: Pro tier                                                            │
│   • Vercel: Pro tier                                                               │
│   • Consider: Redis for session state (if multi-replica)                         │
│   • Total: ~$50-100/mo infrastructure                                              │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 13.3 Connection Scaling

| Vehicles | Parser Replicas | Session Management | Notes |
|---|---|---|---|
| 1-50 | 1 | In-memory Map | Simple MVP approach |
| 50-100 | 2 | TCP sticky sessions OR Redis | Connection deduplication |
| 100-500 | 2+ | Redis shared state | Horizontal scaling ready |

---

## 14. Integration Points

### 14.1 External Integrations (Future)

| Integration | Priority | Complexity | Timeline |
|---|---|---|---|
| **SMS Alerts (Twilio)** | P1 | Low | Post-MVP |
| **Email Alerts (SendGrid)** | P1 | Low | Post-MVP |
| **Maintenance Scheduling** | P2 | Medium | Phase 2+ |
| **Fuel Analytics** | P2 | Medium | Phase 2+ |
| **Fleet Management Software** | P3 | High | Future |
| **ERP Integration** | P3 | High | Future |
| **Driver Behavior Monitoring** | P4 | Medium | Future |

### 14.2 API Design (Future)

For future API integrations:

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              External API Design                                    │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   RESTful Endpoints (Future):                                                       │
│   ───────────────────────                                                           │
│   GET    /api/v1/fleets                 — List user's fleets                      │
│   GET    /api/v1/fleets/:id              — Fleet details                           │
│   GET    /api/v1/fleets/:id/vehicles     — List fleet vehicles                     │
│   GET    /api/v1/vehicles/:id/telemetry  — Vehicle telemetry history              │
│   GET    /api/v1/vehicles/:id/alerts     — Vehicle alerts                          │
│   POST   /api/v1/webhooks                — Inbound webhook integration             │
│                                                                                     │
│   Authentication:                                                                    │
│   ───────────────                                                                     │
│   • API keys for service-to-service                                                │
│   • OAuth 2.0 for user-facing integrations                                          │
│   • Rate limiting: 1000 req/min per API key                                        │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 15. Development Roadmap Dependencies

### 15.1 Phase Dependencies

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              Development Phases                                    │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   Phase 1: Simulation Build (Week 1)                                               │
│   ════════════════════════════════                                                  │
│   Dependencies: None                                                               │
│   Blockers: None                                                                   │
│   Deliverables:                                                                     │
│   • Supabase project created, schema deployed                                      │
│   • Railway TCP server deployed and accessible                                     │
│   • Ghost Fleet simulator running and generating packets                            │
│   • End-to-end pipeline verified (Ghost → Parser → Supabase)                      │
│                                                                                     │
│   Phase 2: Angular Dashboard (Days 4-7)                                             │
│   ═══════════════════════════════════════                                           │
│   Dependencies: Phase 1 complete                                                    │
│   Blockers: Live data required for testing                                         │
│   Deliverables:                                                                     │
│   • Dashboard deployed to Vercel staging                                           │
│   • Real-time telemetry display working                                             │
│   • Alert thresholds implemented                                                   │
│   • Fleet map displaying vehicle positions                                         │
│                                                                                     │
│   Phase 3: Hardware Integration (Week 2-4)                                           │
│   ═════════════════════════════════════════                                         │
│   Dependencies: Phase 1 & 2 stable                                                  │
│   Blockers: Hardware procurement (2-4 weeks from Alibaba)                           │
│   Deliverables:                                                                     │
│   • Real 4G OBD2 device connected                                                   │
│   • Protocol parser validated against real device                                  │
│   • Dashboard rendering real vehicle data                                           │
│   • Zero frontend code changes from Phase 2                                        │
│                                                                                     │
│   Phase 4: Pilot Program (Month 2)                                                  │
│   ═════════════════════════════                                                     │
│   Dependencies: Phase 3 complete                                                    │
│   Blockers: First external user                                                    │
│   Deliverables:                                                                     │
│   • Pilot user operational with 1-3 vehicles                                       │
│   • Structured feedback collection                                                │
│   • Bug fixes and feature gap documentation                                        │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 15.2 Critical Path

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                               Critical Path                                         │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   Start ──▶ Supabase Setup ──▶ Railway Deploy ──▶ Ghost Fleet Test ──▶ Phase 1 ✓    │
│                          │                                                           │
│                          │                                                           │
│                          ▼                                                           │
│                   Railway TCP ──▶ Ghost Fleet ──▶ Verify ──▶ Phase 2              │
│                      Server         Connection      Pipeline                        │
│                          │                                                           │
│                          │                                                           │
│                          ▼                                                           │
│                   Parser Code ──▶ Supabase ──▶ Angular ──▶ Phase 3                │
│                      (real device)   Insert       Dashboard                       │
│                          │                                                           │
│                          │                                                           │
│                          ▼                                                           │
│                   Bug Fixes ──▶ Pilot ──▶ Phase 4                                  │
│                                       User                                         │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Appendix A: Document References

| Document | Location | Description |
|---|---|---|
| Main PRD | `docs/VitalsDrive_PRD.md` | Product overview, problem statement, MVP features |
| Layer 1 PRD | `docs/PRD-Layer1-Ingestion-Server.md` | TCP ingestion server detailed specs |
| Layer 2 PRD | `docs/PRD-Layer2-Data-Storage.md` | Supabase schema, RLS, API design |
| Layer 3 PRD | `docs/PRD-Layer3-Angular-Dashboard.md` | Angular dashboard detailed specs |
| Ghost Fleet PRD | `docs/PRD-Ghost-Fleet-Simulator.md` | Simulator protocol and usage |

---

## Appendix B: Acronyms and Glossary

| Term | Definition |
|---|---|
| **CAN** | Controller Area Network — vehicle bus standard |
| **DTC** | Diagnostic Trouble Code — OBD-II fault codes |
| **IMEI** | International Mobile Equipment Identity — 15-digit device identifier |
| **OBD2** | On-Board Diagnostics II — vehicle self-diagnostics standard |
| **PID** | Parameter ID — OBD-II data parameter identifier |
| **RLS** | Row Level Security — PostgreSQL access control |
| **RTO** | Recovery Time Objective — time to restore service |
| **RPO** | Recovery Point Objective — maximum acceptable data loss |
| **SLA** | Service Level Agreement — uptime commitment |
| **TCP** | Transmission Control Protocol — connection-oriented network protocol |
| **TLS** | Transport Layer Security — cryptographic protocol |
| **UUID** | Universally Unique Identifier — 128-bit label format |
| **WebSocket** | Bidirectional communication protocol over TCP |

---

**Document Version:** 1.0  
**Last Updated:** March 2026  
**Authors:** VitalsDrive Engineering  
**Review Cycle:** Quarterly or upon significant architectural changes
