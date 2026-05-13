---
phase: 02-auth-fleet-management
reviewed: 2026-05-13T10:18:00Z
depth: standard
files_reviewed: 23
files_reviewed_list:
  - packages/supabase/migrations/009_phase2_auth_schema.sql
  - packages/auth-service/src/auth/auth.service.ts
  - packages/auth-service/src/auth/auth.controller.ts
  - packages/auth-service/src/users/users.service.ts
  - packages/auth-service/src/auth/jwt.strategy.ts
  - packages/auth-service/src/auth/auth.service.spec.ts
  - packages/dashboard/src/app/core/services/auth.service.ts
  - packages/dashboard/src/app/core/interceptors/auth.interceptor.ts
  - packages/dashboard/src/app/core/guards/auth.guard.ts
  - packages/dashboard/src/app/app.routes.ts
  - packages/dashboard/src/environments/environment.ts
  - packages/dashboard/src/environments/environment.development.ts
  - packages/dashboard/src/app/core/services/vehicle.service.ts
  - packages/dashboard/src/app/core/models/vehicle.model.ts
  - packages/dashboard/src/app/features/fleet-management/fleet-management.component.ts
  - packages/dashboard/src/app/features/fleet-management/fleet-management.component.html
  - packages/dashboard/src/app/features/fleet-management/add-vehicle-dialog.component.ts
  - packages/dashboard/src/app/features/fleet-management/remove-vehicle-dialog.component.ts
  - packages/dashboard/src/app/pages/onboarding/vehicle/onboarding-vehicle.component.ts
  - packages/dashboard/src/app/features/settings/invite/invite.component.ts
  - packages/dashboard/src/app/features/settings/invite/invite.component.html
  - packages/dashboard/src/app/features/join/join.component.ts
  - packages/dashboard/src/app/features/join/join.component.html
findings:
  critical: 9
  warning: 8
  info: 3
  total: 20
fixed:
  critical: 9
  warning: 6
  info: 3
  total: 18
open:
  warning: 2
  total: 2
status: partially_fixed
fixed_at: 2026-05-13T11:10:00Z
open_issues:
  - WR-01: Refresh token rotation not implemented (design-level, out of scope for Phase 2)
  - WR-06: onboardingStepGuard double-query (design-level, non-blocking)
---

# Phase 02: Auth & Fleet Management — Code Review Report

**Reviewed:** 2026-05-13T10:18:00Z
**Depth:** standard
**Files Reviewed:** 23
**Status:** issues_found

## Summary

This phase implements Auth0 integration, JWT exchange, Supabase RLS migration, fleet vehicle CRUD, invite links, and a join flow. The overall structure is sound but there are 9 critical issues — several of which represent real security vulnerabilities or data-loss paths that must be fixed before the feature ships.

Key concern areas:
- **Hardcoded credentials** in environment files and JWT strategy
- **Token storage in localStorage** exposes JWTs to XSS
- **Race condition in signIn()** — code unreachable after loginWithRedirect()
- **RLS policy conflict** on org_invites allows unauthenticated reads of valid tokens
- **Join flow TOCTOU** — single-use token can be double-redeemed under concurrent requests
- **Invite component** does not enforce org-owner authorization before inserting
- **Refresh token reuse** — refresh tokens are never rotated or invalidated

---

## Critical Issues

### CR-01: Hardcoded Auth0 domain, audience, and JWT secret fallback in production paths

**Files:**
- `packages/auth-service/src/auth/auth.service.ts:7-8`
- `packages/auth-service/src/auth/jwt.strategy.ts:11`

**Issue:** `AUTH0_DOMAIN` and `AUTH0_AUDIENCE` fall back to hardcoded literal values when `process.env` is unset. The JWT strategy falls back to the literal string `'vitalsdrive-secret-key'`. If the service is ever deployed without env vars set (e.g., a misconfigured container, a test environment accidentally routed to production traffic), requests will be validated against a publicly known secret. The domain fallback also hardcodes a real Auth0 tenant name, leaking tenant identity in source.

