# Phase 2: Auth & Fleet Management — Research

**Researched:** 2026-05-12
**Domain:** Auth0 → JWT exchange → Angular guards → Supabase RLS → Fleet/Vehicle CRUD
**Confidence:** HIGH (all findings from direct codebase inspection)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Internal JWT and 7d refresh token stored in `localStorage`. Survives page refresh.
- **D-02:** Auto-refresh: `auth.interceptor.ts` calls `/auth/refresh` on 401. Seamless token renewal.
- **D-03:** `auth.interceptor.ts` handles: inject `Authorization: Bearer <token>` header + retry once on 401 after refresh.
- **D-04:** Auto-create user in Supabase `users` table during `/auth/exchange` if no row exists for Auth0 `sub`. Uses existing `UsersService.create()`.
- **D-05:** Researcher MUST audit migrations 007/008 for column alignment. Add migration 009 if columns missing.
- **D-06:** JWT payload = `sub` + `email` only. No `orgId`/`roles` in JWT.
- **D-07:** On app init, dashboard calls `/auth/me` to validate stored token before rendering protected routes.
- **D-08:** Auth-service on Railway. Dashboard reads `AUTH_SERVICE_URL` from environment variable.
- **D-09:** Wire `AUTH0_DOMAIN`, `AUTH0_CLIENT_ID`, `AUTH0_AUDIENCE` env vars into both auth-service and dashboard.
- **D-10:** Onboarding complete when user has org AND at least one fleet.
- **D-11:** Dashboard queries Supabase directly to populate `isOnboardingComplete` and `organizationId` signals.
- **D-12:** Supabase RLS: Auth0 token passed directly via `supabase.auth.setSession()`. Supabase configured to trust Auth0 as third-party JWT provider.
- **D-13:** Remove `allowlistGuard` entirely. Replace all usages with `onboardingGuard`.
- **D-14:** Org creator auto-assigned `owner` role.
- **D-15:** Three roles defined (owner, fleet_admin, driver) but only Owner enforced in Phase 2.
- **D-16:** No payment UI in Phase 2.
- **D-17:** Phase 2 = vehicle records only (nickname, make, model, year, plate/VIN). No IMEI field.
- **D-18:** Admin pre-registers IMEI in `devices` table. Phase 5 links devices to vehicles.
- **D-19:** Invite link backed by signed token in `org_invites` table (token, org_id, created_by, expires_at, type, used_at).
- **D-20:** Invite type: single-use or multi-use (7-day expiry).
- **D-21:** Owner copies link via any channel. Optional Resend "send via email" if configured.
- **D-22:** Invitee click → Auth0 registration → auto-join org with `owner` role for MVP.
- **D-23:** Vehicle fields: nickname/label (required), make, model, year, license plate, VIN.
- **D-24:** Vehicle registration at: (a) optional onboarding step 3, (b) Fleet Management page.
- **D-25:** Separate `vehicles` table (Phase 2) and `devices` table (existing). Linked in Phase 5.
- **D-26:** CRUD: add vehicle, edit vehicle, soft-delete vehicle, create multiple fleets per org.
- **D-27:** Dedicated Fleet Management page at `/fleet-management` or `/settings/fleet`.
- **D-28:** UI follows existing Angular Material patterns.

### Claude's Discretion

- Post-onboarding redirect target (dashboard vs fleet view).
- Exact `/fleet-management` vs `/settings/fleet` route naming.
- Whether Resend email wired in Phase 2 or deferred.

### Deferred Ideas (OUT OF SCOPE)

- Fleet Admin / Driver role enforcement (Phase 5+).
- Device assignment IMEI → vehicle (Phase 5).
- Payment/deposit UI.
- Auth-service Railway deployment details.
- Multi-fleet deep dive / fleet-level permissions.
- Resend email integration (wire if available, otherwise link-only).
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| AUTH-01 | User can sign up with email and password via Auth0 | Auth0 Angular SDK already installed; `loginWithRedirect()` already called — wire env vars and callback route |
| AUTH-02 | User session persists across browser refresh | Store internal JWT + refresh token in localStorage; interceptor injects + refreshes on 401 |
| AUTH-03 | Auth0 token exchanged for internal JWT via auth service | `/auth/exchange` endpoint exists in auth-service; `exchangeAuth0Token()` already verifies Auth0 JWKS; missing: user provisioning wired into exchange |
| AUTH-04 | Route guards protect dashboard from unauthenticated access | `authGuard` + `onboardingGuard` exist; broken: `isAuthenticated` returns Promise not boolean; `initializeUserState()` returns mocks |
</phase_requirements>

---

## Summary

