# VitalsDrive

## What This Is

VitalsDrive is a real-time vehicle telemetry platform that monitors fleet health through OBD2 devices. 4G OBD2 devices send TCP data to a Node.js parser, which stores telemetry in Supabase. An Angular dashboard reads from Supabase in real-time, providing fleet operators with live GPS, DTC alerts, battery voltage, and coolant temperature monitoring.

## Core Value

Prevent expensive vehicle failures — DTC codes, dead batteries, and overheating engines — before they cause breakdowns. Fleet Health, Simplified.

## Requirements

### Validated

- Existing TCP ingestion server (Teltonika Codec 8 Extended protocol) deployed on Railway
- Supabase database schema with vehicles, telemetry, and alerts tables
- Angular dashboard with Auth0 authentication, live fleet map, and real-time subscriptions
- Ghost fleet simulator for testing
- Auth0 → JWT exchange via NestJS auth service

### Active

- [ ] DTC alert system with specific codes and plain-English explanations
- [ ] Battery health monitoring with voltage threshold alerts
- [ ] Coolant temperature alerting for overheating detection
- [ ] Live fleet map with real-time vehicle positions
- [ ] Vehicle health scoring and alert counts
- [ ] Multi-user fleet management (org roles, allowlists)
- [ ] Onboarding flow for new devices and users

### Out of Scope

- Remote DTC clearing — deferred to Diagnostic+ tier (requires device firmware support)
- Fuel analytics — deferred to Diagnostic+ tier
- Maintenance scheduling — deferred to Diagnostic+ tier
- AI predictive scoring — deferred to Predictive tier
- Driver behavior tracking — not core to vitals monitoring
- Geofencing — complex, not aligned with "3 alerts" MVP

## Context

- 5 separate git repos: docs, dashboard, parser, simulator, supabase (+ auth-service)
- Supabase production project: `odwctmlawibhaclptsew`
- Auth0 domain: `ronbiter.auth0.com`
- Parser deployed on Railway port 5050
- Dashboard deployed on Vercel
- Simulator emulates FMC003 Teltonika devices for testing
- Angular 21 with standalone components, signals-based reactivity
- Each package is independently deployable

## Constraints

- **Tech stack**: Teltonika FMC003 devices with Codec 8 Extended protocol — parser must handle this binary format
- **Data flow**: Real-time via Supabase subscriptions, not polling
- **Multi-repo**: 5 repos require coordinated changes across packages
- **Deployment**: Parser (Railway), Dashboard (Vercel), Supabase (hosted), Simulator (local/Railway)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Supabase as database | Real-time subscriptions built-in, reduces infra complexity | — Active |
| Auth0 for auth | SDK handles OIDC flow, internal JWT exchange for API calls | — Active |
| Angular 21 standalone components | Modern Angular, no NgModules, signals instead of NgRx | — Active |
| Teltonika Codec 8 Extended protocol | FMC003 device protocol, replaces initial SinoTrack assumption | ✓ Confirmed |
| Microservices per package | Each independently deployable, no shared code between packages | ✓ Confirmed |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-14 — Phase 3 (Live Fleet Dashboard) complete: resource()-driven VehicleService, real-time Leaflet fleet map with stale-state markers, dashboard loading/empty states, connection pill + reconnect toast. Migration 012 (telemetry RLS fix + get_latest_telemetry RPC) deployed. FLEET-01/02/03 validated. 5 items pending human browser UAT (03-HUMAN-UAT.md).*
