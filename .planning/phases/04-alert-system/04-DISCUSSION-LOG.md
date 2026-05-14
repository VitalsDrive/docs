# Phase 4: Alert System - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-14
**Phase:** 4-alert-system
**Areas discussed:** Alert trigger location, DTC code lookup, Threshold configurability, Alert display style, Map marker alert state, Nav notification indicator, Alert dedup window, Alert history & retention, Email/push notifications, DTC alert grouping

---

## Alert Trigger Location

| Option | Description | Selected |
|--------|-------------|----------|
| DB trigger (AFTER INSERT on telemetry_logs) | Fires on every insert, writes to alerts table, always active regardless of dashboard state | ✓ |
| Client-side computed | Angular computed() in VehicleService, only fires when dashboard open | |
| Hybrid (DB trigger + client hint) | DB trigger creates records, client also highlights inline | |

**User's choice:** DB trigger
**Notes:** User asked for detailed explanation of how the trigger approach works (transaction flow, performance at scale, dedup mechanism). After explanation, confirmed DB trigger approach.

---

## DTC Code Lookup

| Option | Description | Selected |
|--------|-------------|----------|
| Static JSON bundled in dashboard | ~200KB JSON, instant lookup, no DB query, works offline | ✓ |
| Postgres DTC lookup table | Queryable, updatable without deploy, but adds DB round-trip | |
| Show code only, no translation | Display raw code, no plain-English | |

**User's choice:** Static JSON bundled in dashboard
**Notes:** P-codes only (P0xxx, P1xxx) — matches FMC003 device output, ~2000 codes.

---

## Threshold Configurability

| Option | Description | Selected |
|--------|-------------|----------|
| Hardcoded defaults only | Fixed constants in trigger, telemetry_rules table stays empty for MVP | ✓ |
| Configurable via telemetry_rules | Trigger reads per-fleet rules, requires backoffice UI | |
| Hardcoded + telemetry_rules override | Defaults with per-fleet overrides if row exists | |

**User's choice:** Hardcoded defaults only (MVP)
**Thresholds confirmed:** Battery warning <11.8V / critical <11.5V. Coolant warning >100°C / critical >110°C. Any non-empty dtc_codes array triggers alert.

---

## Alert Display Style

| Option | Description | Selected |
|--------|-------------|----------|
| Toast + /alerts list page | New alerts trigger toast, /alerts shows persistent paginated list with acknowledgment | ✓ |
| Toast only | Transient toasts, no history | |
| /alerts list page only | Persistent list, no toasts | |

**User's choice:** Toast + /alerts list page
**Acknowledgment:** Per-alert acknowledgment (Acknowledge button per row). Acknowledged alerts stay visible but dimmed.
**Vehicle card badges:** Severity-colored badge with alert count — confirmed.

---

## Map Marker Alert State

| Option | Description | Selected |
|--------|-------------|----------|
| Color-coded by severity | Red=critical, yellow=warning, green=healthy | ✓ + pulse |
| Pulse/icon overlay only | Normal marker + small pulsing overlay | |
| No map change | Map markers unchanged | |

**User's choice:** Color-coded by severity AND pulsing animation
**Notes:** User specifically requested both color coding AND pulsing — "color coded by civility and pulsing".

---

## Nav Notification Indicator

| Option | Description | Selected |
|--------|-------------|----------|
| Bell icon with count badge | Bell icon in shell header, updates via Realtime, links to /alerts | ✓ |
| Count only, no bell icon | Simple red badge number | |
| No nav indicator | No shell-level indicator | |

**User's choice:** Bell icon with count badge

---

## Alert Dedup Window

| Option | Description | Selected |
|--------|-------------|----------|
| 1 hour | Low battery won't re-alert until acknowledged or 1 hour passes | ✓ |
| 30 minutes | More frequent re-alerts, noisier | |
| 4 hours | Very quiet, risk of missing recurrence | |

**User's choice:** 1 hour

---

## Alert History & Retention

| Option | Description | Selected |
|--------|-------------|----------|
| Last 7 days | Paginated, fast, relevant | ✓ |
| Last 30 days | More history | |
| All time | Full history, needs date filter | |

**User's choice:** Last 7 days (paginated)

---

## DTC Alert Grouping

| Option | Description | Selected |
|--------|-------------|----------|
| One alert per DTC code | 3 codes = 3 separate alerts, individual acknowledgment | ✓ |
| One combined alert | All codes in one alert row | |

**User's choice:** One alert per DTC code

---

## Email/Push Notifications

| Option | Description | Selected |
|--------|-------------|----------|
| Defer — dashboard-only for Phase 4 | NOTF-01/02/03 are v2 requirements, future phase | ✓ |
| Include basic email notification | Adds scope via Supabase Edge Function or Resend | |

**User's choice:** Defer to future phase

---

## Claude's Discretion

- Exact SQL trigger function structure and PL/pgSQL implementation
- Angular component architecture for AlertsPageComponent and AlertListItemComponent
- Realtime subscription management strategy
- Pagination approach for /alerts (offset vs cursor-based)
- RLS policy SQL for alerts table

## Deferred Ideas

- Email notifications (NOTF-01) — high user value, deferred to Phase 6 or future milestone
- Push notifications (NOTF-02/03) — deferred to future phase
- Per-fleet configurable thresholds via telemetry_rules — table exists, deferred post-MVP
- Bulk alert acknowledgment (acknowledge all / by vehicle)
- Alert history beyond 7 days in UI
