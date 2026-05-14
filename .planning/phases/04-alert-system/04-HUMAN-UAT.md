---
status: partial
phase: 04-alert-system
source: [04-VERIFICATION.md]
started: 2026-05-15T00:00:00Z
updated: 2026-05-15T00:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. DB trigger smoke — 4 alert rows + dedup
expected: INSERT into telemetry_logs with voltage=11.7, temp=105, dtc_codes=['P0300','P0301'] produces BATTERY_WARNING + COOLANT_WARNING + P0300 + P0301 (4 rows). Repeating the INSERT produces 0 new rows.
result: [pending]

### 2. Realtime toast fires on threshold breach
expected: With dashboard open, inserting a telemetry row crossing a threshold (e.g. voltage=11.7) causes a toast notification to appear within ~2 seconds — no page refresh required.
result: [pending]

### 3. Realtime delivery after fleet change (CR-01 fix)
expected: If a user joins a new fleet mid-session (or fleet list changes), alerts for that new fleet arrive via Realtime without a page reload. The stale-closure bug is fixed — the effect() re-subscribes when fleetService.fleets() changes.
result: [pending]

### 4. Fleet map marker severity colors + pulse
expected: Vehicles with critical unacknowledged alerts show red (#ef4444) markers. Warning alerts → yellow (#eab308). No unacknowledged alerts → green (#84cc16). Markers pulse when unacknowledged alerts are present.
result: [pending]

### 5. Header bell badge live count
expected: Bell icon shows unacknowledged count badge. Badge is hidden when count=0. Clicking bell navigates to /alerts. Count updates without page refresh when a new alert arrives via Realtime.
result: [pending]

## Summary

total: 5
passed: 0
issues: 0
pending: 5
skipped: 0
blocked: 0

## Gaps
