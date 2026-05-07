# VitalsDrive Onboarding Process

**Status**: Draft  
**Version**: 1.0  
**Last Updated**: 2026-03-30

---

## 1. Overview

This document describes the onboarding process for new customers (organizations) signing up for VitalsDrive fleet management platform.

### 1.1 Goals

- Self-service signup for new organizations
- Access control via allowlist (admin approval required)
- Quick onboarding: signup → first fleet → dashboard in minutes
- Secure, role-based access control

### 1.2 Key Concepts

| Concept | Description |
|---------|-------------|
| **Organization** | The top-level tenant in VitalsDrive. Represents a customer company. Contains multiple fleets. |
| **Fleet** | A grouping of vehicles within an organization. Vehicles are assigned to fleets. |
| **User** | A person with access to an organization. Users have roles within the org. |
| **Device** | OBD2 tracker hardware installed in a vehicle. Pre-provisioned via IMEI lookup. |
| **Allowlist** | Access control mechanism. New organizations start as `allowlisted = false` and must be approved. |

---

## 2. User Roles

| Role | Scope | Permissions |
|------|-------|-------------|
| `owner` | Organization | Full control — manage org settings, all fleets, all members, billing |
| `admin` | Organization | Manage all fleets + members within org |
| `member` | Organization | View/manage vehicles in all org fleets |
| `viewer` | Organization | Read-only access across all org fleets |

**Design Decision**: MVP uses organization-level membership. A user's role applies to all fleets within their organization. Future versions may support per-fleet roles.

---

## 3. Signup Flow

### 3.1 User Journey

```
┌─────────────────┐
│  Visit /signup  │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Enter:                         │
│  - Organization name             │
│  - Email                        │
│  - Password                     │
│  - Confirm password             │
│  - Agree to Terms of Service    │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Create Auth User               │
│  + Create Organization          │
│  + Create first Fleet           │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Redirect to /onboarding        │
│  (Name your first fleet)       │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Redirect to /pending           │
│  (Account pending activation)   │
└─────────────────────────────────┘
```

### 3.2 Post-Signup State

After signup, the following exists:

| Entity | Status | Notes |
|--------|--------|-------|
| **Auth User** | Created | Email/password, not verified |
| **Organization** | `allowlisted = false` | Waiting for admin approval |
| **Fleet** | Created with user-provided name | Provisioning code generated |
| **Fleet Member** | `role = 'owner'` | User linked to org + fleet |

### 3.3 Signup Validation

- **Email**: Required, valid email format
- **Password**: Required, minimum 6 characters
- **Confirm Password**: Must match password
- **Organization Name**: Required, non-empty
- **Terms of Service**: Must be agreed to

### 3.4 Error Handling

| Error | Display |
|-------|---------|
| Email already exists | "An account with this email already exists" |
| Weak password | "Password must be at least 6 characters" |
| Network error | "Unable to connect. Please try again." |

---

## 4. Access Control

### 4.1 Allowlist Strategy

All new organizations start with `allowlisted = false`. This prevents access until an admin activates them.

**Flow**:
```
Signup → Account created but locked → Admin enables allowlist → User can login
```

**Why this approach?**
- Controls pricing: can verify payment/invoice before enabling
- Prevents unauthorized access during trial periods
- Allows manual review for enterprise customers

### 4.2 Admin Activation

An admin (currently via Supabase dashboard or direct database access) sets `organizations.allowlisted = true` for approved customers.

**Future**: Billing integration (Stripe) can auto-enable allowlist after successful payment.

---

## 5. Onboarding Flow

### 5.1 First-Time Setup

After signup, users are redirected to `/onboarding` to name their first fleet.

```
┌─────────────────────────────────┐
│  "Name Your Fleet"              │
│                                 │
│  "What would you like to call   │
│   your first fleet?"            │
│                                 │
│  [ Fleet Name input         ]  │
│                                 │
│  [ Skip for now ] [ Create ]   │
└─────────────────────────────────┘
```

### 5.2 Options

- **Create**: User enters a fleet name → fleet created with provisioning code
- **Skip**: Default fleet named "My Fleet" created

### 5.3 Post-Onboarding

After onboarding, `is_onboarding_complete = true` and user can access the dashboard.

---

## 6. Login Flow

### 6.1 Authentication

Users authenticate via Supabase Auth (email/password).

### 6.2 Post-Login Checks

```
Login successful
       │
       ▼
┌──────────────────────────┐
│ Check: Organization       │
│ allowlisted = true?      │
└────────────┬─────────────┘
             │
     ┌───────┴───────┐
     │               │
    NO              YES
     │               │
     ▼               ▼
┌──────────┐  ┌────────────────┐
│ /pending │  │ Check: Has      │
│          │  │ fleet?         │
└──────────┘  └───────┬────────┘
                      │
              ┌───────┴───────┐
              │               │
             NO              YES
              │               │
              ▼               ▼
        ┌──────────┐  ┌────────────┐
        │/onboard │  │ /dashboard  │
        │         │  │            │
        └──────────┘  └────────────┘
```

### 6.3 Route Guards

