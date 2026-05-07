# VitalsDrive Auth & Onboarding Implementation Plan

## Context

The user answered the architectural questions:
1. **Organizations vs Fleets**: Separate `organizations` table that owns fleets
2. **Fleet naming on signup**: Prompt them to name it during onboarding
3. **Self-service signup**: Control access (invite-only for now); "bring your own hardware" is a future phase that affects pricing
4. **Deliverable**: Draft this PRD as part of the current scope

---

## Phase 1 — Data Model Evolution

### 1.1 New `organizations` Table

```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  plan TEXT NOT NULL DEFAULT 'starter' CHECK (plan IN ('starter', 'professional', 'enterprise')),
  can_add_devices BOOLEAN NOT NULL DEFAULT true,
  max_devices INTEGER,
  owner_id UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 1.2 Evolve `fleets` Table

```sql
ALTER TABLE fleets ADD COLUMN organization_id UUID REFERENCES organizations(id);
ALTER TABLE fleets DROP COLUMN owner_id; -- No longer needed; ownership is org-level
ALTER TABLE fleets ADD COLUMN created_at TIMESTAMPTZ NOT NULL DEFAULT now();
ALTER TABLE fleets ADD COLUMN updated_at TIMESTAMPTZ NOT NULL DEFAULT now();
```

### 1.3 Evolve `fleet_members` Table

```sql
ALTER TABLE fleet_members ADD COLUMN organization_id UUID REFERENCES organizations(id);
-- Role now spans org + fleet: 'owner' | 'admin' | 'member' | 'viewer'
```

### 1.4 Migration File

Save as: `packages/supabase/migrations/006_orgs_and_fleets.sql`

---

## Phase 2 — Auth Implementation (Clerk)

### 2.1 Users Table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_auth_id TEXT NOT NULL,
  external_auth_provider TEXT NOT NULL DEFAULT 'clerk',
  email TEXT NOT NULL UNIQUE,
  name TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_external_auth ON users(external_auth_id, external_auth_provider);
```

### 2.2 Signup Flow (Clerk)

- **Step 1 — Email/Password Signup**: Clerk SDK `signUp(email, password)`
- **Step 2 — Email Verification**: Clerk sends verification email
- **Step 3 — Webhook**: Clerk calls `/webhooks/clerk` → creates `users` record
- **Step 4 — Create Organization**: Prompt for org name → insert into `organizations`
- **Step 5 — Create First Fleet**: Prompt for fleet name → insert into `fleets`
- **Step 6 — Link User as Org Owner**: Insert into `fleet_members` with role `'owner'`
- **Step 7 — Redirect to Dashboard**

### 2.3 Login Flow (Clerk)

- Clerk SDK `signInWithPassword(email, password)`
- Receive Clerk JWT
- POST to auth service `/auth/exchange` with Clerk JWT
- Receive internal JWT → store in memory
- Redirect to dashboard

### 2.4 Logout Flow

- Clear internal JWT from memory
- Clerk `signOut()`
- Redirect to login

### 2.5 Session Management

- Auth service issues internal JWT (15 min expiry)
- HTTP interceptor attaches internal JWT to requests
- Auth guard checks internal JWT validity
- Refresh endpoint renews internal JWT

### 2.6 Auth Guard (`auth.guard.ts`)

```typescript
canActivate(): boolean {
  if (this.authService.isAuthenticated) return true;
  this.router.navigate(['/login']);
  return false;
}
```

### 2.7 Role-Based Guards

| Guard | Check |
|-------|-------|
| `auth.guard.ts` | Has valid internal JWT |
| `orgOwner.guard.ts` | User is org owner in fleet_members |
| `orgAdmin.guard.ts` | User is org owner or admin in fleet_members |
| `fleetAdmin.guard.ts` | User is fleet owner or admin in fleet_members |

Implementation: query `fleet_members` for user's role using `users.id`.

---

## Phase 3 — Onboarding Flow UI

### 3.1 Pages

- `/onboarding/organization` — "What's your company name?"
- `/onboarding/fleet` — "What's your first fleet called?"
- `/onboarding/complete` — Success + redirect to dashboard

### 3.2 Route Guards

- If authenticated but no org → redirect to `/onboarding/organization`
- If authenticated + org but no fleets → redirect to `/onboarding/fleet`

### 3.3 Components

- `onboarding-organization.component.ts`
- `onboarding-fleet.component.ts`
- `onboarding-complete.component.ts`

---

## Phase 4 — Update Existing Services

### 4.1 `FleetService`

- Add `organizationId` to all fleet queries
- Fetch fleets filtered by `organization_id`
- `createFleet()` must accept `organizationId` and set it

### 4.2 `VehicleService`

- All queries should be fleet-scoped (via `organization_id` joins)

### 4.3 `DeviceService`

- All queries should be fleet-scoped

### 4.4 `SidebarComponent`

- Show user email + logout button
- Show organization name

