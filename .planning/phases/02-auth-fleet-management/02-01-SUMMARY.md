---
phase: 02-auth-fleet-management
plan: "01"
subsystem: supabase
tags: [migration, auth0, rls, schema, org_invites]
dependency_graph:
  requires: []
  provides: [schema-009, org_invites_table, auth0-compatible-users-rls]
  affects: [auth-service, dashboard]
tech_stack:
  added: []
  patterns: [idempotent-migration, jwt-sub-rls, token-invite]
key_files:
  created:
    - packages/supabase/migrations/009_phase2_auth_schema.sql
  modified: []
decisions:
  - "Use auth.jwt()->>'sub' not auth.uid() for all Auth0-user RLS — auth.uid() returns NULL for Auth0 users since they have no row in auth.users"
  - "Drop organizations.owner_id FK to auth.users before retyping to TEXT — FK must be removed before column type change"
  - "org_invites.created_by is TEXT (Auth0 sub), not UUID FK — no FK to auth.users since Auth0 users are not in that table"
  - "Dual SELECT policy on org_invites: owner-scoped read AND anonymous token-based read for invite landing page"
metrics:
  duration: "15m"
  completed_date: "2026-05-13"
  tasks_completed: 1
  tasks_total: 3
  files_created: 1
  files_modified: 0
---

# Phase 2 Plan 01: Phase 2 Auth Schema Migration Summary

Single idempotent migration dropping Auth0-incompatible FK constraints, retyping UUID columns to TEXT for Auth0 sub storage, adding org_invites table, and fixing RLS policies to use JWT sub claim.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Write migration 009 — resolve all Phase 2 schema blockers | 6f58c3a | packages/supabase/migrations/009_phase2_auth_schema.sql |

## Tasks Pending (Checkpoints)

| Task | Type | Status |
|------|------|--------|
| 2 | checkpoint:human-action | Awaiting — push migration to production Supabase |
| 3 | checkpoint:human-action | Awaiting — configure Auth0 as JWT provider in Supabase dashboard |

## What Was Built

Migration 009 with 7 sections:

1. **FK removal** — `ALTER TABLE users DROP CONSTRAINT IF EXISTS users_id_fkey` removes the `auth.users` FK that blocked every Auth0 first-login insert
2. **Provider default** — `external_auth_provider` default changed from `'clerk'` to `'auth0'`
3. **vehicles columns** — `nickname VARCHAR(100)` added; `vin NOT NULL` dropped
4. **UUID → TEXT** — `fleet_members.user_id` and `organizations.owner_id` retyped to TEXT via `USING` cast; prerequisite FK on `organizations.owner_id` dropped first
5. **org_invites table** — token, org_id, created_by (TEXT), type (single-use/multi-use), role, expires_at, used_at; RLS enabled; 3 indexes
6. **org_invites RLS** — 4 policies: owner read, owner insert, owner update, anonymous token read (invite landing page); service role bypass
7. **users RLS** — dropped `USING (true)` and `id = auth.uid()` policies; replaced with `external_auth_id = (auth.jwt()->>'sub')` for select/update; service role bypass

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] organizations.owner_id FK must be dropped before TYPE change**
- **Found during:** Task 1, Section 4
- **Issue:** `organizations.owner_id UUID NOT NULL REFERENCES auth.users(id)` — cannot retype UUID column to TEXT while FK constraint exists; Postgres rejects the ALTER
- **Fix:** Added `ALTER TABLE organizations DROP CONSTRAINT IF EXISTS organizations_owner_id_fkey` before the TYPE change
- **Files modified:** packages/supabase/migrations/009_phase2_auth_schema.sql
- **Commit:** 6f58c3a

**2. [Rule 2 - Missing] fleet_members.user_id FK to users must be dropped before TYPE change**
- **Found during:** Task 1, Section 4
- **Issue:** Migration 001 created `fleet_members.user_id UUID REFERENCES users(id) ON DELETE CASCADE` — same problem; FK blocks retype
- **Fix:** Added `ALTER TABLE fleet_members DROP CONSTRAINT IF EXISTS fleet_members_user_id_fkey` before the TYPE change
- **Files modified:** packages/supabase/migrations/009_phase2_auth_schema.sql
- **Commit:** 6f58c3a

**3. [Rule 2 - Missing] Drop migration 004 broken users RLS policy**
- **Found during:** Task 1, Section 7
- **Issue:** Migration 004 created `"Users can view their own profile" USING (id = auth.uid())` — `auth.uid()` is NULL for Auth0 users; this policy silently hides every user row
- **Fix:** Added `DROP POLICY IF EXISTS "Users can view their own profile"` and `"Users can update own profile"` to the cleanup list in Section 7
- **Files modified:** packages/supabase/migrations/009_phase2_auth_schema.sql
- **Commit:** 6f58c3a

## Known Stubs

None — this plan produces only DDL/SQL. No UI rendering paths.

## Threat Flags

No new network endpoints introduced. RLS policies match threat register mitigations:
- T-02-01-01: org_invites INSERT requires `owner_id = auth.jwt()->>'sub'` ✓
- T-02-01-02: service_role bypass scoped to `auth.jwt()->>'role' = 'service_role'` ✓
- T-02-01-03: `external_auth_id = auth.jwt()->>'sub'` uses verified Auth0 JWT sub ✓
- T-02-01-04: org_invites SELECT restricted to org owner; token is `gen_random_uuid()` output ✓

## Self-Check

- [x] `packages/supabase/migrations/009_phase2_auth_schema.sql` exists
- [x] Contains `DROP CONSTRAINT IF EXISTS users_id_fkey`
- [x] Contains `ALTER COLUMN user_id TYPE TEXT`
- [x] Contains `ADD COLUMN IF NOT EXISTS nickname`
- [x] Contains `ALTER COLUMN vin DROP NOT NULL`
- [x] Contains `CREATE TABLE IF NOT EXISTS org_invites`
- [x] Contains `external_auth_id = (auth.jwt()->>'sub')`
- [x] Begins with `BEGIN;` and ends with `COMMIT;`
- [x] Commit 6f58c3a exists in packages/supabase git log