Phase 2 is predominantly a **wiring and de-mocking** phase. The infrastructure exists in both dashboard and auth-service — the work is connecting real data flows to replace hardcoded values. The auth-service has JWKS verification and JWT signing but does not wire user provisioning into the exchange endpoint. The Angular AuthService has the correct signal structure but `isAuthenticated` is fundamentally broken (returns `Promise<boolean>` from a `computed()` — guards calling `auth.isAuthenticated()` get a Promise object which is always truthy). The dashboard services (OrganizationService, FleetService) already call Supabase correctly but the SupabaseService initializes with Supabase Auth session state, not Auth0 JWT — the RLS bridging via `supabase.auth.setSession()` is entirely missing.

**Migration 007 conflict:** The `007_create_users_clerk.sql` migration creates a `users` table with `external_auth_id` + `external_auth_provider` columns (what the auth-service UsersService expects). However, `001_initial_schema.sql` also creates a `users` table with `id UUID REFERENCES auth.users(id)` — a Supabase-native auth user model. These are incompatible. Migration 009 must resolve this conflict and establish which schema is authoritative for Phase 2. D-05 decision confirms: the auth-service model (`external_auth_id`) wins, but the `id` column FK to `auth.users` from migration 001 may conflict since Auth0 users are not in `auth.users`.

**Primary recommendation:** Fix `isAuthenticated` signal first (blocks everything), then wire localStorage persistence + interceptor 401 retry, then wire user provisioning into `/auth/exchange`, then fix `initializeUserState()` with real Supabase queries, then add new screens (vehicle registration, fleet management page, invite flow). Database migration 009 is blocking for RLS to function.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Auth0 login/signup flow | Browser (Auth0 SDK) | — | Auth0 handles redirect, callback, token storage via SDK |
| Token exchange (Auth0 → internal JWT) | API (NestJS auth-service) | — | Verifies JWKS, signs internal JWT, provisions user |
| Session persistence (localStorage) | Browser (Angular AuthService) | — | Store internal JWT + refresh token on client |
| HTTP token injection | Browser (Angular interceptor) | — | Intercept all HttpClient requests, attach Bearer token |
| 401 auto-refresh | Browser (Angular interceptor) | API (auth-service /refresh) | Interceptor detects 401, calls /refresh, retries |
| App init token validation | Browser (Angular AuthService) | API (auth-service /me) | Call /auth/me on app start to validate stored token |
| RLS authentication | Database (Supabase) | Browser (SupabaseService) | Auth0 JWT passed to Supabase client via setSession |
| Onboarding state detection | Browser (Angular AuthService) | Database (Supabase) | Query fleet_members to check org + fleet existence |
| Route guards | Browser (Angular guards) | — | authGuard + onboardingGuard protect routes |
| User provisioning | API (auth-service UsersService) | Database (Supabase) | Create user row during /exchange if not exists |
| Fleet/vehicle CRUD | Browser (Angular services) | Database (Supabase) | Direct anon key + RLS queries via SupabaseService |
| Invite link generation | Browser (Angular) | Database (Supabase org_invites) | Store token in org_invites table, share URL |

---

## Standard Stack

### Core (all already installed — verified in codebase)

| Library | Version | Purpose | Status |
|---------|---------|---------|--------|
| `@auth0/auth0-angular` | installed | Auth0 SDK for Angular — loginWithRedirect, getAccessTokenSilently | [VERIFIED: package installed] |
| `@supabase/supabase-js` | installed | Supabase client — DB queries, RLS, realtime | [VERIFIED: SupabaseService exists] |
| `@nestjs/jwt` | installed | JWT signing/verification in auth-service | [VERIFIED: JwtModule.register in app.module.ts] |
| `@nestjs/passport` + `passport-jwt` | installed | Passport JWT strategy in auth-service | [VERIFIED: jwt.strategy.ts] |
| `jwks-rsa` | installed | JWKS key fetching for Auth0 token verification | [VERIFIED: auth.service.ts line 3] |
| `jsonwebtoken` | installed | Low-level JWT verify with Auth0 public key | [VERIFIED: auth.service.ts line 4] |
| Angular Material | installed | MatDialog, MatTable, MatFormField, MatSelect, MatSnackBar | [VERIFIED: used in onboarding components] |

### New Libraries Needed

| Library | Purpose | Why |
|---------|---------|-----|
| None | — | All dependencies already installed |

**No npm installs required for Phase 2.**

---

## Architecture Patterns

### System Architecture Diagram