```
// auth.service.ts
const AUTH0_DOMAIN = process.env.AUTH0_DOMAIN || 'ronbiter.auth0.com';   // line 7
const AUTH0_AUDIENCE = process.env.AUTH0_AUDIENCE || 'https://ronbiter.auth0.com/api/v2/';  // line 8

// jwt.strategy.ts
secretOrKey: process.env.JWT_SECRET || 'vitalsdrive-secret-key',  // line 11
```

**Fix:** Fail fast at startup if required env vars are absent. Never provide a default for secrets.

```typescript
// auth.service.ts
const AUTH0_DOMAIN = process.env.AUTH0_DOMAIN;
const AUTH0_AUDIENCE = process.env.AUTH0_AUDIENCE;
if (!AUTH0_DOMAIN || !AUTH0_AUDIENCE) {
  throw new Error('AUTH0_DOMAIN and AUTH0_AUDIENCE must be set');
}

// jwt.strategy.ts
const secret = process.env.JWT_SECRET;
if (!secret) throw new Error('JWT_SECRET must be set');
super({ ..., secretOrKey: secret });
```

---

### CR-02: Supabase anon key committed in environment files (production key exposed)

**Files:**
- `packages/dashboard/src/environments/environment.ts:8`
- `packages/dashboard/src/environments/environment.development.ts:5`

**Issue:** The production Supabase anon key (`eyJhbGci...`) is committed verbatim in both environment files. While Supabase anon keys are partially public-facing, they grant direct database access constrained only by RLS. Given that RLS policies in this migration have gaps (see CR-04, CR-05), the committed key is an amplifying risk. The production key should not appear in a source file committed to a git repository.

**Fix:** Move keys to `.env` files (git-ignored), inject at build time via Angular's `fileReplacements` or a CI environment variable substitution step. At minimum, document that `environment.ts` must never be committed with real keys.

---

### CR-03: Refresh tokens stored in localStorage — XSS gives full session persistence

**File:** `packages/dashboard/src/app/core/services/auth.service.ts:110-111`

**Issue:** Both the access token and the 7-day refresh token are written to `localStorage`. Any XSS (injected script, malicious dependency) can read them with `localStorage.getItem('vd_refresh_token')` and mint new access tokens indefinitely for 7 days without re-authenticating through Auth0. The access token alone being in localStorage is already high-risk; the refresh token makes session hijacking persistent.

```typescript
localStorage.setItem(this.TOKEN_KEY, data.accessToken);   // line 110
localStorage.setItem(this.REFRESH_KEY, data.refreshToken); // line 111
```

**Fix:** Store the refresh token in an `HttpOnly` cookie (set by the auth-service via `Set-Cookie`). Keep the access token in memory only (the `internalToken` field). The interceptor already uses `getInternalToken()` for in-memory access — extend this pattern by removing all `localStorage` write/read of refresh tokens and moving that responsibility server-side.

---

### CR-04: org_invites has conflicting SELECT policies — unauthenticated users can enumerate valid tokens

**File:** `packages/supabase/migrations/009_phase2_auth_schema.sql:354-358`

**Issue:** Two SELECT policies exist on `org_invites`:

1. `"Org owner can view invites"` — requires `o.owner_id = auth.jwt()->>'sub'` (authenticated)
2. `"Anyone can read valid invite by token"` — no auth requirement, matches any row where `expires_at > now() AND (used_at IS NULL OR type = 'multi-use')`

Policy 2 is unconditional. Any unauthenticated actor can call Supabase directly and read every valid, unexpired invite token across all organizations by scanning without a `token` filter. This exposes all outstanding invite tokens to enumeration.

```sql
CREATE POLICY "Anyone can read valid invite by token" ON org_invites
    FOR SELECT USING (
        expires_at > now()
        AND (used_at IS NULL OR type = 'multi-use')
    );
```

**Fix:** The join flow only needs to look up one specific token. Restrict this policy to require a token equality match, or enforce the filter in application code and remove the open policy entirely. The minimal safe version:

```sql
-- Remove the open policy. The join flow should use the service role
-- (via auth-service endpoint) to validate a token, not a direct anon query.
DROP POLICY "Anyone can read valid invite by token" ON org_invites;
```

