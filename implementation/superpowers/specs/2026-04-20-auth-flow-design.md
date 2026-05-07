# Auth Flow Design - Clerk + Custom Auth Service

**Status:** Approved
**Version:** 1.0
**Created:** 2026-04-20

---

## 1. Overview

Replace Supabase Auth with Clerk as the authentication provider, with a custom NestJS auth service that issues internal JWTs for backend services. This provides flexibility to migrate auth providers in the future without affecting backend services.

---

## 2. Architecture

### 2.1 Components

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Clerk     │────▶│  Auth       │────▶│  Internal      │
│  (auth)   │     │  Service    │     │  Service Token  │
└─────────────┘     │  (NestJS)  │     │  (JWT)          │
                    └──────────────┘     └────────┬────────┘
                                                  │
┌─────────────────────────────────────────────────┘
│  Dashboard (Angular)
│  - HTTP interceptor attaches internal token
└─────────────────────────────────────────────────────┘
                                                  │
              ┌────────────────────────────────────┤
              ▼                                    ▼
     ┌─────────────────┐            ┌─────────────────┐
     │  Parser         │            │  Simulator      │
     │  (validates    │            │  (validates     │
     │   internal JWT) │            │   internal JWT) │
     └─────────────────┘            └─────────────────┘
```

### 2.2 Token Flow

1. **User Login:** Clerk handles email/password, Google OAuth
2. **Token Exchange:** Dashboard exchanges Clerk JWT for internal JWT via auth service
3. **Internal JWT:** Short-lived (15 min), signed with shared secret
4. **Refresh:** Auth service handles refresh, can re-validate with Clerk if needed
5. **Backend Calls:** Parser/simulator validate internal JWT via shared middleware

---

## 3. Database Schema

### 3.1 New `users` Table

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

**Design Rationale:** Generic column names (`external_auth_id`, `external_auth_provider`) make the table provider-agnostic. Future migrations to Auth0 or other providers require only adding new rows or updating the provider field.

### 3.2 Updated `organizations` Table

```sql
ALTER TABLE organizations
  ADD COLUMN owner_id UUID REFERENCES users(id);

-- Remove old reference to auth.users if exists
-- owner_id now points to users.id instead of auth.users(id)
```

### 3.3 Updated `fleet_members` Table

```sql
ALTER TABLE fleet_members
  ADD COLUMN user_id UUID REFERENCES users(id);

-- Remove old user_id reference if it pointed to auth.users
```

---

## 4. Auth Service (NestJS)

### 4.1 Package

`packages/auth-service/`

### 4.2 Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/auth/exchange` | POST | Exchange Clerk JWT for internal JWT |
| `/auth/refresh` | POST | Refresh internal JWT |
| `/auth/validate` | POST | Validate internal JWT (for backend services) |
| `/auth/me` | GET | Get current user from internal JWT |
| `/webhooks/clerk` | POST | Handle Clerk webhooks (user created, updated, deleted) |

### 4.3 Internal JWT Payload

```typescript
interface InternalJwtPayload {
  sub: string;          // user ID (UUID from users table)
  email: string;
  orgId: string;         // current organization
  roles: string[];       // ['owner', 'admin', 'member', 'viewer']
  iat: number;
  exp: number;
}
```

### 4.4 Token Configuration

| Token | Expiry | Storage |
|-------|--------|---------|
| Internal Access Token | 15 minutes | Memory (dashboard) |
| Internal Refresh Token | 7 days | HTTP-only cookie or memory |

### 4.5 Dependencies

```json
{
  "@nestjs/common": "^10.x",
  "@nestjs/core": "^10.x",
  "@nestjs/jwt": "^10.x",
  "@nestjs/passport": "^10.x",
  "passport": "^0.7.x",
  "clerk": "^4.x"
}
```

---

## 5. Dashboard Integration

### 5.1 Updated Auth Service

`packages/dashboard/src/app/core/services/auth.service.ts`

- Replace Supabase Auth calls with Clerk SDK
- Add token exchange: Clerk JWT → internal JWT
- Store internal JWT in memory (not localStorage for security)
- HTTP interceptor attaches internal JWT to outgoing requests

### 5.2 HTTP Interceptor

`packages/dashboard/src/app/core/interceptors/auth.interceptor.ts`

```typescript
intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
  const token = this.authService.getInternalToken();
  if (token) {
    req = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` }
    });
  }
  return next.handle(req);
}
```

### 5.3 Updated Guards

All existing guards (`auth.guard.ts`, `org-owner.guard.ts`, `org-admin.guard.ts`, `fleet-admin.guard.ts`) update to use `users` table via auth service instead of checking Supabase auth state.

---

## 6. Backend Service Integration

### 6.1 Parser Service

`packages/parser/`

- Add NestJS auth middleware to validate internal JWT
- Extract user ID and roles from token
- Reject requests without valid internal JWT

### 6.2 Simulator Service

`packages/simulator/`

- Same auth middleware as parser

### 6.3 Shared Auth Middleware

```typescript
// packages/auth-service/src/middleware/jwt-auth.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class JwtAuthMiddleware implements NestMiddleware {
  constructor(private jwtService: JwtService) {}

  use(req: Request, res: Response, next: NextFunction) {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'No token provided' });
    }

    const token = authHeader.substring(7);
    try {
      const payload = this.jwtService.verify(token);
      req['user'] = payload;
      next();
    } catch (err) {
      return res.status(401).json({ error: 'Invalid token' });
    }
  }
}
```

---

## 7. Clerk Integration

### 7.1 Clerk Configuration

1. Create Clerk account and application
2. Configure email/password and Google OAuth
3. Optional: Enable MFA (user opt-in)
4. Set up webhook endpoint: `https://api.vitalsdrive.com/webhooks/clerk`