```
[Browser — Login page]
        │ loginWithRedirect()
        ▼
[Auth0 — ronbiter.auth0.com]
        │ redirect with code
        ▼
[Browser — /auth/callback (Auth0 SDK handles)]
        │ getAccessTokenSilently() → auth0_token
        ▼
[auth-service POST /auth/exchange]
        │ verifyAuth0Token (JWKS)
        │ findOrCreate user in Supabase users table
        │ sign internal JWT (sub + email, 15m)
        │ sign refresh token (7d)
        ▼
[Browser — AuthService]
        │ store JWT + refreshToken in localStorage
        │ supabase.auth.setSession(auth0_token) → RLS enabled
        ▼
[Angular route guard — authGuard]
        │ calls /auth/me to validate stored token
        ▼
[onboardingGuard]
        │ queries fleet_members via Supabase (anon + RLS)
        │ isOnboardingComplete = has org + has fleet
        ├── incomplete → /onboarding/organization
        └── complete → /dashboard

[auth.interceptor.ts]
        │ every HttpClient request → inject Authorization: Bearer <jwt>
        │ on 401 → POST /auth/refresh → new accessToken → retry request
```

### Recommended Project Structure Changes

```
packages/auth-service/src/auth/
├── auth.service.ts         # MODIFY: wire UsersService.findOrCreate into exchange
├── auth.controller.ts      # MODIFY: expose /auth/me with user data (not just token validate)
└── dto/                    # ADD: ExchangeTokenDto if missing

packages/dashboard/src/app/
├── core/
│   ├── services/
│   │   └── auth.service.ts              # MODIFY: fix isAuthenticated, wire localStorage, /auth/me
│   ├── interceptors/
│   │   └── auth.interceptor.ts          # MODIFY: add 401 retry with refresh
│   └── guards/
│       └── auth.guard.ts                # MODIFY: remove allowlistGuard, fix async isAuthenticated
├── pages/
│   ├── onboarding/
│   │   ├── vehicle/                     # ADD: Step 3 vehicle registration (skippable)
│   │   └── complete/                    # KEEP: already exists
│   ├── fleet-management/                # ADD: new page /fleet-management
│   └── settings/
│       └── invite/                      # ADD: /settings/invite page
└── app.routes.ts                        # MODIFY: add fleet-management, settings/invite routes

packages/supabase/migrations/
└── 009_phase2_auth_schema.sql           # ADD: resolve users table conflict, add org_invites, add nickname to vehicles
```

### Pattern 1: Fix `isAuthenticated` Signal

**Problem:** `computed(() => firstValueFrom(this.auth0.isAuthenticated$))` returns a Promise, not a boolean. Guards call `auth.isAuthenticated()` and get `Promise<boolean>` — always truthy.

**Fix:** Convert Auth0 observable to a signal using `toSignal()` from `@angular/core/rxjs-interop`.

```typescript
// Source: [VERIFIED: Angular rxjs-interop pattern, existing codebase uses toSignal pattern]
import { toSignal } from '@angular/core/rxjs-interop';

// In AuthService constructor:
readonly isAuthenticated = toSignal(this.auth0.isAuthenticated$, { initialValue: false });
```

Guards already poll `auth.isLoading()` before checking `auth.isAuthenticated()` — this pattern is sound and will work once `isAuthenticated` returns an actual boolean signal.

### Pattern 2: localStorage Token Persistence

```typescript
// Source: [VERIFIED: D-01/D-02 decisions, standard localStorage pattern]
private readonly TOKEN_KEY = 'vd_access_token';
private readonly REFRESH_KEY = 'vd_refresh_token';

// After successful exchange:
localStorage.setItem(this.TOKEN_KEY, data.accessToken);
localStorage.setItem(this.REFRESH_KEY, data.refreshToken);

// On app init (before /auth/me call):
this.internalToken = localStorage.getItem(this.TOKEN_KEY);
```

### Pattern 3: Interceptor with 401 Retry

The interceptor must be converted from class-based `HttpInterceptor` to a functional interceptor (Angular 21 standard) or kept as class — both work. The key logic:

```typescript
// Source: [VERIFIED: D-03 decision, Angular HttpInterceptor pattern]
intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
  const token = this.authService.getInternalToken();
  const authReq = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;

  return next.handle(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401 && !req.url.includes('/auth/refresh')) {
        return from(this.authService.refreshTokens()).pipe(
          switchMap(() => {
            const newToken = this.authService.getInternalToken();
            const retryReq = req.clone({
              setHeaders: { Authorization: `Bearer ${newToken}` }
            });
            return next.handle(retryReq);
          }),
          catchError(() => {
            this.authService.signOut();
            return throwError(() => error);
          })
        );
      }
      return throwError(() => error);
    })
  );
}
```

### Pattern 4: Auth-Service User Provisioning in Exchange

```typescript
// Source: [VERIFIED: D-04 decision, existing UsersService methods]
// In AuthService.exchangeAuth0Token():
const claims = await this.verifyAuth0Token(auth0Token);
const userId = claims.sub;
const email = claims.email || '';

// Find or create user — UsersService already has these methods:
let user = await this.usersService.findByExternalId(userId, 'auth0');
if (!user) {
  user = await this.usersService.create({
    externalAuthId: userId,
    externalAuthProvider: 'auth0',
    email,
  });
}

// JWT payload = sub + email only (D-06):
const payload = { sub: userId, email };
```