Then expose a dedicated `/auth/join` endpoint in auth-service that validates the token server-side using the service role key.

---

### CR-05: Join flow TOCTOU — single-use token can be double-redeemed under concurrent requests

**File:** `packages/dashboard/src/app/features/join/join.component.ts:40-116`

**Issue:** The redemption sequence is:
1. Read token (line 40–45)
2. Check `used_at IS NULL` (line 54)
3. Insert `fleet_members` row (line 93–100)
4. Update `used_at` (line 113–116)

Steps 1–4 are non-atomic client-side reads and writes. Two concurrent requests with the same token both pass step 2 (neither has set `used_at` yet), both succeed at step 3 (the second hits the `23505` conflict and is silently swallowed as "already a member"), and both succeed at step 4. This does not cause double-membership but does demonstrate the lack of atomicity — a malicious actor could chain this with a token that targets a different user_id before step 4 executes. More importantly, the silent conflict swallow at line 104 means the user gets success even if the insert silently failed for a non-duplicate reason that happens to return `23505`.

**Fix:** Move redemption to a Supabase DB function (or auth-service endpoint) that performs the check-and-mark atomically:

```sql
CREATE OR REPLACE FUNCTION redeem_invite(p_token TEXT, p_user_id TEXT)
RETURNS TEXT  -- returns org_id on success, raises exception on failure
LANGUAGE plpgsql SECURITY DEFINER AS $$
DECLARE
  v_invite org_invites%ROWTYPE;
BEGIN
  SELECT * INTO v_invite FROM org_invites
    WHERE token = p_token
      AND expires_at > now()
      AND (used_at IS NULL OR type = 'multi-use')
    FOR UPDATE;  -- row-level lock

  IF NOT FOUND THEN
    RAISE EXCEPTION 'invalid_token';
  END IF;

  -- insert fleet_member, upsert to handle already-member
  INSERT INTO fleet_members (user_id, fleet_id, organization_id, role)
    SELECT p_user_id, f.id, v_invite.org_id, v_invite.role
    FROM fleets f WHERE f.organization_id = v_invite.org_id LIMIT 1
    ON CONFLICT DO NOTHING;

  IF v_invite.type = 'single-use' THEN
    UPDATE org_invites SET used_at = now() WHERE id = v_invite.id;
  END IF;

  RETURN v_invite.org_id;
END;
$$;
```

---

### CR-06: signIn() flow is broken — code after loginWithRedirect() is unreachable

**File:** `packages/dashboard/src/app/core/services/auth.service.ts:83-99`

**Issue:** `loginWithRedirect()` navigates the browser to Auth0. It does not return a resolved observable — the page is unloaded. Any code after `await firstValueFrom(this.auth0.loginWithRedirect())` (lines 90–91) will never execute in a normal browser flow. `getAccessTokenSilently()` and `exchangeToken()` are dead code inside `signIn()`. Token exchange must be triggered in the Auth0 callback handler (`AuthGuard` or `AppComponent`), not here.

```typescript
// Lines 88-91 — loginWithRedirect navigates away; lines 90-91 never execute
await firstValueFrom(this.auth0.loginWithRedirect());
const auth0Token = await firstValueFrom(this.auth0.getAccessTokenSilently()); // dead
await this.exchangeToken(auth0Token); // dead
```

**Fix:** Remove the dead code. Trigger token exchange in the Auth0 callback path (e.g., an `(isAuthenticated$)` subscriber in `AppComponent` or a dedicated callback route), not inside `signIn()`.

---

### CR-07: auth.guard.ts busy-polls on isLoading() — potential infinite loop

**File:** `packages/dashboard/src/app/core/guards/auth.guard.ts:11-13`

**Issue:** The guard spins on `auth.isLoading()` with a 50ms sleep loop:

```typescript
while (auth.isLoading()) {
  await new Promise(resolve => setTimeout(resolve, 50));
}
```