---

## Phase 5 — PRD: Onboarding Process

Document in `docs/Onboarding-Process.md`:

1. **Goal**: Get a new customer from signup to a working dashboard with their first fleet
2. **Pre-conditions**: User has an email + password
3. **Steps**: Signup → Verify email → Create org → Create fleet → Dashboard
4. **Access Control**: Invite-only for now; self-service signup with email verification
5. **Future**: BYOH (Bring Your Own Hardware) pricing phase
6. **Edge Cases**: Email already exists, org name taken, fleet name taken

---

## Phase 6 — Invite System (Future Phase)

- `invitations` table: `id, organization_id, email, role, invited_by, expires_at`
- Send invite link → user clicks → signs up → auto-joined to org
- Org owner/admin can invite members by email

---

## File Inventory

### New Files

| File | Purpose |
|------|---------|
| `packages/auth-service/` | NestJS auth microservice |
| `packages/auth-service/migrations/001_create_users_table.sql` | Users table with external_auth_id |
| `packages/auth-service/src/auth/auth.controller.ts` | Auth endpoints (exchange, refresh, validate) |
| `packages/auth-service/src/auth/auth.service.ts` | Token exchange logic |
| `packages/auth-service/src/auth/jwt.strategy.ts` | JWT validation strategy |
| `packages/auth-service/src/users/users.service.ts` | User CRUD |
| `packages/auth-service/src/middleware/jwt-auth.middleware.ts` | Token validation |
| `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts` | HTTP interceptor for internal JWT |
| `packages/dashboard/src/app/pages/onboarding/organization/onboarding-organization.component.ts` | Org name form |
| `packages/dashboard/src/app/pages/onboarding/fleet/onboarding-fleet.component.ts` | Fleet name form |
| `packages/dashboard/src/app/pages/onboarding/complete/onboarding-complete.component.ts` | Success page |
| `packages/dashboard/src/app/core/guards/org-owner.guard.ts` | Org owner guard |
| `packages/dashboard/src/app/core/guards/org-admin.guard.ts` | Org admin guard |
| `packages/dashboard/src/app/core/models/organization.model.ts` | Org types |
| `packages/dashboard/src/app/core/services/organization.service.ts` | Org CRUD service |

### Modified Files

| File | Change |
|------|--------|
| `packages/dashboard/src/app/core/services/auth.service.ts` | Replace Supabase Auth with Clerk |
| `packages/dashboard/src/app/core/services/supabase.service.ts` | Remove auth state |
| `packages/dashboard/src/app/core/services/fleet.service.ts` | Org-scoped queries |
| `packages/dashboard/src/app/core/services/vehicle.service.ts` | Org-scoped queries |
| `packages/dashboard/src/app/core/services/device.service.ts` | Org-scoped queries |
| `packages/dashboard/src/app/core/guards/auth.guard.ts` | Use internal JWT |
| `packages/dashboard/src/app/layout/sidebar/sidebar.component.ts` | User info + logout |
| `packages/dashboard/src/app/pages/login/login.component.ts` | Clerk login |
| `packages/dashboard/src/app/pages/signup/signup.component.ts` | Clerk signup |
| `packages/dashboard/src/app/app.routes.ts` | Onboarding routes + guards |

---

## Task Breakdown

### Wave 1 (Database & Schema)

1. **T1**: Write migration `001_create_users_table.sql` with `external_auth_id` + `external_auth_provider`
2. **T2**: Update `organizations` table for `users` FK (not `auth.users`)
3. **T3**: Update `fleet_members` table for `users` FK

### Wave 2 (Auth Service)

4. **T4**: Create `packages/auth-service/` NestJS project
5. **T5**: Implement auth controller (exchange, refresh, validate endpoints)
6. **T6**: Implement JWT strategy and token exchange logic
7. **T7**: Create user CRUD (sync from Clerk webhooks)

### Wave 3 (Dashboard Integration)

8. **T8**: Update `auth.service.ts` with Clerk SDK
9. **T9**: Create HTTP interceptor for internal JWT
10. **T10**: Update `auth.guard.ts` to use internal JWT
11. **T11**: Update role guards to use `users` table

### Wave 4 (Backend Services)

12. **T12**: Add JWT middleware to parser service
13. **T13**: Add JWT middleware to simulator service

### Wave 5 (UI & Onboarding)

14. **T14**: Update login/signup pages for Clerk
15. **T15**: Create onboarding components
16. **T16**: Update sidebar with user info

### Wave 6 (Verification)

17. **T17**: Test signup flow (Clerk → users table)
18. **T18**: Test login flow (Clerk JWT → internal JWT)
19. **T19**: Test dashboard → parser auth
20. **T20**: Verify end-to-end flow

---

## Verification

After each wave:
- Run `npm run build` in `packages/dashboard`
- Run `npm run build` in `packages/auth-service`
- Verify token exchange works
- Test parser/simulator auth