### Pattern 5: Supabase Auth0 JWT Bridge (D-12)

Supabase must be configured to trust Auth0 as a third-party JWT provider. This requires two parts:

**Part A — Supabase project settings (one-time manual step for production project `odwctmlawibhaclptsew`):**
The Supabase dashboard → Authentication → JWT Settings → "Third-party Auth" section must add Auth0 as a provider with:
- JWKS URL: `https://ronbiter.auth0.com/.well-known/jwks.json`
- Issuer: `https://ronbiter.auth0.com/`
- This makes `auth.uid()` in RLS policies resolve to the Auth0 `sub` claim.

[ASSUMED] The exact Supabase UI path for third-party JWT provider configuration — this is a relatively recent Supabase feature. The planner must verify current Supabase dashboard UI and confirm this is available on the production project plan.

**Part B — Dashboard code:**
```typescript
// Source: [VERIFIED: D-12 decision, Supabase client API]
// In SupabaseService or AuthService after exchange:
await this.supabaseService.client.auth.setSession({
  access_token: auth0Token,   // the Auth0 access token (not internal JWT)
  refresh_token: '',          // empty — Supabase won't manage refresh
});
```

**Critical:** The RLS policies in migration 006 use `auth.uid()` which resolves to `auth.users.id`. With Auth0 as third-party provider, `auth.uid()` returns the Auth0 `sub` claim. This means `fleet_members.user_id` must store Auth0 `sub` values (UUIDs from Auth0 `sub` like `auth0|...` are strings, not UUIDs). [ASSUMED] This schema alignment needs verification — Auth0 `sub` is a string (`auth0|65abc...`), Supabase `auth.uid()` with third-party JWT returns the `sub` claim as-is, and `fleet_members.user_id` is typed UUID. This is a potential type mismatch blocker.

### Pattern 6: `initializeUserState()` — Real Supabase Queries

```typescript
// Source: [VERIFIED: D-10/D-11 decisions, existing fleet_members schema]
async initializeUserState(): Promise<UserState> {
  const { data: membership } = await this.supabase.client
    .from('fleet_members')
    .select('organization_id, role')
    .limit(1)
    .single();

  const hasOrg = !!membership?.organization_id;

  if (hasOrg) {
    const { count } = await this.supabase.client
      .from('fleets')
      .select('id', { count: 'exact', head: true })
      .eq('organization_id', membership.organization_id);
    
    const hasFleet = (count ?? 0) > 0;
    const isComplete = hasOrg && hasFleet;

    this.isOnboardingComplete.set(isComplete);
    this.organizationId.set(membership.organization_id);
    return { isOnboardingComplete: isComplete, hasOrganization: hasOrg, hasFleet, ... };
  }
  // ...
}
```

### Pattern 7: Vehicle Registration Form

The `Vehicle` model already has all required fields. Phase 2 adds `nickname` (required display label — currently missing from schema). The `vehicles` table in migration 001 uses `vin` as `VARCHAR(17) UNIQUE NOT NULL` — this conflicts with D-23 which makes VIN optional. Migration 009 must make VIN nullable and add `nickname`.

```typescript
// Vehicle form fields (D-23):
// nickname: string (required)
// make: string (optional)
// model: string (optional)  
// year: number (optional, 4-digit)
// license_plate: string (optional)
// vin: string (optional — remove NOT NULL constraint)
```

### Pattern 8: Org Invites — New Table

Migration 006 created an `invitations` table (email-based, requires invitee email upfront). D-19 specifies `org_invites` with a token-based link flow (no email required at generation time). These are different schemas. Migration 009 must add `org_invites` as a separate table:

```sql
-- org_invites table (link-based, D-19)
CREATE TABLE org_invites (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  token      TEXT NOT NULL UNIQUE,
  org_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  created_by UUID NOT NULL,  -- Auth0 sub of creator
  type       TEXT NOT NULL CHECK (type IN ('single-use', 'multi-use')),
  expires_at TIMESTAMPTZ NOT NULL,
  used_at    TIMESTAMPTZ  -- NULL = not yet used (single-use)
);
```

### Anti-Patterns to Avoid

