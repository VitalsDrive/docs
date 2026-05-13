# Phase 2: Auth & Fleet Management - Context

**Gathered:** 2026-05-12
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 2 delivers the identity layer end-to-end: users can authenticate via Auth0, the internal JWT exchange service works in production, and the dashboard reflects real authentication state (not mocked). Includes: user provisioning, org/fleet creation, invite flow, vehicle registration, and fleet management CRUD. 

**Out of scope:** device assignment (IMEI → vehicle) deferred to Phase 5. Payment/deposit UI not in app. Fleet Admin / Driver roles defined but not enforced (Owner only for MVP).

</domain>

<decisions>
## Implementation Decisions

### Session Persistence
- **D-01:** Internal JWT and 7d refresh token stored in `localStorage`. Survives page refresh.
- **D-02:** Auto-refresh strategy: `auth.interceptor.ts` calls `/auth/refresh` on 401. Seamless token renewal without re-prompting Auth0.
- **D-03:** `auth.interceptor.ts` handles both concerns: inject `Authorization: Bearer <token>` header on every request, and retry once on 401 after refreshing.

### User Provisioning
- **D-04:** Auto-create user in Supabase `users` table during `/auth/exchange` if no row exists for the Auth0 `sub`. Uses existing `UsersService.create()`.
- **D-05:** Researcher MUST audit migrations `007_create_users_clerk.sql` and `008_add_org_and_roles_to_users.sql` to verify column alignment with auth-service expectations (`external_auth_id`, `external_auth_provider`, `organization_id`, `roles`). Add migration `009` if columns are missing or misnamed.
- **D-06:** JWT payload = `sub` + `email` only. `orgId` and `roles` are NOT embedded in the JWT. Dashboard guards query Supabase separately for org/role data.
- **D-07:** On app init, dashboard calls `/auth/me` to validate the stored token is still valid before rendering protected routes.
- **D-08:** Auth-service deployed to Railway (same platform as parser). Dashboard reads `AUTH_SERVICE_URL` from environment variable — not hardcoded `localhost:3001`.
- **D-09:** Auth0 tenant (`ronbiter.auth0.com`) already exists. Phase wires `AUTH0_DOMAIN`, `AUTH0_CLIENT_ID`, `AUTH0_AUDIENCE` env vars into both auth-service and dashboard. No new Auth0 tenant/app creation needed.

### Onboarding Completion Logic
- **D-10:** Onboarding is complete when the user has an organization AND at least one fleet. Both must exist before redirecting to the main dashboard.
- **D-11:** Dashboard queries Supabase directly (anon key + RLS) to populate `isOnboardingComplete` and `organizationId` signals. `initializeUserState()` must be unwired from its mocked return values.
- **D-12:** Supabase RLS authentication: Auth0 token passed directly to Supabase client (`supabase.auth.setSession()`). Supabase configured to trust Auth0 as a third-party JWT provider — **researcher must document the Auth0 JWT provider setup steps for production Supabase project `odwctmlawibhaclptsew`**.
- **D-13:** Remove `allowlistGuard` entirely. Replace all usages in `app.routes.ts` with `onboardingGuard`. One guard concept.
- **D-14:** Org creator is automatically assigned `owner` role in the org.

### Roles (MVP Scope)
- **D-15:** Three roles defined in the data model: `owner`, `fleet_admin`, `driver`. Phase 2 implements Owner only — no role-based restrictions enforced on UI. Data model supports future expansion.

### Device & Payment Model
- **D-16:** Device payment is upfront and out-of-band. No payment UI in the app for MVP. Devices are purchased/deposited separately (invoice or Stripe Payment Link managed outside VitalsDrive).
- **D-17:** Phase 2 = vehicle records only (make, model, year, plate/VIN, nickname). No IMEI/device field on vehicles. Device assignment (IMEI → vehicle) is Phase 5.
- **D-18:** Admin pre-registers IMEI in Supabase `devices` table before shipping to customer. Customer sees "unassigned devices" in Phase 5 and links them to registered vehicles.

