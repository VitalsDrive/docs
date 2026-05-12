# Phase 2: Auth & Fleet Management - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-12
**Phase:** 02-auth-fleet-management
**Areas discussed:** Session persistence, User provisioning, Onboarding completion logic, Vehicle registration scope, Invite flow mechanics

---

## Session Persistence

| Option | Description | Selected |
|--------|-------------|----------|
| localStorage | Simple, survives refresh. Standard SPA choice. | ✓ |
| sessionStorage | Cleared when tab closes. Stricter session lifecycle. | |
| Re-exchange on every load | Never store internal JWT — exchange fresh on each Auth0 confirm. | |

**User's choice:** localStorage

| Option | Description | Selected |
|--------|-------------|----------|
| Auto-refresh via refresh token | 7d refresh token in localStorage, interceptor calls /auth/refresh on 401. | ✓ |
| Re-exchange Auth0 token on 401 | Call getAccessTokenSilently() + re-exchange on expiry. | |
| You decide | Leave to planner. | |

**User's choice:** Auto-refresh via refresh token

| Option | Description | Selected |
|--------|-------------|----------|
| Inject token + handle 401 refresh | Single interceptor for both concerns. | ✓ |
| Token injection only | Manual 401 handling per-service. | |
| You decide | Leave interceptor strategy to planner. | |

**User's choice:** auth.interceptor.ts handles both token injection and 401 retry

---

## User Provisioning

| Option | Description | Selected |
|--------|-------------|----------|
| Auto-create on first exchange | Create user in Supabase if no row exists during /auth/exchange. | ✓ |
| Manual pre-registration | Admin must create user before first login. | |
| Create + set pending state | Auto-create with status='pending', pending page shown. | |

**User's choice:** Auto-create on first exchange

| Option | Description | Selected |
|--------|-------------|----------|
| Check and fix — add migration 009 | Researcher verifies 007/008 columns, adds 009 if needed. | ✓ |
| Assume it's fine | No schema investigation. | |
| You decide | Let researcher decide. | |

**User's choice:** Researcher audits migrations 007/008, adds 009 if misaligned

| Option | Description | Selected |
|--------|-------------|----------|
| Real orgId and roles in JWT | Embed org/roles from Supabase row into JWT at exchange time. | |
| Keep orgId/roles out of JWT | JWT has sub + email only, guards query Supabase separately. | ✓ |
| You decide | Leave JWT payload design to researcher. | |

**User's choice:** JWT = sub + email only. Guards query Supabase for org/roles.

| Option | Description | Selected |
|--------|-------------|----------|
| Dashboard trusts JWT directly | Decode only, no re-validation call. | |
| Dashboard re-validates via /auth/me on load | Call /auth/me on init to confirm token validity. | ✓ |
| You decide | Let researcher decide. | |

**User's choice:** Dashboard calls /auth/me on app init

| Option | Description | Selected |
|--------|-------------|----------|
| Railway | Same platform as parser. NestJS server fits naturally. | ✓ |
| Vercel serverless | Would require significant NestJS adapter work. | |
| Not deploying this phase | Keep local. | |

**User's choice:** Railway. User asked for take on complexity/pricing — Claude recommended Railway. User agreed.

| Option | Description | Selected |
|--------|-------------|----------|
| Already configured — wire env vars | Auth0 tenant exists, just need env vars. | ✓ |
| Needs to be created in this phase | Auth0 setup is in scope. | |
| I'll set up manually (prerequisite) | Manual infra outside the phase. | |

**User's choice:** Auth0 already configured. Wire AUTH0_DOMAIN, AUTH0_CLIENT_ID, AUTH0_AUDIENCE.

---

## Onboarding Completion Logic

| Option | Description | Selected |
|--------|-------------|----------|
| Org exists (MVP) | Org created = done. Fleet optional. | |
| Org + at least one fleet | Both required before dashboard access. | ✓ |
| Org + fleet + device registered | Strictest gate. | |

**User's choice:** Org + at least one fleet

| Option | Description | Selected |
|--------|-------------|----------|
| Query Supabase directly from Angular | Dashboard reads orgs + fleets tables. | ✓ |
| Auth-service /me returns onboarding state | Server-side query, returned in /me response. | |
| You decide | Leave to researcher. | |

**User's choice:** Dashboard queries Supabase directly

| Option | Description | Selected |
|--------|-------------|----------|
| Redirect to /dashboard immediately | Straight to main dashboard after onboarding. | |
| Redirect to /dashboard/fleet | Land on fleet view. | |
| You decide | Leave to planner. | ✓ |

**User's choice:** You decide

| Option | Description | Selected |
|--------|-------------|----------|
| Anon key + RLS | Standard client-side Supabase pattern. | ✓ |
| Auth-service proxies all queries | All data through auth-service service role key. | |
| You decide | Let researcher assess. | |

**User's choice:** Anon key + RLS