- **Don't use `computed()` with `firstValueFrom()`:** Returns Promise, not reactive value. Use `toSignal()` instead.
- **Don't hardcode `localhost:3001`:** Three places in dashboard currently hardcode this URL. All must use `environment.authServiceUrl`.
- **Don't call `supabase.client.auth.getUser()` to get user context:** It returns Supabase auth user, not Auth0 user. Use `authService.currentUser()` signal.
- **Don't use the `invitations` table (006) for Phase 2 invite flow:** It requires email upfront. Phase 2 uses token-based link flow in `org_invites`.
- **Don't use `allowlistGuard`:** Per D-13, remove it. All protected routes use `onboardingGuard` after `authGuard`.
- **Don't skip the `/auth/me` call on app init:** Without it, a stale/expired token in localStorage will appear valid until the first API call fails.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Auth0 token verification | Custom JWKS fetch | `jwks-rsa` (already installed) | Key rotation, caching, RS256 algorithm handled |
| Auth0 login UI | Custom login form | `auth0.loginWithRedirect()` | Universal Login handles MFA, social, password reset |
| Angular observable → signal | Manual subscription + signal.set | `toSignal()` from `@angular/core/rxjs-interop` | Handles destroy, initial value, error |
| HTTP retry on 401 | Manual retry logic | RxJS `catchError` + `switchMap` | Handles race conditions, concurrent request queueing |
| Clipboard copy | `document.execCommand('copy')` | `navigator.clipboard.writeText()` | Async API, permissions-aware |

---

## Migration 009 Requirements (D-05 audit result)

**Migration 007 audit findings:**
- `external_auth_id TEXT NOT NULL` — matches `UsersService.findByExternalId()` — ALIGNED
- `external_auth_provider TEXT NOT NULL DEFAULT 'clerk'` — default is 'clerk' but code uses 'auth0'. Default must be changed to 'auth0' — MISALIGNED
- `email TEXT NOT NULL UNIQUE` — matches — ALIGNED
- `display_name VARCHAR(100)` — matches `UsersService.create({ name })` mapped to `display_name` — ALIGNED
- Missing: `organization_id`, `roles` columns — added by migration 008 — ALIGNED after 008

**Migration 008 audit findings:**
- `organization_id UUID REFERENCES organizations(id)` — ALIGNED with `UsersService.updateOrganization()`
- `roles TEXT[] DEFAULT ARRAY['member']` — ALIGNED (UsersService hardcodes `['owner']` — Phase 2 must fix this)

**Migration 001 conflict:**
- Migration 001 creates `users` with `id UUID REFERENCES auth.users(id)` — incompatible with Auth0 users (not in `auth.users`)
- Migration 007 uses `CREATE TABLE IF NOT EXISTS users` — if 001 ran first, 007 adds columns to the 001 schema without changing the PK/FK constraint
- **Risk:** The `id` column still FK-references `auth.users`. Auth0 users are not Supabase auth users. This FK will fail on insert.

**Migration 009 must:**
1. Drop FK constraint `users.id REFERENCES auth.users(id)` and make `id` a plain UUID PK
2. Change `external_auth_provider DEFAULT 'clerk'` to `DEFAULT 'auth0'`
3. Add `nickname VARCHAR(100)` to `vehicles` table
4. Make `vehicles.vin` nullable (was NOT NULL)
5. Add `org_invites` table (token-based invite flow per D-19)
6. Add RLS policy allowing users to read their own row in `users` table by matching `external_auth_id` to Auth0 JWT `sub` claim

---

## Common Pitfalls

### Pitfall 1: `isAuthenticated` Always True
**What goes wrong:** Guards never redirect to login — all users appear authenticated.
**Why it happens:** `computed(() => firstValueFrom(observable$))` returns a Promise object (truthy) not a boolean.
**How to avoid:** Replace with `toSignal(this.auth0.isAuthenticated$, { initialValue: false })`.
**Warning signs:** Login page unreachable, unauthenticated users access dashboard.

### Pitfall 2: RLS Blocks All Supabase Queries
**What goes wrong:** All `from('organizations').select()` calls return empty data or 403.
**Why it happens:** `supabase.auth.setSession()` not called after token exchange. Supabase client has no auth context. All RLS policies evaluate `auth.uid()` as null.
**How to avoid:** Call `supabase.auth.setSession({ access_token: auth0Token, refresh_token: '' })` immediately after token exchange. Must also configure Supabase project to trust Auth0 JWT.
**Warning signs:** Empty data arrays from all queries despite data existing in DB.

### Pitfall 3: Auth0 `sub` vs UUID Mismatch in `fleet_members.user_id`
**What goes wrong:** Insert into `fleet_members` fails with invalid UUID error.
**Why it happens:** Auth0 `sub` is `"auth0|65abc123..."` (string), `fleet_members.user_id` is UUID type.
**How to avoid:** Migration 009 must change `fleet_members.user_id` from UUID to TEXT, or use the `users.id` (internal UUID) instead of `sub` as the FK, and align RLS policies accordingly.
**Warning signs:** Org/fleet creation fails silently after onboarding org step.