### Invite Flow
- **D-19:** Owner generates an invite link backed by a signed token stored in an `org_invites` Supabase table (token, org_id, created_by, expires_at, type, used_at).
- **D-20:** Owner chooses invite type at generation: **single-use** (invalidated after first click) or **multi-use** (anyone with link can join within the window). Both expire after 7 days.
- **D-21:** Owner copies the link and sends it via any channel (WhatsApp, email, Slack). Optional "Send via email" button using Resend — wire the email delivery if Resend is configured, otherwise link-only mode.
- **D-22:** When invitee clicks the link: Auth0 registration → auto-join org with the **role pre-selected by the inviter at invite generation** (default: `viewer`). Role options: `viewer`, `driver`, `fleet_admin`. The `org_invites` table stores the target role; redemption reads it and sets `fleet_members.role` accordingly. Invitees never join as `owner` via link. Owner can promote roles manually via Settings later.

### Vehicle Registration
- **D-23:** Vehicle fields: nickname/label (required), make, model, year, license plate, VIN. No IMEI — device assignment is Phase 5.
- **D-24:** Registration accessible at two points: (a) optional step during onboarding after fleet creation (skippable), and (b) Fleet Management page post-login.
- **D-25:** Separate `vehicles` table (Phase 2) and `devices` table (existing). Linked in Phase 5 via foreign key (`vehicles.device_id` or join table).

### Fleet Management
- **D-26:** CRUD operations in Phase 2: add vehicle, edit vehicle details, remove/deactivate vehicle (soft delete), create multiple fleets within an org.
- **D-27:** Dedicated Fleet Management page at `/fleet-management` or `/settings/fleet` — not inline in dashboard. Dashboard shows live data; fleet management page handles configuration.
- **D-28:** UI follows existing Angular Material patterns — same form style, same component library. Researcher to identify existing form components to reuse.