`isLoading` is a writable signal set to `false` at initialization and only set to `true` inside `signIn()`. If any code path sets `isLoading` to `true` and fails to reset it (e.g., an exception before `isLoading.set(false)` in the `finally` block — which doesn't always exist in all callers), the guard will spin forever, freezing navigation. There is no timeout or escape condition. All four guards in this file share the same pattern.

**Fix:** Add a timeout escape:

```typescript
const deadline = Date.now() + 5000; // 5s max wait
while (auth.isLoading() && Date.now() < deadline) {
  await new Promise(resolve => setTimeout(resolve, 50));
}
if (auth.isLoading()) {
  // Loading timed out — treat as unauthenticated
  return router.createUrlTree(['/login']);
}
```

---

### CR-08: Invite component does not verify caller is org owner before inserting invite

**File:** `packages/dashboard/src/app/features/settings/invite/invite.component.ts:44-83`

**Issue:** `generateInvite()` sets `created_by` from `this.authService.currentUser()?.id` (line 58) and inserts directly via the anon Supabase client. The RLS policy `"Org owner can create invites"` should block non-owners, but:

1. If the Supabase JWT is not correctly forwarded (e.g., `supabaseService.client` uses the anon key without the user's JWT set), the `auth.jwt()->>'sub'` will be null and the policy will silently fail or error.
2. `created_by` is set to `null` if `currentUser()` returns null (line 58: `?? null`). An unauthenticated call with `created_by = null` will still attempt the insert.
3. No client-side role check is performed before hitting the database — any authenticated user (not just owners) can attempt invite creation.

**Fix:** Add a client-side guard before the DB call, and ensure Supabase client has the user JWT set:

```typescript
async generateInvite(): Promise<void> {
  const createdBy = this.authService.currentUser()?.id;
  if (!createdBy) {
    this.error.set('You must be signed in to create invites.');
    return;
  }
  // Optionally: check isOwner signal
  if (!this.authService.isOwner()) {
    this.error.set('Only org owners can create invite links.');
    return;
  }
  // ... rest of method
}
```

Also ensure `supabaseService.client` is initialized with the user's internal JWT (not just the anon key) so RLS policies are evaluated against the correct `sub`.

---

### CR-09: Redirect after join navigates to non-existent route `/onboarding/org`

**File:** `packages/dashboard/src/app/features/join/join.component.ts:124`

**Issue:** After successful redemption, the component redirects to `/onboarding/org`:

```typescript
this.router.navigate(['/onboarding/org']);
```

The route table in `app.routes.ts` defines `/onboarding/organization` (line 60), not `/onboarding/org`. The wildcard `**` catch-all redirects to `/dashboard` (line 222), so the user lands on dashboard instead of the intended onboarding step. This silently skips the intended UX flow for newly joined users.

**Fix:**

```typescript
this.router.navigate(['/onboarding/organization']);
```

---

## Warnings

### WR-01: Refresh tokens are never rotated or revoked — stolen refresh tokens are valid for 7 days

**File:** `packages/auth-service/src/auth/auth.service.ts:71-83`

**Issue:** `refreshToken()` issues a new access token from any valid refresh token. There is no refresh token rotation (issuing a new refresh token on each use and invalidating the old one), no revocation list, and no server-side state. A stolen refresh token (from localStorage — see CR-03) is valid for the full 7-day window with no way to invalidate it short of rotating the `JWT_SECRET`.

**Fix:** Either implement refresh token rotation with a server-side store (Redis or Supabase table) tracking issued/revoked tokens, or reduce the refresh token TTL and force re-authentication through Auth0 more frequently.

---

### WR-02: `mapToUser` in UsersService hardcodes `['member']` role fallback — ignores DB roles column

**File:** `packages/auth-service/src/users/users.service.ts:106`

**Issue:**

```typescript
roles: Array.isArray(dbRow.roles) ? dbRow.roles : ['member'],
```

If the `users` table has no `roles` column (it is not defined in the migration schema), `dbRow.roles` will always be `undefined`, so every user always gets `['member']`. Role-based logic in `checkUserRoles()` (auth.service.ts:279) derives `isOwner`/`isAdmin` from these roles — both will always be `false`. The `isAdmin` signal fed to `adminGuard` is therefore always false for all users, making the `/backoffice` route permanently inaccessible.

**Fix:** Either add a `roles` column to the `users` table in a migration, or derive roles from `fleet_members.role` at query time. Do not silently fall back to a hardcoded value.

---

### WR-03: `environment.ts` is imported directly in dashboard `auth.service.ts` — wrong environment for production builds

**File:** `packages/dashboard/src/app/core/services/auth.service.ts:7`

**Issue:**

```typescript
import { environment } from "../../../environments/environment.development";
```

The import explicitly targets `environment.development.ts`, bypassing Angular's file replacement mechanism. In a production build, Angular replaces `environment.ts` with `environment.prod.ts` via `fileReplacements`. This hardcoded import always uses the development environment regardless of build target — production builds will point `authServiceUrl` at `http://localhost:3001`.

**Fix:**

```typescript
import { environment } from "../../../environments/environment";
```

---

### WR-04: `loadVehicles()` subscribes to `telemetryBatch$` on every call — subscriptions accumulate

**File:** `packages/dashboard/src/app/core/services/vehicle.service.ts:115-117`

**Issue:** Every call to `loadVehicles()` (e.g., when the user switches fleets in the dropdown) creates a new subscription to `telemetryBatch$`:

```typescript
this.telemetryService.telemetryBatch$
  .pipe(takeUntil(this.destroy$))
  .subscribe((batch) => this.processBatch(batch));
```

`takeUntil(this.destroy$)` only cancels on service destruction (never during a session). After `N` fleet switches, there are `N` active subscriptions all calling `processBatch`, processing each batch `N` times and multiplying writes to `telemetryMap`.

**Fix:** Use a `switchMap` pattern or track a subject that resets on each `loadVehicles` call:

```typescript
private readonly reloadVehicles$ = new Subject<void>();

// In loadVehicles():
this.reloadVehicles$.next(); // cancels previous subscription
this.telemetryService.telemetryBatch$
  .pipe(takeUntil(this.reloadVehicles$), takeUntil(this.destroy$))
  .subscribe((batch) => this.processBatch(batch));
```

---

### WR-05: `getVehicle()` fetches by ID without a fleet filter — any authenticated user can read any vehicle

**File:** `packages/dashboard/src/app/core/services/vehicle.service.ts:162-192`

**Issue:** The Supabase query on line 170–174 fetches the vehicle by `id` alone with no fleet-scoped filter:

```typescript
const { data, error } = await this.supabase.client
  .from('vehicles')
  .select('*')
  .eq('id', id)
  .single();
```

The org membership check (lines 178–185) is done client-side after the fetch. If RLS on `vehicles` does not restrict SELECT by org (the migration's vehicle SELECT policy was not recreated — see the migration: no `SELECT` policy for vehicles is defined), the anon client can read any vehicle row by guessing its UUID.

**Fix:** Add an inline fleet filter to the query, or ensure RLS has a `FOR SELECT` policy on `vehicles`. The migration recreates policies for INSERT, DELETE, and UPDATE on vehicles but has no SELECT policy — any authenticated user can read all vehicle rows.

---

### WR-06: `onboardingStepGuard` calls `auth.completeOnboarding()` which re-runs `initializeUserState()` unnecessarily and can cause a redirect loop

**File:** `packages/dashboard/src/app/core/guards/auth.guard.ts:102-104`

**Issue:** When fleets are found, the guard calls `await auth.completeOnboarding()` (line 102) then immediately returns `router.createUrlTree(['/dashboard'])`. `completeOnboarding()` calls `initializeUserState()` which queries `fleet_members` and `fleets` again — these are the same queries the guard just ran. This is redundant double-querying on every navigation to any onboarding route when the user is fully onboarded. It also means any navigation to `/onboarding/*` when fully onboarded triggers a redirect to `/dashboard` — but `/dashboard` is protected by `onboardingGuard` which calls... nothing recursive, so it is not a loop. However the double-query on each navigation is wasteful and fragile.

**Fix:** Remove the `auth.completeOnboarding()` call from inside the guard. The guard already has the state it needs (fleets loaded, orgs loaded). Set signals directly or let `initializeUserState` run once at app startup.

---

### WR-07: `loadInitialTelemetry()` fetches up to 500 rows with no per-vehicle limit — skews history for large fleets

**File:** `packages/dashboard/src/app/core/services/vehicle.service.ts:198-214`

**Issue:**

```typescript
.limit(500)
```

The 500-row limit is shared across all vehicles. For a fleet with 50 vehicles, each vehicle gets at most 10 records in the initial snapshot (and potentially 0 for vehicles later in the sort order). The `calculateHealthScore` and `alertService.checkAndCreateAlerts` will silently receive no telemetry for some vehicles, showing incorrect health scores of 100 (the default when `telemetry` is undefined).

**Fix:** Either increase the limit relative to fleet size, or restructure to fetch `LIMIT 1 ORDER BY timestamp DESC` per vehicle using a lateral join or an RPC function.

---

### WR-08: Edit button in fleet-management.component.html opens AddVehicleDialog instead of an edit dialog

**File:** `packages/dashboard/src/app/features/fleet-management/fleet-management.component.html:65`

**Issue:**

```html
(click)="openAddVehicleDialog()"
```

The edit (pencil) button on each vehicle card calls `openAddVehicleDialog()` which opens the Add Vehicle form with empty fields — no vehicle data is passed. Clicking edit appears to work (no crash) but discards all existing vehicle data and creates a duplicate vehicle instead of updating the existing one. This is functional data corruption — a user who clicks "edit" then "Add Vehicle" creates a duplicate.

**Fix:** Implement `openEditVehicleDialog(vehicle: Vehicle)` that opens a dialog pre-populated with the vehicle's existing data and calls `vehicleService.updateVehicle()` on submit.

---

## Info

### IN-01: `signIn()` method has dead code that should be removed

**File:** `packages/dashboard/src/app/core/services/auth.service.ts:88-91`

Dead code left over from the incorrect signIn implementation (also flagged as CR-06). Lines 89–91 inside `signIn()` will never execute. Even if CR-06 is fixed, `signIn()` should simply call `loginWithRedirect()` and return — token exchange belongs in the callback handler.

---

### IN-02: `fleet_members` SELECT policy grants FOR ALL (including INSERT/UPDATE/DELETE) to any member

**File:** `packages/supabase/migrations/009_phase2_auth_schema.sql:131-132`

```sql
CREATE POLICY "Users can view own membership" ON fleet_members
    FOR ALL USING (user_id = (auth.jwt()->>'sub'));
```

`FOR ALL` with only a `USING` clause (no `WITH CHECK`) covers SELECT, UPDATE, and DELETE for the authenticated user's own row. A member can delete their own `fleet_members` row (leaving the fleet) and update their own role — including escalating it to `owner`. Changing `FOR ALL` to `FOR SELECT` is the minimal safe policy for a read-only membership check.

**Fix:**

```sql
CREATE POLICY "Users can view own membership" ON fleet_members
    FOR SELECT USING (user_id = (auth.jwt()->>'sub'));
```

Add separate, restricted policies for any needed self-service actions (e.g., leave fleet).

---

### IN-03: `getVehicleDisplayName` returns `"null null"` when make/model are null

**File:** `packages/dashboard/src/app/core/models/vehicle.model.ts:36-38`

```typescript
export function getVehicleDisplayName(vehicle: Vehicle): string {
  return `${vehicle.year ? vehicle.year + ' ' : ''}${vehicle.make} ${vehicle.model}`.trim();
}
```

Since Phase 2 makes `make` and `model` nullable (migration line 100–102), this returns `"null null"` when both are absent. This string is passed to `alertService.checkAndCreateAlerts` as the vehicle name label.

**Fix:**

```typescript
export function getVehicleDisplayName(vehicle: Vehicle): string {
  if (vehicle.nickname) return vehicle.nickname;
  const parts = [vehicle.year, vehicle.make, vehicle.model].filter(Boolean);
  return parts.length > 0 ? parts.join(' ') : 'Unknown Vehicle';
}
```

---

_Reviewed: 2026-05-13T10:18:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