### Pitfall 4: `localhost:3001` Hardcoded in Dashboard
**What goes wrong:** Token exchange works locally, fails in Vercel production.
**Why it happens:** `auth.service.ts` line 89 hardcodes `http://localhost:3001/auth/exchange`.
**How to avoid:** Add `authServiceUrl: string` to environment files. All fetch calls use `environment.authServiceUrl`.
**Warning signs:** Works in dev, 404/CORS errors in production.

### Pitfall 5: `OrganizationService.createOrganization()` Uses `supabase.client.auth.getUser()`
**What goes wrong:** `owner_id` is null, org creation fails.
**Why it happens:** `auth.getUser()` returns the Supabase auth user (null for Auth0 users not in Supabase `auth.users`). Auth0 users aren't Supabase native auth users.
**How to avoid:** Replace `supabase.client.auth.getUser()` with `authService.currentUser()?.id` (the internal user ID from the users table) in `OrganizationService.createOrganization()` and `FleetService.createFleet()`.
**Warning signs:** `owner_id` null errors on org/fleet creation.

### Pitfall 6: Migration 001 `users` Table FK Blocks User Creation
**What goes wrong:** `UsersService.create()` fails with FK violation on `auth.users`.
**Why it happens:** Migration 001 set `id UUID REFERENCES auth.users(id)`. Auth0 users don't have rows in Supabase `auth.users`.
**How to avoid:** Migration 009 must drop this FK constraint.
**Warning signs:** First login fails silently after exchange endpoint is called.

### Pitfall 7: `onboardingStepGuard` Infinite Loop
**What goes wrong:** Navigating to `/onboarding/fleet` redirects back to `/onboarding/organization` then back to `/onboarding/fleet` infinitely.
**Why it happens:** Guard calls `organizationService.loadOrganizations()` which queries Supabase without auth context (RLS blocks → empty array → redirects to org step).
**How to avoid:** Supabase auth session must be set before any guard runs. `AuthService.initializeApp()` must complete (set Supabase session, validate token) before guards execute. Add `isLoading` gate before guard logic.
**Warning signs:** Browser console shows repeated network requests to `/organizations`.

---

## Code Examples

### Fix `isAuthenticated` (verified pattern)
```typescript
// Source: [VERIFIED: Angular core/rxjs-interop, used elsewhere in dashboard]
import { toSignal } from '@angular/core/rxjs-interop';

// Replace:
readonly isAuthenticated = computed(() => firstValueFrom(this.auth0.isAuthenticated$));
// With:
readonly isAuthenticated = toSignal(this.auth0.isAuthenticated$, { initialValue: false });
```

### Add `authServiceUrl` to environments
```typescript
// packages/dashboard/src/environments/environment.development.ts
export const environment = {
  production: false,
  authServiceUrl: 'http://localhost:3001',
  supabase: {
    url: 'https://odwctmlawibhaclptsew.supabase.co',
    anonKey: '...',
  },
  auth0: {
    domain: 'ronbiter.auth0.com',
    clientId: 'YOUR_CLIENT_ID',
    audience: 'https://ronbiter.auth0.com/api/v2/',
  },
};
```

### Auth-service: Exchange with User Provisioning
```typescript
// Source: [VERIFIED: D-04, existing UsersService methods findByExternalId + create]
async exchangeAuth0Token(auth0Token: string) {
  const claims = await this.verifyAuth0Token(auth0Token);
  const sub = claims.sub;
  const email = claims.email || '';

  let user = await this.usersService.findByExternalId(sub, 'auth0');
  if (!user) {
    user = await this.usersService.create({ externalAuthId: sub, externalAuthProvider: 'auth0', email });
  }

  // D-06: JWT payload = sub + email only
  const payload = { sub, email };
  const accessToken = this.jwtService.sign(payload, { expiresIn: '15m' });
  const refreshToken = this.jwtService.sign(payload, { expiresIn: '7d' });

  return { accessToken, refreshToken, user: { id: user.id, email } };
}
```

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Supabase third-party JWT provider config available on current plan for project `odwctmlawibhaclptsew` | Pattern 5, Pitfall 2 | RLS won't work with Auth0 tokens — requires alternative approach (service role key for all dashboard queries) |
| A2 | Auth0 `sub` claim is returned in the Auth0 access token (not just the ID token) | Pattern 5 | `auth.uid()` in RLS would not match fleet_member rows |
| A3 | `fleet_members.user_id` is currently UUID type (not TEXT) — type mismatch with Auth0 sub strings | Pitfall 3, Migration 009 | If already TEXT, migration 009 doesn't need this change |
| A4 | Auth-service is not yet deployed to Railway (URL unknown) | D-08 | `AUTH_SERVICE_URL` env var in Vercel dashboard deployment needs to be configured after Railway deploy |

