# VitalsDrive — Authentication Architecture

**Version:** 1.0
**Status:** Draft
**Last Updated:** April 2026
**Document Owner:** VitalsDrive Engineering

---

## 1. Overview

VitalsDrive uses **Supabase Auth** for all authentication and authorization. The Auth0 integration and custom NestJS auth-service have been removed.

### 1.1 Architecture Comparison

| Component | Before (Auth0) | After (Supabase Auth) |
|---|---|---|
| Authentication | Auth0 (external) | Supabase Auth (built-in) |
| Token Exchange | auth-service (NestJS) | Removed |
| Session | Custom JWT from auth-service | Supabase session token |
| Access Control | Custom middleware | Supabase RLS policies |
| Dashboard | @auth0/auth0-angular | Supabase SDK directly |
| Realtime | Anon key | Session token |

---

## 2. Authentication Flow

```
Browser Dashboard
    │
    │ 1. signIn(email/password or OAuth)
    ▼
Supabase Auth
    │
    │ 2. session.access_token
    ▼
Supabase Client (authenticated)
    │
    │ 3. REST API calls with session token
    ▼
Supabase PostgreSQL + RLS
```

### 2.1 Flow Description

1. User signs in via Supabase Auth (email/password or OAuth provider)
2. Supabase returns a session with `access_token` and `refresh_token`
3. Session token is stored in browser and auto-refreshed by SDK
4. All Supabase requests include the session token
5. RLS policies at the database level evaluate `auth.uid()` for access control

---

## 3. Data Model

### 3.1 Profiles Table

A public profile table that mirrors `auth.users`:

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary key, references `auth.users(id)` |
| email | TEXT | User email |
| full_name | TEXT | Display name |
| avatar_url | TEXT | Profile picture URL |
| created_at | TIMESTAMPTZ | Account creation time |

### 3.2 Fleet Membership

Links users to fleets with role-based access:

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| user_id | UUID | References `auth.users(id)` |
| fleet_id | UUID | References `fleets(id)` |
| role | TEXT | `owner`, `admin`, or `member` |
| created_at | TIMESTAMPTZ | Join time |

---

## 4. RLS Policy Strategy

### 4.1 Core Principle

All access control is enforced at the database level via Row Level Security. The Supabase service role key is used only by the ingestion server (Layer 1) and bypasses RLS.

### 4.2 Policy Reference

| Table | Policy | Expression |
|---|---|---|
| profiles | SELECT/UPDATE own | `auth.uid() = id` |
| fleets | SELECT (member) | User is in `fleet_members` for this fleet |
| fleets | INSERT/UPDATE (owner) | `auth.uid() = owner_id` |
| vehicles | SELECT (member) | User is in `fleet_members` for vehicle's fleet |
| vehicles | INSERT (fleet owner) | `auth.uid() = fleet owner_id` |
| telemetry_logs | SELECT (member) | User is in `fleet_members` for vehicle's fleet |
| alerts | SELECT (member) | User is in `fleet_members` for vehicle's fleet |

---

## 5. Supabase Auth Configuration

| Setting | Value |
|---|---|
| Email signup | Enabled |
| Email confirmation | Optional (OFF recommended for faster onboarding) |
| Min password length | 8 characters |
| OAuth providers | Google (enabled), GitHub (optional) |
| Site URL | `https://vitalsdrive.com` |
| Redirect URLs | `http://localhost:4200/auth/callback` (dev), `https://vitalsdrive.com/auth/callback` (prod) |

---

## 6. Session Management

The Supabase SDK handles:
- Token storage in browser
- Automatic token refresh before expiry
- Session detection on page load
- Auth state change events (for UI reactivity)

---

## 7. Migration History (April 2026)

| Component | Action |
|---|---|
| Auth0 integration | Removed |
| @auth0/auth0-angular dependency | Removed from dashboard |
| packages/auth-service/ (NestJS) | Deleted |
| Token exchange flow (/auth/exchange) | Removed |
| Custom JWT validation middleware | Removed |
| RLS policies | Updated to use `auth.uid()` directly |

---

## 8. Onboarding & Roles

**Reference:** `PRD-Onboarding.md`

### 8.1 User Roles

| Role | Permissions |
|---|---|
| owner | Full access to fleet, can manage members, billing, vehicles |
| admin | Fleet management, vehicle assignment, alert management |
| member | View fleet vehicles, telemetry, and alerts |

### 8.2 Onboarding Flow

1. User signs up via Supabase Auth
2. User is presented with onboarding step to create their first fleet
3. Fleet owner status is granted upon creation
4. User can then add vehicles (via device IMEI pairing) and invite members

---

*Document Version: 1.0 — Last Updated: April 2026*

### Related Documents

| Document | Path |
|---|---|
| Main PRD | `../VitalsDrive_PRD.md` |
| Consolidated Data Model | `data-model.md` |
| System Architecture | `system-overview.md` |
| Onboarding PRD | `../PRD-Onboarding.md` |
| Data Storage PRD | `../PRD-Layer2-Data-Storage.md` |