### 7.2 Webhook Events

| Event | Action |
|-------|--------|
| `user.created` | Create `users` record, trigger onboarding |
| `user.updated` | Update `users` record |
| `user.deleted` | Soft-delete or archive `users` record |

### 7.3 Environment Variables

```env
# Clerk
CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

# Auth Service
JWT_SECRET=your-256-bit-secret
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
```

---

## 8. Signup Flow

### 8.1 New User Signup

```
1. User visits /signup
2. Clerk signUp(email, password)
3. Clerk sends verification email
4. User clicks verification link
5. Clerk webhook → create users record
6. Redirect to /onboarding/organization
7. User creates organization
8. User creates first fleet
9. Redirect to dashboard
```

### 8.2 Login Flow

```
1. User visits /login
2. Clerk signInWithPassword(email, password)
3. Receive Clerk JWT
4. POST /auth/exchange with Clerk JWT
5. Receive internal JWT
6. Store internal JWT, redirect to dashboard
```

---

## 9. Files to Create/Modify

### New Files

| File | Purpose |
|------|---------|
| `packages/auth-service/package.json` | NestJS auth service |
| `packages/auth-service/src/app.module.ts` | NestJS app module |
| `packages/auth-service/src/auth/auth.controller.ts` | Auth endpoints |
| `packages/auth-service/src/auth/auth.service.ts` | Auth logic |
| `packages/auth-service/src/auth/jwt.strategy.ts` | JWT strategy |
| `packages/auth-service/src/users/users.service.ts` | User CRUD |
| `packages/auth-service/src/users/users.controller.ts` | User endpoints |
| `packages/auth-service/src/middleware/jwt-auth.middleware.ts` | Token validation |
| `packages/auth-service/migrations/001_create_users_table.sql` | Users table |
| `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts` | HTTP interceptor |

### Modified Files

| File | Change |
|------|--------|
| `packages/dashboard/src/app/core/services/auth.service.ts` | Replace Supabase Auth with Clerk |
| `packages/dashboard/src/app/core/guards/auth.guard.ts` | Use internal JWT |
| `packages/dashboard/src/app/core/guards/org-owner.guard.ts` | Use users table |
| `packages/dashboard/src/app/core/guards/org-admin.guard.ts` | Use users table |
| `packages/dashboard/src/app/core/guards/fleet-admin.guard.ts` | Use users table |
| `packages/dashboard/src/app/app.routes.ts` | Update guard references |
| `packages/dashboard/src/app/pages/login/login.component.ts` | Clerk login |
| `packages/dashboard/src/app/pages/signup/signup.component.ts` | Clerk signup |
| `packages/dashboard/src/app/core/services/supabase.service.ts` | Remove auth state |

### Updated PRDs

| File | Change |
|------|--------|
| `docs/Auth-Onboarding-Plan.md` | Supabase Auth → Clerk |
| `docs/Plan-Auth-Onboarding.md` | Supabase Auth → Clerk |
| `docs/PRD-Layer2-Data-Storage.md` | users table schema |
| `docs/Onboarding-Process.md` | auth.users → users |
| `docs/PRD-Onboarding.md` | auth.users → users |

---

## 10. Migration Considerations

### 10.1 Fresh Start

No existing users to migrate. This is a greenfield deployment.

### 10.2 Future Migration from Clerk

If migrating from Clerk to another provider (e.g., Auth0):

1. Add new `external_auth_provider` values
2. Update auth service to accept new provider JWTs
3. No changes to internal JWT format or backend services
4. Users table remains the source of truth

---

## 11. Security Considerations

- Internal JWT secret must be shared between auth service and backend services
- Consider rotating JWT secret periodically
- Store internal JWT in memory, not localStorage (prevents XSS theft)
- Use HTTP-only cookies for refresh token storage
- Validate internal JWT on every backend request (parser, simulator)

---

## 12. Acceptance Criteria

- [ ] Users can sign up with email/password via Clerk
- [ ] Users can log in with Google via Clerk
- [ ] Dashboard exchanges Clerk JWT for internal JWT
- [ ] Parser service validates internal JWT on incoming requests
- [ ] Simulator service validates internal JWT on incoming requests
- [ ] Role-based guards work with users table
- [ ] Webhooks sync Clerk user events to users table
- [ ] MFA is available as user option (enabled in Clerk dashboard)
- [ ] Internal JWT has 15-minute expiry
- [ ] Refresh token has 7-day expiry