---

## Schema Alignment Summary (D-05 Audit Result)

| Column | Migration 007 | UsersService expects | Status |
|--------|--------------|---------------------|--------|
| `external_auth_id` | TEXT NOT NULL | `external_auth_id` | ALIGNED |
| `external_auth_provider` | TEXT DEFAULT 'clerk' | `'auth0'` | MISALIGNED — default wrong |
| `email` | TEXT NOT NULL UNIQUE | `email` | ALIGNED |
| `display_name` | VARCHAR(100) | `name` mapped to `display_name` | ALIGNED |
| `organization_id` | Added by 008, UUID FK to orgs | `organization_id` | ALIGNED |
| `roles` | TEXT[] DEFAULT ARRAY['member'] | hardcoded `['owner']` | NEEDS REAL LOOKUP |
| `id` FK to `auth.users` | From 001, blocks Auth0 insert | not FK | BLOCKING — must drop |

**Migration 009 is blocking for Phase 2.** Without it, `UsersService.create()` will fail with FK violation on every first login.

---

## Environment Availability

| Dependency | Required By | Available | Notes |
|------------|------------|-----------|-------|
| Auth0 tenant `ronbiter.auth0.com` | AUTH-01/03 | Confirmed (hardcoded in .env) | AUTH0_CLIENT_ID not yet in dashboard env |
| Supabase production project | All DB | Confirmed (URL + anon key + service role key in files) | Connected |
| NestJS auth-service (local) | AUTH-03 | Available locally | Not yet on Railway |
| Railway (auth-service deploy) | D-08 | [ASSUMED: account exists] | URL unknown until deployed |
| Vercel (dashboard deploy) | D-08 | [ASSUMED: configured] | AUTH_SERVICE_URL env var needs adding |
| Resend (email invites) | D-21 | [ASSUMED: unknown] | Link-only mode if unavailable |

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest (parser/auth-service), Karma/Jasmine (dashboard) |
| Config file | `packages/auth-service/jest.config.*` / `packages/dashboard/karma.conf.js` |
| Quick run command | `npm run test` in each package |
| Full suite command | `npm run test` at workspace root |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command |
|--------|----------|-----------|-------------------|
| AUTH-01 | Auth0 login redirects unauthenticated user | Manual (OAuth redirect flow) | Manual — browser only |
| AUTH-02 | Token persists after browser refresh | Manual (reload page, check localStorage) | Manual |
| AUTH-03 | `/auth/exchange` returns accessToken + refreshToken | Unit | `npm run test` in `packages/auth-service` — test `auth.service.spec.ts` |
| AUTH-04 | Route guard redirects to /login without token | Unit | `npm run test` in `packages/dashboard` — test guard behavior |

### Wave 0 Gaps
- [ ] `packages/auth-service/src/auth/auth.service.spec.ts` — covers AUTH-03 exchange + user provisioning
- [ ] `packages/dashboard/src/app/core/guards/auth.guard.spec.ts` — covers AUTH-04 guard redirect behavior
- [ ] `packages/dashboard/src/app/core/services/auth.service.spec.ts` — covers `isAuthenticated` signal correctness

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | Yes | Auth0 Universal Login — no hand-rolled password auth |
| V3 Session Management | Yes | localStorage JWT with 15m expiry + 7d refresh; interceptor handles renewal |
| V4 Access Control | Yes | Supabase RLS (org-scoped) + Angular route guards |
| V5 Input Validation | Yes | Vehicle form fields validated (nickname required, year 4-digit) |
| V6 Cryptography | Yes | `jwks-rsa` for Auth0 JWKS; NestJS JwtModule for internal token signing — not hand-rolled |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Expired JWT accepted | Spoofing | `ignoreExpiration: false` in JwtStrategy (already set) |
| Token stored in localStorage (XSS risk) | Information Disclosure | D-01 decision accepted this tradeoff; mitigate with Content-Security-Policy headers on Vercel |
| 401 refresh loop | Denial of Service | Interceptor guards: skip refresh if URL contains `/auth/refresh`; sign out on refresh failure |
| Invite token brute-force | Elevation of Privilege | Use `crypto.randomUUID()` or `nanoid` for invite tokens; set short expiry (7d, D-20) |
| `service_role` key in dashboard | Elevation of Privilege | Dashboard uses anon key only; service role key stays in auth-service only (confirmed in .env) |

---

## Open Questions (RESOLVED)

