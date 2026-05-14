---
status: partial
phase: 04-alert-system
source: [04-VERIFICATION.md]
started: 2026-05-15T00:00:00Z
updated: 2026-05-15T00:00:00Z
---

## Current Test

Automated via Chrome DevTools MCP + Supabase MCP (2026-05-15).
Vehicle: UAT FreshCar (`17e05156-1731-4469-b907-8c663a0a1862`), fleet `703f3545-5767-4567-83b0-99e1deded53a`.

## Tests

### 1. DB trigger smoke — 4 alert rows + dedup
expected: INSERT into telemetry_logs with voltage=11.7, temp=105, dtc_codes=['P0300','P0301'] produces BATTERY_WARNING + COOLANT_WARNING + P0300 + P0301 (4 rows). Repeating the INSERT produces 0 new rows.
result: PASS ✅
notes: Confirmed via Supabase MCP SQL. First INSERT created exactly 4 rows (BATTERY_WARNING, COOLANT_WARNING, P0300, P0301). Repeat INSERT with identical values created 0 new rows — dedup guard (1-hour window + acknowledged=false check) worked correctly.

### 2. Realtime toast fires on threshold breach
expected: With dashboard open, inserting a telemetry row crossing a threshold (e.g. voltage=11.7) causes a toast notification to appear within ~2 seconds — no page refresh required.
result: FAIL ❌
notes: `postgres_changes` INSERT events are not delivered to the browser. Root cause: `_authedClient` in `supabase.service.ts` forwards the VD JWT via `global.headers.Authorization` (REST path only). The Realtime WebSocket connection uses the anon key with no user JWT, so `auth.jwt()->>sub` returns NULL in the Supabase Realtime RLS context — events are silently dropped. Attempted fixes (setAuth, accessToken async fn) caused channel state "errored", likely due to JWT claim incompatibility (Auth0-issued JWT lacks `role: authenticated` claim expected by Supabase Realtime Phoenix auth). Reverted `supabase.service.ts` to original. Fleet_id server-side filter added as defense-in-depth but does not resolve the RLS issue. See gap: Realtime JWT forwarding.

### 3. Realtime delivery after fleet change (CR-01 fix)
expected: If a user joins a new fleet mid-session (or fleet list changes), alerts for that new fleet arrive via Realtime without a page reload. The stale-closure bug is fixed — the effect() re-subscribes when fleetService.fleets() changes.
result: FAIL ❌
notes: Same root cause as Test 2 — Realtime postgres_changes events not delivered regardless of fleet change. The `effect()` re-subscription logic in `alert.service.ts` is correctly implemented (CR-01 code fix confirmed), but cannot be validated end-to-end until the Realtime JWT issue is resolved.

### 4. Fleet map marker severity colors + pulse
expected: Vehicles with critical unacknowledged alerts show red (#ef4444) markers. Warning alerts → yellow (#eab308). No unacknowledged alerts → green (#84cc16). Markers pulse when unacknowledged alerts are present.
result: PASS ✅
notes: Confirmed via DOM inspection of `.map-marker` SVG elements. UAT FreshCar: fill `#eab308` (warning yellow) with CSS class `pulse`. UAT StaleCar (stale telemetry, no active alerts): fill `#5a4530` (stale brown, no pulse). Color and pulse logic in `fleet-map.component.ts` `getMarkerColor()` / `getVehicleState()` confirmed correct.

### 5. Header bell badge live count
expected: Bell icon shows unacknowledged count badge. Badge is hidden when count=0. Clicking bell navigates to /alerts. Count updates without page refresh when a new alert arrives via Realtime.
result: PARTIAL PASS ⚠️
notes: Badge correctly reads from `unacknowledgedCount` (DB-backed, not in-memory `activeAlertCount`). Badge hidden at 0. Clicking bell navigates to /alerts. Badge count decrements correctly on acknowledge (optimistic update via CR-03 fix — `acknowledged_at` set in signal). Live Realtime update without page refresh: NOT tested — blocked by same Realtime JWT issue as Tests 2 & 3.

## Summary

total: 5
passed: 2
issues: 1
pending: 0
skipped: 0
blocked: 2

## Gaps

### GAP-01: Realtime JWT forwarding (blocks Tests 2, 3, and partial 5)
- **Root cause**: `SupabaseService._authedClient` sends VD JWT via `Authorization` header (REST only). Supabase Realtime WebSocket connects with anon key; no user JWT reaches Realtime's RLS evaluation context.
- **Why setAuth failed**: The VD JWT (issued by NestJS auth-service) may be missing `role: authenticated` claim or other fields Supabase Realtime's Phoenix auth requires. Raw Auth0 JWT confirmed incompatible (`iss: https://ronbiter.auth0.com/`, no `role`).
- **Fix options**:
  1. Investigate VD JWT claims from `auth-service` — ensure `role: authenticated` is present; pass token via `accessToken: async () => vdToken` in `createClient` options
  2. Replace `postgres_changes` with Supabase Broadcast channel (DB trigger calls `pg_notify` or edge function broadcasts) — bypasses RLS/JWT entirely
  3. Disable RLS on `alerts` table Realtime publication and rely solely on server-side `fleet_id` filter (security trade-off)
