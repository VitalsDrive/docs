---
phase: 02-auth-fleet-management
plan: "02-02"
subsystem: auth-service
tags: [auth, jwt, nestjs, user-provisioning]
dependency_graph:
  requires: [02-01]
  provides: [AUTH-03]
  affects: [dashboard, supabase/users]
tech_stack:
  added: [jest, ts-jest, "@nestjs/testing", "@types/jest"]
  patterns: [find-or-create, D-06 JWT payload, TDD RED/GREEN]
key_files:
  created:
    - packages/auth-service/src/auth/auth.service.spec.ts
  modified:
    - packages/auth-service/src/auth/auth.service.ts
    - packages/auth-service/src/auth/auth.controller.ts
    - packages/auth-service/src/users/users.service.ts
    - packages/auth-service/src/auth/jwt.strategy.ts
    - packages/auth-service/package.json
decisions:
  - "D-06 enforced: JWT payload restricted to {sub, email} only — orgId and roles removed"
  - "find-or-create pattern in exchangeAuth0Token uses findByExternalId before create"
  - "jwt.strategy.ts validate() now returns {sub, email} only — no userId alias, no orgId/roles"
metrics:
  duration: "~20 minutes"
  completed: "2026-05-13"
  tasks_completed: 2
  files_modified: 6
requirements:
  - AUTH-03
---

# Phase 2 Plan 02: Auth-Service Fix — User Provisioning, JWT Payload, /me Endpoint Summary

**One-liner:** NestJS auth-service wired with real user provisioning via find-or-create, JWT payload reduced to sub+email (D-06), GET /auth/me exposed with JWT guard, roles read from DB column.

## What Was Done

### Task 1 (RED): Failing tests written

Created `auth.service.spec.ts` with 5 tests covering:
1. verifyAuth0Token called + findByExternalId called + returns accessToken/refreshToken/user
2. New user (findByExternalId=null) triggers usersService.create
3. Existing user skips usersService.create
4. JWT payload has sub+email only — no orgId, no roles
5. validateToken returns {sub, email}

Added Jest + ts-jest + @nestjs/testing to devDependencies. Used `jest.mock('jwks-rsa')` and `jest.mock('jsonwebtoken')` to avoid ESM import issues. Tests ran RED (3 failures) as expected before implementation.

### Task 2 (GREEN): Three files fixed

**auth.service.ts:**
- Added `UsersService` constructor injection (2nd parameter)
- `exchangeAuth0Token()`: added find-or-create block after verifyAuth0Token; payload now `{ sub, email }` only (removed `orgId: 'default'` and `roles: ['owner']`)
- `refreshToken()`: stripped orgId/roles from re-signed payload

**auth.controller.ts:**
- Changed `/me` endpoint from `@Post` to `@Get`
- Added `@UseGuards(AuthGuard('jwt'))` decorator
- Returns `{ sub, email }` from validateToken claims

**users.service.ts:**
- Fixed `mapToUser()`: `roles: Array.isArray(dbRow.roles) ? dbRow.roles : ['member']` (was hardcoded `['owner']`)

**jwt.strategy.ts** (deviation Rule 2 — missing correctness fix):
- `validate()` now returns `{ sub, email }` only, matching D-06 payload shape

## Test Results

```
PASS src/auth/auth.service.spec.ts
  AuthService
    exchangeAuth0Token
      ✓ Test 1: calls verifyAuth0Token and findByExternalId, returns accessToken + refreshToken + user
      ✓ Test 2: when findByExternalId returns null (new user), calls usersService.create
      ✓ Test 3: when findByExternalId returns existing user, does NOT call usersService.create
      ✓ Test 4: JWT payload contains sub + email only — no orgId, no roles
      ✓ Test 5: validateToken returns { sub, email } matching the signed payload

Tests: 5 passed, 5 total
```

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added Jest/ts-jest to package.json**
- **Found during:** Task 1
- **Issue:** No test runner configured in package.json — `npm run test` script missing entirely
- **Fix:** Added jest, ts-jest, @nestjs/testing, @types/jest to devDependencies; added jest config block; added `test` and `lint` scripts
- **Files modified:** packages/auth-service/package.json

**2. [Rule 3 - Blocking] ESM module issue with jwks-rsa**
- **Found during:** Task 1 (first test run)
- **Issue:** jwks-rsa uses ESM exports which Jest CJS mode cannot parse
- **Fix:** Added `jest.mock('jwks-rsa')` and `jest.mock('jsonwebtoken')` at top of spec file before imports
- **Files modified:** packages/auth-service/src/auth/auth.service.spec.ts

**3. [Rule 2 - Missing correctness] jwt.strategy.ts returned userId alias + orgId/roles**
- **Found during:** Task 2
- **Issue:** `validate()` was returning `{ userId: payload.sub, ..., orgId, roles }` — contradicts D-06 and would have exposed stale fields to route handlers via `req.user`
- **Fix:** Changed to return `{ sub: payload.sub, email: payload.email }` only
- **Files modified:** packages/auth-service/src/auth/jwt.strategy.ts

## Git Status

No `.git` repository exists at `packages/auth-service/` — commits skipped for this package. All code changes were applied to disk. The auth-service package has not been initialized as a git repo under the VitalsDrive organization.

## Known Stubs

None — all wired logic uses real Supabase queries via UsersService.

## Threat Flags

No new security surface introduced beyond the plan's threat model. All T-02-02-0x threats addressed:
- T-02-02-01: verifyAuth0Token uses jwks-rsa RS256 verification (unchanged, already present)
- T-02-02-02: JWT payload is now {sub, email} only (fixed)
- T-02-02-03: JWT_SECRET from env var (unchanged)
- T-02-02-04: roles read from dbRow.roles (fixed)

## Self-Check

- [x] auth.service.spec.ts exists at packages/auth-service/src/auth/auth.service.spec.ts
- [x] auth.service.ts contains findByExternalId call
- [x] auth.service.ts does NOT contain orgId: 'default' in payload
- [x] auth.controller.ts contains @Get('me')
- [x] users.service.ts contains dbRow.roles
- [x] npm run test: 5 passed, 0 failed
- [x] npm run lint: exits 0 (tsc --noEmit clean)

## Self-Check: PASSED