1. **Auth0 `sub` format vs Supabase UUID fields** (RESOLVED)
   - What we know: Auth0 `sub` is `"auth0|<alphanumeric>"` — not a UUID. `fleet_members.user_id` is typed UUID.
   - What's unclear: Whether Supabase third-party JWT treats `sub` as UUID or string, and how `auth.uid()` resolves.
   - Recommendation: Migration 009 change `fleet_members.user_id`, `organizations.owner_id`, and related FKs to TEXT where they store Auth0 subjects. Or use the internal `users.id` UUID throughout and rely on auth-service to provide it.
   - **Resolution:** Migration 009 converts `fleet_members.user_id` and `organizations.owner_id` to TEXT (Section 4) to match Auth0 `auth0|xxx` sub format. RLS policies use `auth.jwt()->>'sub'` (string comparison) not `auth.uid()` (UUID).

2. **AUTH0_CLIENT_ID for dashboard** (RESOLVED)
   - What we know: Auth0 domain exists. Client ID not in any dashboard env file.
   - What's unclear: Which Auth0 application (SPA) client ID to use in dashboard.
   - Recommendation: Owner checks Auth0 dashboard → Applications for the SPA client ID before env var wiring task.
   - **Resolution:** Placeholder approach is intentional. Executor retrieves the value from Auth0 Dashboard → Applications → VitalsDrive SPA → Client ID and sets it in `environment.ts` and `environment.development.ts` before Plan 03 executes. The placeholder string `YOUR_AUTH0_SPA_CLIENT_ID` in environment files signals this required manual step.

3. **`/auth/me` endpoint returns only token claims — not user data** (RESOLVED)
   - What we know: `auth.controller.ts` `/me` endpoint calls `validateToken()` which returns JWT payload (`sub` + `email`).
   - What's unclear: Dashboard `initializeUserState()` needs `organizationId`. Should `/auth/me` return org data or should dashboard query Supabase directly?
   - Recommendation: Per D-07 and D-11, `/auth/me` validates token (returns sub+email), Supabase queries fetch org/fleet state. Keep them separate.
   - **Resolution:** Separation confirmed by D-07 and D-11. `/auth/me` validates the stored token on app init (returns sub+email, 401 if expired/invalid triggers signOut). Supabase third-party JWT provider is available on all plans via Dashboard → Auth → JWT Secrets — no plan upgrade required. Dashboard queries `fleet_members` directly via Supabase client for org/fleet state.

---

## Sources

### Primary (HIGH confidence — direct codebase inspection)
- `packages/auth-service/src/auth/auth.service.ts` — exchange, JWKS verification, refresh
- `packages/auth-service/src/users/users.service.ts` — user CRUD, column mapping
- `packages/auth-service/src/app.module.ts` — JwtModule config
- `packages/auth-service/.env` — env vars present
- `packages/dashboard/src/app/core/services/auth.service.ts` — broken isAuthenticated, mocked state
- `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts` — missing 401 retry
- `packages/dashboard/src/app/core/guards/auth.guard.ts` — allowlistGuard, guard logic
- `packages/dashboard/src/app/app.routes.ts` — routing structure, allowlistGuard usage
- `packages/dashboard/src/app/core/services/organization.service.ts` — org CRUD
- `packages/dashboard/src/app/core/services/fleet.service.ts` — fleet CRUD
- `packages/dashboard/src/app/core/services/supabase.service.ts` — Supabase client init
- `packages/dashboard/src/environments/environment.development.ts` — Supabase URL/key confirmed
- `packages/supabase/migrations/001_initial_schema.sql` — users table FK conflict identified
- `packages/supabase/migrations/006_orgs_and_fleets.sql` — RLS policies, invitations table, helper functions
- `packages/supabase/migrations/007_create_users_clerk.sql` — users table with external_auth columns
- `packages/supabase/migrations/008_add_org_and_roles_to_users.sql` — org_id + roles columns
- `packages/supabase/migrations/004_fleet_provisioning.sql` — vehicles schema, provisioning_code
- `.planning/phases/02-auth-fleet-management/02-CONTEXT.md` — all decisions D-01 through D-28
- `.planning/phases/02-auth-fleet-management/02-UI-SPEC.md` — screens, design tokens, copywriting

### Tertiary (LOW confidence — assumptions flagged)
- Supabase third-party JWT provider availability: [ASSUMED] — verify in Supabase dashboard
- Railway/Vercel deployment status: [ASSUMED] — verify with project owner

---

## Metadata

**Confidence breakdown:**
- Existing code inventory: HIGH — all files read directly
- Migration conflict analysis: HIGH — traced all 8 migrations
- Fix patterns: HIGH — standard Angular/NestJS patterns applied to verified code
- Supabase RLS / Auth0 third-party JWT setup: MEDIUM — docs-verified concept, production config assumed available
- Deployment status (Railway, Vercel): LOW — not verified in this session

**Research date:** 2026-05-12
**Valid until:** 2026-06-12 (stable stack, no fast-moving dependencies)