### Claude's Discretion
- Post-onboarding redirect target (dashboard vs fleet view) — planner chooses based on UX flow.
- Exact `/fleet-management` vs `/settings/fleet` route naming — planner decides.
- Whether email delivery via Resend is wired in Phase 2 or deferred — depends on Resend account availability.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/ROADMAP.md` §Phase 2 — Goal, success criteria, plan stubs
- `.planning/REQUIREMENTS.md` — AUTH-01, AUTH-02, AUTH-03, AUTH-04 requirement definitions

### Auth Infrastructure — Existing Code
- `packages/auth-service/src/auth/auth.service.ts` — JWT exchange, Auth0 JWKS verification, refresh logic
- `packages/auth-service/src/auth/auth.controller.ts` — /auth/exchange, /auth/refresh, /auth/validate, /auth/me endpoints
- `packages/auth-service/src/users/users.service.ts` — Supabase user CRUD (findByExternalId, create, updateOrganization)
- `packages/auth-service/src/auth/jwt.strategy.ts` — Passport-JWT strategy

### Dashboard Auth — Existing Code
- `packages/dashboard/src/app/core/services/auth.service.ts` — Angular AuthService (mock state that must be replaced)
- `packages/dashboard/src/app/core/guards/auth.guard.ts` — authGuard, onboardingGuard, onboardingStepGuard (allowlistGuard to be removed)
- `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts` — needs token injection + 401 retry wiring
- `packages/dashboard/src/app/app.routes.ts` — routing tree (login, signup, onboarding, protected shell)

### Database Schema
- `packages/supabase/migrations/006_orgs_and_fleets.sql` — orgs and fleets schema
- `packages/supabase/migrations/007_create_users_clerk.sql` — users table (Clerk-era, verify Auth0 alignment)
- `packages/supabase/migrations/008_add_org_and_roles_to_users.sql` — roles columns
- `packages/supabase/migrations/004_fleet_provisioning.sql` — fleet provisioning schema
- `packages/supabase/migrations/005_device_provisioning.sql` — devices table (Phase 5 will link vehicles here)

### Environment Config
- `packages/dashboard/src/environments/` — supabaseUrl, supabaseAnonKey (production project: `odwctmlawibhaclptsew`)
- `packages/auth-service/src/auth/auth.service.ts` — AUTH0_DOMAIN, AUTH0_AUDIENCE env var references

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **Auth0Angular SDK**: Already installed and wired in dashboard (`@auth0/auth0-angular`). `loginWithRedirect()` and `getAccessTokenSilently()` already called in `auth.service.ts`.
- **NestJS auth-service**: Exchange/refresh/validate endpoints exist. JWKS client initialized. Only missing: real user provisioning wired into exchange, and env-var config.
- **Angular route guards**: `authGuard`, `onboardingGuard`, `onboardingStepGuard` exist. Shell structure with `canActivate` already applied to protected routes.
- **Onboarding pages**: `/onboarding/organization` and `/onboarding/fleet` components already exist. Only missing: real Supabase writes and completion detection.
- **Supabase client**: `@supabase/supabase-js` in dependencies, `supabaseUrl` + `supabaseAnonKey` in environment files.
- **`auth.interceptor.ts`**: Exists but not fully wired — needs token injection + 401 refresh logic.

### Established Patterns
- Angular Signals for reactive state (`isLoading`, `isAuthenticated`, `isOnboardingComplete`, etc.) — all already declared as `signal()` or `computed()` in `auth.service.ts`.
- `firstValueFrom()` for observable-to-promise conversion — used throughout dashboard.
- NestJS `@nestjs/passport` + `@nestjs/jwt` pattern — already in auth-service deps.
- Angular Material forms — existing throughout dashboard UI.

### Known Issues to Fix
- `isAuthenticated` in `auth.service.ts` is declared as `computed(() => firstValueFrom(...))` — returns a Promise, not a boolean. Guards depend on this signal. Must be fixed.
- `initializeUserState()` returns hardcoded `true` for all flags — must be replaced with real Supabase queries.
- `checkUserRoles()` hardcodes `isAdmin=true, isOwner=true` — must query real roles.
- `roles` in `users.service.ts` hardcoded as `['owner']` with TODO comment — must implement real role lookup.

### Integration Points
- **Dashboard → auth-service:** `/auth/exchange` (login), `/auth/refresh` (token renewal), `/auth/me` (on-load validation). URL via `AUTH_SERVICE_URL` env var.
- **Dashboard → Supabase:** orgs, fleets, vehicles, org_invites tables via anon key + Auth0 JWT for RLS.
- **auth-service → Supabase:** `users` table via service role key (`SUPABASE_SERVICE_ROLE_KEY`).
- **Auth0 → Supabase:** Auth0 configured as third-party JWT provider in Supabase project settings.

</code_context>

<specifics>
## Specific Ideas

- **Auth0 domain:** `ronbiter.auth0.com` — already hardcoded, move to `AUTH0_DOMAIN` env var.
- **Invite table name:** `org_invites` — token, org_id, created_by, expires_at, type (single/multi), used_at.
- **Email provider:** Resend — if available, wire the "send via email" option. Otherwise link-only mode is fine.
- **Vehicle nickname:** Required field (e.g., "Truck 1", "Ron's Van") — primary identifier in fleet list.
- **Supabase project ID (production):** `odwctmlawibhaclptsew`

</specifics>

<deferred>
## Deferred Ideas

- **Fleet Admin / Driver roles enforcement** — roles are modeled in Phase 2 but only Owner is enforced. Role-based UI gates deferred to Phase 5+.
- **Device assignment (IMEI → vehicle)** — Phase 5: Device Onboarding.
- **Payment/deposit UI** — out-of-band, handled via invoice or Stripe Payment Link outside VitalsDrive.
- **Auth-service Railway deployment details** (Railway config, CORS, env var management) — planner scopes this. Not blocking planning.
- **Multi-fleet deep dive** — multiple fleets per org is in scope (D-26) but the detailed management UX (who can create fleets, fleet-level permissions) is left to planner/Phase 3.
- **Resend email integration** — wire if account is available; otherwise link-only. Not blocking Phase 2.

</deferred>

---

*Phase: 02 - Auth & Fleet Management*
*Context gathered: 2026-05-12*