| Route | Guards Required | Redirect If Failed |
|-------|----------------|-------------------|
| `/login` | None | — |
| `/signup` | None | — |
| `/pending` | `authGuard` | `/login` |
| `/onboarding` | `authGuard` | `/login` |
| `/dashboard` | `authGuard`, `allowlistGuard`, `onboardingGuard` | respective guard failures |
| `/backoffice/*` | `authGuard`, `allowlistGuard`, `onboardingGuard`, `adminGuard` | respective guard failures |

---

## 7. Invite Team Members

### 7.1 Planned Feature (Out of Scope for MVP)

Future version will include:

- Invite by email
- Assign roles (admin, member, viewer)
- Pending invites tracking
- Email invite acceptance flow

### 7.2 Current Workaround

For MVP, new users are invited by:
1. Admin creates user via Supabase Auth dashboard
2. Admin manually adds `fleet_members` record linking user to organization

---

## 8. Session Management

### 8.1 Auth State Requirements

The frontend must maintain reactive state for:

- Current authenticated user
- Authentication status (logged in / out)
- Loading state during async operations
- Allowlist status (org approved to access dashboard)
- Onboarding completion status (user has at least one fleet)
- User role within their organization

### 8.2 State Synchronization

The Supabase Auth `onAuthStateChange` listener must be used to:
- Detect login/logout events
- Keep local auth state in sync with Supabase
- Trigger appropriate redirects on auth state changes

---

## 9. Future Phases

### 9.1 Phase 2: Self-Service Team Invites

- Users can invite team members by email
- Invitee receives email with acceptance link
- Role assignment during invite
- Pending/accepted invite tracking

### 9.2 Phase 3: Billing Integration

- Stripe integration for subscription management
- Auto-activate allowlist after successful payment
- Plan-based feature gates

### 9.3 Phase 4: Multi-Organization Support

- Users can belong to multiple organizations
- Org switcher in UI
- Session scoped to current org

### 9.4 Phase 5: Per-Fleet Roles

- Granular permissions per fleet
- Fleet-level admin role
- Member limited to specific fleets

### 9.5 Phase 6: "Bring Your Own Hardware"

- Self-service device purchase flow
- Device registration portal
- Hardware provisioning codes

---

## 10. Database Schema

See `architecture/data-model.md` for the full consolidated schema. Below are the onboarding-specific details.

### 10.1 organizations

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Organization identifier |
| name | TEXT | NOT NULL | Display name |
| allowlisted | BOOLEAN | NOT NULL, DEFAULT false | Whether org is approved for access |
| created_at | TIMESTAMPTZ | NOT NULL | Creation time |

### 10.2 fleet_members

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Membership identifier |
| fleet_id | UUID | FK → fleets | Fleet |
| user_id | UUID | FK → auth.users | User (maps to Supabase Auth) |
| organization_id | UUID | FK → organizations | Parent organization |
| role | TEXT | CHECK in (owner, admin, member, viewer) | User role |
| joined_at | TIMESTAMPTZ | NOT NULL | Join time |

**Constraint:** UNIQUE(fleet_id, user_id)

### 10.3 Fleet-Organization Link

Fleets must have an `org_id` column referencing `organizations(id)` with a NOT NULL constraint. This links the fleet-level auth model from Layer2 to the organization-level onboarding flow.

---

## 11. API Reference

### 11.1 Auth Service Methods

| Method | Description |
|--------|-------------|
| `signUp(email, password, orgName, fleetName?)` | Create account + org + fleet |
| `signIn(email, password)` | Authenticate, check status, redirect |
| `signOut()` | Clear session, redirect to login |
| `completeOnboarding(fleetName)` | Create first fleet |
| `isAdmin()` | Check if current user is admin/owner |
| `isOwner()` | Check if current user is owner |
| `getUserEmail()` | Get current user email |
| `getUserId()` | Get current user ID |

### 11.2 Supabase RPC Functions

| Function | Description |
|----------|-------------|
| `create_organization_and_first_fleet(p_org_name, p_fleet_name, p_user_id)` | Create org + fleet + membership |
| `check_user_login_status(p_user_id)` | Check allowlist + membership |
| `is_onboarding_complete(p_user_id)` | Check if user has a fleet |
| `is_user_org_allowlisted()` | Convenience check for allowlist |
| `get_user_primary_org(p_user_id)` | Get user's org ID |

---

## 12. Security Considerations

### 12.1 Row Level Security

- All tables have RLS enabled
- Policies restrict access based on `fleet_members` membership
- Service role required for admin operations

### 12.2 Input Validation

- All user inputs are validated client-side and server-side
- SQL injection prevented via parameterized queries
- XSS prevented via Angular's default escaping

### 12.3 Password Requirements

- Minimum 6 characters
- Supabase handles hashing (bcrypt)
- No password strength enforcement in MVP (future: zxcvbn)

---

## 13. Open Questions

1. **Email Verification**: Should we require email verification before allowing login? Currently deferred.

2. **Password Reset**: Implement self-service password reset via Supabase? Currently not implemented.

3. **Organization Name Uniqueness**: Should org names be unique? Currently not enforced.

4. **Fleet Name Uniqueness**: Should fleet names be unique within an org? Currently not enforced.

5. **Multiple Organizations**: When to support users belonging to multiple orgs? Deferred to Phase 4.

---

## 14. Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-03-30 | Initial draft |
