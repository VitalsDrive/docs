---
phase: 02-auth-fleet-management
plan: "06"
subsystem: dashboard
tags: [invite, fleet-members, auth0, angular, supabase]
dependency_graph:
  requires: [02-05]
  provides: [invite-redemption, fleet-member-insert]
  affects: [app.routes.ts, fleet_members, org_invites]
tech_stack:
  added: []
  patterns: [signal-state, standalone-component, lazy-route]
key_files:
  created:
    - packages/dashboard/src/app/features/join/join.component.ts
    - packages/dashboard/src/app/features/join/join.component.html
    - packages/dashboard/src/app/features/join/join.component.scss
  modified:
    - packages/dashboard/src/app/app.routes.ts
decisions:
  - "Public /join route (no authGuard on route) — component handles unauthenticated redirect internally to preserve returnUrl"
  - "Auth0 sub accessed via currentUser().id (user.sub stored as id in MockUser computed signal)"
  - "fleet_members insert includes organization_id column (confirmed from migration 006 schema)"
  - "Conflict on duplicate membership (23505) treated as success — idempotent join"
metrics:
  duration: "~15 minutes"
  completed: "2026-05-13"
  tasks_completed: 1
  tasks_total: 1
  files_created: 3
  files_modified: 1
---

# Phase 2 Plan 06: Invite Redemption (/join) Summary

**One-liner:** `/join?token=` route validates org_invites token, inserts fleet_members row with invite.role, marks single-use tokens consumed, redirects to /onboarding/org.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Build JoinComponent — token validation, fleet_members insert, redirect | 0df28e5 | join.component.ts, join.component.html, join.component.scss, app.routes.ts |

## What Was Built

`JoinComponent` at `/join?token=<uuid>` implements the full D-22 invite redemption flow:

1. Reads `token` from `ActivatedRoute.queryParams`
2. Queries `org_invites` with expiry filter (`gt('expires_at', now)`)
3. Rejects single-use tokens where `used_at` is already set
4. Checks `authService.isAuthenticated()` — redirects to `/login?returnUrl=/join?token=...` if not authenticated
5. Gets Auth0 sub from `authService.currentUser().id` (user.sub stored as id in MockUser)
6. Queries first fleet for the org via `fleets.organization_id = invite.org_id`
7. Inserts `fleet_members` row: `{ user_id, fleet_id, organization_id, role: invite.role }` — role is never 'owner'
8. On duplicate membership (pg error 23505), continues as success (idempotent)
9. Sets `used_at = now()` on single-use tokens after successful insert
10. Sets `success = true`, then redirects to `/onboarding/org` after 1500ms

**Route:** Registered as public (no authGuard) in `app.routes.ts` before the protected shell. Unauthenticated redirect is handled inside the component to preserve the `returnUrl`.

**UI states:** Loading (spinner + "Joining your team…"), Error (error_outline icon + message + Back to Home), Success (check_circle icon + "You've joined!" + "Redirecting to your fleet…").

## Verification

```
Build: PASSED (0 new errors, 1 pre-existing unrelated warning)
grep org_invites join.component.ts: FOUND (line 45)
grep fleet_members join.component.ts: FOUND (line 80)
grep used_at join.component.ts: FOUND (lines 54, 104)
grep "'join'" app.routes.ts: FOUND (line 35)
```

## Threat Model Coverage

| Threat | Mitigation Applied |
|--------|--------------------|
| T-02-06-01: Token enumeration | Token is crypto.randomUUID (128-bit), expires 7d, single-use invalidated |
| T-02-06-02: Role elevation via insert | Role read from org_invites.role — never 'owner' by DB CHECK constraint |
| T-02-06-03: Unauthenticated redemption | isAuthenticated() checked before any insert; redirects to /login |
| T-02-06-04: Token reuse | used_at set immediately after insert; SELECT filters used_at IS NULL |
| T-02-06-05: Expired token replay | expires_at > now() filter in SELECT query |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical Field] Added organization_id to fleet_members insert**
- **Found during:** Task 1
- **Issue:** Plan spec showed `{ user_id, fleet_id, role }` but migration 006 confirms `organization_id` is a required FK column with NOT NULL-equivalent enforcement via FK constraint
- **Fix:** Added `organization_id: invite.org_id` to the fleet_members insert payload
- **Files modified:** join.component.ts

**2. [Rule 2 - Missing Error Handling] Added graceful handling of duplicate membership**
- **Found during:** Task 1
- **Issue:** If user clicks invite link twice, the second insert would fail with pg error 23505; plan did not specify behavior
- **Fix:** Check `memberErr.code !== '23505'`; treat duplicate as success (idempotent join)
- **Files modified:** join.component.ts

## Known Stubs

None — all data flows are wired to live Supabase queries.

## Phase 2 Completion

This is the final plan in Phase 2 (Auth & Fleet Management). All 6 plans complete:
- 02-01: Auth0 RLS migration
- 02-02: Auth0 Angular integration
- 02-03: Dashboard auth layer fixes
- 02-04: Vehicle CRUD service
- 02-05: Onboarding step 3 + invite generator
- 02-06: Invite redemption (/join) ← this plan

D-22 fully implemented: invite links generated by Plan 05 are now redeemable via Plan 06.

## Self-Check: PASSED

- packages/dashboard/src/app/features/join/join.component.ts: FOUND
- packages/dashboard/src/app/features/join/join.component.html: FOUND
- packages/dashboard/src/app/features/join/join.component.scss: FOUND
- packages/dashboard/src/app/app.routes.ts: modified (join route added)
- Commit 0df28e5: FOUND