| Option | Description | Selected |
|--------|-------------|----------|
| Auth0 token directly to Supabase | Supabase trusts Auth0 JWT. RLS uses auth.uid() = Auth0 sub. | ✓ |
| Internal JWT to Supabase | Custom JWT issuer setup in Supabase. More complex. | |
| You decide | Let researcher assess. | |

**User's choice:** Auth0 token passed directly to Supabase for RLS

| Option | Description | Selected |
|--------|-------------|----------|
| Remove allowlistGuard | Replace all usages with onboardingGuard. | ✓ |
| Keep for now | Don't break working code. | |
| You decide | Leave to planner. | |

**User's choice:** Remove allowlistGuard

| Option | Description | Selected |
|--------|-------------|----------|
| Not set up — researcher documents setup | Auth0 JWT provider setup is a Phase 2 prerequisite. | ✓ |
| Already configured | No action needed. | |
| I'll set it up manually | Manual infra, treat as prerequisite. | |

**User's choice:** Not set up. Researcher documents Auth0 JWT provider configuration for Supabase.

**Extended discussion — full user journey:**
User raised broad question about employee onboarding, device payment, and overall flow. Discussion resolved:
- Employees: owner sends invite link. Employee registers via Auth0, auto-joins org.
- Invite flow included in Phase 2 (not deferred).
- Roles: Owner, Fleet Admin, Driver defined in model. Phase 2 MVP = Owner only.
- Device payment: upfront, out-of-band. No payment UI in app.
- Phase 2 = vehicle records only. Phase 5 = device assignment.
- Org creator = owner automatically.

---

## Vehicle Registration Scope

| Option | Description | Selected |
|--------|-------------|----------|
| IMEI / Device ID | Links vehicle to telemetry hardware. | |
| Make, Model, Year | Vehicle metadata. | ✓ |
| License plate / VIN | Physical vehicle identifier. | ✓ |
| Nickname / Label | Friendly name for fleet list. | ✓ |

**User's choice:** Make/Model/Year + Plate/VIN + Nickname. No IMEI in Phase 2.

| Option | Description | Selected |
|--------|-------------|----------|
| During onboarding only | 3rd onboarding step. | |
| Post-login in fleet management | After onboarding completes. | |
| Both — optional onboarding step + post-login | Skippable during onboarding, always accessible from fleet management. | ✓ |

**User's choice:** Both — optional onboarding step and fleet management page

| Option | Description | Selected |
|--------|-------------|----------|
| Separate tables: vehicles and devices | Clean data model, linked in Phase 5. | ✓ |
| One table: devices with vehicle metadata | Device row covers both. | |
| You decide — let researcher audit | Researcher reads migrations 004/005. | |

**User's choice:** Separate tables

| Option | Description | Selected |
|--------|-------------|----------|
| Edit vehicle details | Update nickname, plate, year. | ✓ |
| Remove/deactivate a vehicle | Soft delete. | ✓ |
| Multiple fleets per org | One org, multiple named fleets. | ✓ |
| None — just add vehicles for Phase 2 | CRUD comes later. | |

**User's choice:** Full CRUD + multiple fleets per org

| Option | Description | Selected |
|--------|-------------|----------|
| Dedicated fleet management page | /fleet-management or /settings/fleet. | ✓ |
| Inline in dashboard sidebar | Accessible from sidebar panel. | |
| You decide | Leave layout to planner. | |

**User's choice:** Dedicated /fleet-management page

| Option | Description | Selected |
|--------|-------------|----------|
| Follow existing Material patterns | Consistent with dashboard. Researcher identifies reusable forms. | ✓ |
| You decide | Let planner match style. | |

**User's choice:** Follow existing Material UI patterns

---

## Invite Flow Mechanics

| Option | Description | Selected |
|--------|-------------|----------|
| Owner copies link, sends themselves | No email from app. Simplest. | |
| App sends email automatically | Resend/SendGrid, owner enters email. | |
| Hybrid — owner chooses how to send | Generate link + optional email button via Resend. | ✓ |

**User's choice:** Hybrid — generate link, owner decides delivery. Optional email via Resend.
**Notes:** User asked for Claude's take on hybrid. Claude recommended: link-first, email optional via Resend (free tier, single API call).

| Option | Description | Selected |
|--------|-------------|----------|
| 7-day expiry, single-use | Standard invite pattern. | |
| 7-day expiry, multi-use | Shareable broadly within window. | |
| Owner chooses | Owner picks type at generation time. | ✓ |

**User's choice:** Owner chooses: single-use or multi-use, both 7-day expiry.

---

## Claude's Discretion

- Post-onboarding redirect target — planner picks based on UX flow
- `/fleet-management` vs `/settings/fleet` route naming — planner decides
- Whether Resend email delivery is wired in Phase 2 — depends on account availability

## Deferred Ideas

- Fleet Admin / Driver role enforcement — Phase 5+
- Device assignment (IMEI → vehicle) — Phase 5
- Payment/deposit UI — out-of-band for MVP
- Railway deployment config details for auth-service — planner scopes
- Resend email wiring — if account not available, link-only is acceptable
