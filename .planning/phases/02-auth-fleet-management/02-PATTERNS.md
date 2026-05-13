# Phase 2: Auth & Fleet Management — Pattern Map

**Mapped:** 2026-05-12
**Files analyzed:** 12 new/modified files
**Analogs found:** 11 / 12

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|---|---|---|---|---|
| `packages/auth-service/src/auth/auth.service.ts` | service | request-response | `packages/auth-service/src/auth/auth.service.ts` (self — modify) | exact |
| `packages/auth-service/src/auth/auth.controller.ts` | controller | request-response | `packages/auth-service/src/auth/auth.controller.ts` (self — modify) | exact |
| `packages/auth-service/src/users/users.service.ts` | service | CRUD | `packages/auth-service/src/users/users.service.ts` (self — modify) | exact |
| `packages/dashboard/src/app/core/services/auth.service.ts` | service | request-response | `packages/dashboard/src/app/core/services/auth.service.ts` (self — modify) | exact |
| `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts` | middleware | request-response | `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts` (self — modify) | exact |
| `packages/dashboard/src/app/core/guards/auth.guard.ts` | middleware | request-response | `packages/dashboard/src/app/core/guards/auth.guard.ts` (self — modify) | exact |
| `packages/dashboard/src/app/core/guards/onboarding.guard.ts` | middleware | request-response | `packages/dashboard/src/app/core/guards/auth.guard.ts` (onboardingGuard fn) | exact |
| `packages/dashboard/src/app/app.routes.ts` | config | request-response | `packages/dashboard/src/app/app.routes.ts` (self — modify) | exact |
| `packages/dashboard/src/environments/environment.ts` | config | — | `packages/dashboard/src/environments/environment.development.ts` | exact |
| `packages/supabase/migrations/009_phase2_auth_schema.sql` | migration | CRUD | `packages/supabase/migrations/006_orgs_and_fleets.sql` | role-match |
| `packages/dashboard/src/app/features/fleet-management/` (new page + service) | component + service | CRUD | `packages/dashboard/src/app/core/services/fleet.service.ts` + `onboarding-fleet.component.ts` | role-match |
| `packages/dashboard/src/app/core/services/vehicle.service.ts` | service | CRUD | `packages/dashboard/src/app/core/services/fleet.service.ts` | role-match |

---

## Pattern Assignments

### `packages/auth-service/src/auth/auth.service.ts` (service, request-response)

**Analog:** self — existing file, targeted modifications only.

**Current imports** (lines 1–4) — keep as-is, add UsersService:
```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import * as jwksClient from 'jwks-rsa';
import * as jwt from 'jsonwebtoken';
// ADD:
import { UsersService } from '../users/users.service';
```

**Current env-var pattern** (lines 6–7) — already correct, verify injected via Railway:
```typescript
const AUTH0_DOMAIN = process.env.AUTH0_DOMAIN || 'ronbiter.auth0.com';
const AUTH0_AUDIENCE = process.env.AUTH0_AUDIENCE || 'https://ronbiter.auth0.com/api/v2/';
```

**Core pattern to REPLACE in `exchangeAuth0Token`** (lines 19–39):

Current broken section (lines 25–30) embeds `orgId` + `roles` in JWT and skips user provisioning:
```typescript
// REMOVE this payload block:
const payload = {
  sub: userId,
  email: email,
  orgId: 'default',      // D-06: must NOT be in JWT
  roles: ['owner'],      // D-06: must NOT be in JWT
};
```

Replace with (D-04, D-06):
```typescript
// Wire UsersService.findOrCreate into exchange (D-04):
let user = await this.usersService.findByExternalId(userId, 'auth0');
if (!user) {
  user = await this.usersService.create({
    externalAuthId: userId,
    externalAuthProvider: 'auth0',
    email,
  });
}

// JWT payload = sub + email ONLY (D-06):
const payload = { sub: userId, email };
const accessToken = this.jwtService.sign(payload, { expiresIn: '15m' });
const refreshToken = this.jwtService.sign(payload, { expiresIn: '7d' });
return { accessToken, refreshToken, user: { id: user.id, email } };
```

**`refreshToken` fix** (lines 61–72) — strip orgId/roles from re-signed token:
```typescript
// Replace payload line 65:
{ sub: payload.sub, email: payload.email }  // not orgId/roles
```

**Error handling pattern** — existing, copy throughout:
```typescript
} catch (error) {
  throw new UnauthorizedException('Invalid Auth0 token');
}
```

---

### `packages/auth-service/src/auth/auth.controller.ts` (controller, request-response)

**Analog:** self — existing file. Add `GET /auth/me` using `@Get` + `@UseGuards(AuthGuard('jwt'))`.

**Current pattern** (lines 1–44) — all endpoints use try/catch + re-throw as `UnauthorizedException`:
```typescript
@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('exchange')
  async exchangeToken(@Body() dto: ExchangeTokenDto) {
    try {
      return await this.authService.exchangeAuth0Token(dto.auth0Token);
    } catch (error) {
      throw new UnauthorizedException('Failed to exchange token');
    }
  }
```

**`/auth/me` endpoint** — current uses `@Post` and returns only token claims. D-07 requires `GET`, returning `sub` + `email` from validated token:
```typescript
// Change @Post('me') to @Get('me'), add @UseGuards(AuthGuard('jwt'))
import { Get, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Get('me')
@UseGuards(AuthGuard('jwt'))
async getCurrentUser(@Headers('authorization') authHeader: string) {
  if (!authHeader?.startsWith('Bearer ')) {
    throw new UnauthorizedException('No token provided');
  }
  const token = authHeader.substring(7);
  return await this.authService.validateToken(token);
}
```

---

### `packages/auth-service/src/users/users.service.ts` (service, CRUD)

**Analog:** self — existing file. Fix `mapToUser` roles hardcode.

**Current broken pattern** (lines 98–108):
```typescript
private mapToUser(dbRow: any): User {
  return {
    ...
    roles: ['owner'], // TODO: implement proper role lookup — MUST FIX
  };
}
```

**Fix** — read from `dbRow.roles` column (added by migration 008):
```typescript
roles: Array.isArray(dbRow.roles) ? dbRow.roles : ['member'],
```

**Supabase client pattern** (lines 18–22) — service role key, keep unchanged:
```typescript
constructor() {
  const supabaseUrl = process.env.SUPABASE_URL || 'http://localhost:54321';
  const supabaseKey = process.env.SUPABASE_SERVICE_ROLE_KEY || '';
  this.supabase = createClient(supabaseUrl, supabaseKey);
}
```

**CRUD pattern** (lines 24–83) — all use `.from('users').select/insert/update.single()`, check `error || !data`, return mapped object or null/throw. Copy this exact pattern for any new user methods.

---

### `packages/dashboard/src/app/core/services/auth.service.ts` (service, request-response)

**Analog:** self — existing file, multiple targeted fixes.

**Current imports** (lines 1–5) — add `toSignal`, `HttpClient`, remove direct `fetch`:
```typescript
import { Injectable, signal, computed, inject, effect } from "@angular/core";
import { takeUntilDestroyed, toSignal } from "@angular/core/rxjs-interop";
import { Router } from "@angular/router";
import { AuthService as Auth0Service, User } from "@auth0/auth0-angular";
import { firstValueFrom } from "rxjs";
// ADD:
import { HttpClient } from "@angular/common/http";
import { environment } from "../../../environments/environment";
```

**Critical fix — `isAuthenticated` signal** (line 47–49):
```typescript
// REMOVE (returns Promise<boolean> not boolean — always truthy):
readonly isAuthenticated = computed(() => {
  return firstValueFrom(this.auth0.isAuthenticated$);
});

// REPLACE WITH:
readonly isAuthenticated = toSignal(this.auth0.isAuthenticated$, { initialValue: false });
```

**localStorage persistence pattern** — add after class field declarations:
```typescript
private readonly TOKEN_KEY = 'vd_access_token';
private readonly REFRESH_KEY = 'vd_refresh_token';

// On app init, restore from localStorage:
constructor() {
  this.internalToken = localStorage.getItem(this.TOKEN_KEY);
  // ... existing auth0.user$ subscription
}
```

**`exchangeToken` fix** — replace hardcoded `localhost:3001` (line 89) with env var:
```typescript
// REMOVE:
const response = await fetch('http://localhost:3001/auth/exchange', { ... });

// REPLACE WITH (using HttpClient + environment):
private async exchangeToken(auth0Token: string): Promise<void> {
  const data = await firstValueFrom(
    this.http.post<ExchangeResponse>(`${environment.authServiceUrl}/auth/exchange`, { auth0Token })
  );
  this.internalToken = data.accessToken;
  localStorage.setItem(this.TOKEN_KEY, data.accessToken);
  localStorage.setItem(this.REFRESH_KEY, data.refreshToken);
}
```

**`initializeUserState` fix** — replace mocks (lines 167–180) with real Supabase queries per Pattern 6 in RESEARCH.md:
```typescript
// REMOVE all hardcoded `true` values and 'mock-org-1'
// REPLACE with supabase.client.from('fleet_members').select(...) queries
// Copy query structure from OrganizationService.loadOrganizations() and FleetService.loadFleets()
```

**`refreshTokens` method** — new method needed for interceptor:
```typescript
async refreshTokens(): Promise<void> {
  const refreshToken = localStorage.getItem(this.REFRESH_KEY);
  if (!refreshToken) throw new Error('No refresh token');
  const data = await firstValueFrom(
    this.http.post<{ accessToken: string }>(`${environment.authServiceUrl}/auth/refresh`, { refreshToken })
  );
  this.internalToken = data.accessToken;
  localStorage.setItem(this.TOKEN_KEY, data.accessToken);
}
```

**`signOut` fix** — clear localStorage:
```typescript
async signOut(): Promise<{ error: any }> {
  this.internalToken = null;
  localStorage.removeItem(this.TOKEN_KEY);
  localStorage.removeItem(this.REFRESH_KEY);
  // ... rest unchanged
}
```

---

### `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts` (middleware, request-response)

**Analog:** self — existing file (lines 1–30). Add 401 retry with refresh.

**Current pattern** (lines 14–29) — token inject only, no retry:
```typescript
intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
  const token = this.authService.getInternalToken();
  if (token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }
  return next.handle(req);
}
```

**Add these imports** at top:
```typescript
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent, HttpErrorResponse } from '@angular/common/http';
import { Observable, from, throwError } from 'rxjs';
import { catchError, switchMap } from 'rxjs/operators';
```

**Replace `intercept` body** with 401 retry (D-02, D-03 — RESEARCH.md Pattern 3):
```typescript
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

---

### `packages/dashboard/src/app/core/guards/auth.guard.ts` (middleware, request-response)

**Analog:** self — existing file. Remove `allowlistGuard` export (D-13).

**Keep unchanged** (lines 7–32): `authCheck` fn, `authGuard`, `authGuardChild` exports.

**Keep unchanged** (lines 60–73): `onboardingGuard` — already correct logic.

**Keep unchanged** (lines 82–125): `onboardingStepGuard` — but fix: `isAuthenticated()` returning boolean after the signal fix above makes lines 15, 92 work correctly.

**Remove entirely** (lines 40–53): `allowlistGuard` export.

**Remove from imports** in file: `isAllowlisted` usage.

---

### `packages/dashboard/src/app/app.routes.ts` (config, request-response)

**Analog:** self — existing file.

**Remove from import** (line 2): `allowlistGuard`.

**Replace all usages** of `allowlistGuard` in `canActivate` arrays — 4 occurrences at lines 81, 90, 99, 108, 113:
```typescript
// BEFORE:
canActivate: [allowlistGuard, onboardingGuard],
// AFTER:
canActivate: [onboardingGuard],
```

**Add new routes** after the `backoffice` block (before closing shell children), following the lazy-load pattern at lines 10–13:
```typescript
{
  path: 'fleet-management',
  loadComponent: () =>
    import('./features/fleet-management/fleet-management.component').then(
      (m) => m.FleetManagementComponent,
    ),
  canActivate: [onboardingGuard],
  title: 'Fleet Management — VitalsDrive',
},
{
  path: 'settings/invite',
  loadComponent: () =>
    import('./features/settings/invite/invite.component').then(
      (m) => m.InviteComponent,
    ),
  canActivate: [onboardingGuard],
  title: 'Invite Members — VitalsDrive',
},
```

**Add vehicle onboarding route** inside the `onboarding` children block after `fleet`:
```typescript
{
  path: 'vehicle',
  loadComponent: () =>
    import('./pages/onboarding/vehicle/onboarding-vehicle.component').then(
      (m) => m.OnboardingVehicleComponent,
    ),
  title: 'Add Your First Vehicle — VitalsDrive',
},
```

---

### `packages/dashboard/src/environments/environment.ts` (config)

**Analog:** `packages/dashboard/src/environments/environment.development.ts` (lines 1–8).

**Current dev environment structure:**
```typescript
export const environment = {
  production: false,
  supabase: {
    url: 'https://odwctmlawibhaclptsew.supabase.co',
    anonKey: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
  },
};
```

**Add `authServiceUrl` and `auth0` block to BOTH environment files** (D-08, D-09):
```typescript
export const environment = {
  production: false,                         // true in environment.ts
  authServiceUrl: 'http://localhost:3001',   // Railway URL in environment.ts
  supabase: {
    url: 'https://odwctmlawibhaclptsew.supabase.co',
    anonKey: '...',
  },
  auth0: {
    domain: 'ronbiter.auth0.com',
    clientId: 'YOUR_AUTH0_SPA_CLIENT_ID',   // from Auth0 dashboard → Applications
    audience: 'https://ronbiter.auth0.com/api/v2/',
  },
};
```

---

### `packages/supabase/migrations/009_phase2_auth_schema.sql` (migration, CRUD)

**Analog:** `packages/supabase/migrations/006_orgs_and_fleets.sql` — copy `BEGIN;`/`COMMIT;` wrapper, `ALTER TABLE`, `CREATE TABLE IF NOT EXISTS`, RLS `CREATE POLICY` with `EXISTS(SELECT 1 FROM ...)` patterns.

**RLS policy pattern** from migration 006 (lines 246–265) — use EXISTS not IN to avoid recursion:
```sql
CREATE POLICY "policy name" ON table_name
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM fleet_members fm
            WHERE fm.user_id = auth.uid()
            AND fm.organization_id = table_name.organization_id
        )
    );
```

**Migration 009 must contain (in order):**

1. Drop FK on `users.id` referencing `auth.users`:
```sql
ALTER TABLE users DROP CONSTRAINT IF EXISTS users_id_fkey;
-- Make id a plain UUID PK (no auth.users FK):
ALTER TABLE users ALTER COLUMN id SET DEFAULT gen_random_uuid();
```

2. Fix `external_auth_provider` default:
```sql
ALTER TABLE users ALTER COLUMN external_auth_provider SET DEFAULT 'auth0';
```

3. Add `nickname` to vehicles, make `vin` nullable (D-23, RESEARCH Pattern 7):
```sql
ALTER TABLE vehicles ADD COLUMN IF NOT EXISTS nickname VARCHAR(100);
ALTER TABLE vehicles ALTER COLUMN vin DROP NOT NULL;
```

4. Change `fleet_members.user_id` from UUID to TEXT for Auth0 sub compatibility (RESEARCH Pitfall 3):
```sql
ALTER TABLE fleet_members ALTER COLUMN user_id TYPE TEXT;
ALTER TABLE organizations ALTER COLUMN owner_id TYPE TEXT;
```

5. Add `org_invites` table (D-19, RESEARCH Pattern 8):
```sql
CREATE TABLE IF NOT EXISTS org_invites (
  id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  token      TEXT        NOT NULL UNIQUE,
  org_id     UUID        NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  created_by TEXT        NOT NULL,  -- Auth0 sub (TEXT not UUID)
  type       TEXT        NOT NULL CHECK (type IN ('single-use', 'multi-use')),
  expires_at TIMESTAMPTZ NOT NULL,
  used_at    TIMESTAMPTZ             -- NULL = not yet used
);

ALTER TABLE org_invites ENABLE ROW LEVEL SECURITY;

CREATE INDEX IF NOT EXISTS idx_org_invites_token ON org_invites(token);
CREATE INDEX IF NOT EXISTS idx_org_invites_org_id ON org_invites(org_id);
CREATE INDEX IF NOT EXISTS idx_org_invites_expires ON org_invites(expires_at);
```

6. RLS for `org_invites` — copy INSERT/SELECT/DELETE policy pattern from migration 006 invitations block (lines 339–371).

7. RLS for `users` own-row read — new policy needed:
```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read their own row" ON users
    FOR SELECT USING (external_auth_id = (auth.jwt()->>'sub'));

CREATE POLICY "Service role can manage users" ON users
    FOR ALL USING (auth.jwt()->>'role' = 'service_role');
```

---

### `packages/dashboard/src/app/features/fleet-management/` (component + service, CRUD)

**Analog (component):** `packages/dashboard/src/app/pages/onboarding/fleet/onboarding-fleet.component.ts`
**Analog (service):** `packages/dashboard/src/app/core/services/fleet.service.ts`

**Component imports pattern** (onboarding-fleet.component.ts lines 1–9):
```typescript
import { Component, inject, signal } from '@angular/core';
import { Router } from '@angular/router';
import { FormsModule } from '@angular/forms';
import { MatButtonModule } from '@angular/material/button';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatInputModule } from '@angular/material/input';
import { MatIconModule } from '@angular/material/icon';
// ADD for table view:
import { MatTableModule } from '@angular/material/table';
import { MatDialogModule, MatDialog } from '@angular/material/dialog';
import { MatSnackBarModule, MatSnackBar } from '@angular/material/snack-bar';
```

**Standalone component decorator pattern** (onboarding-fleet.component.ts lines 11–16):
```typescript
@Component({
  selector: 'app-fleet-management',
  standalone: true,
  imports: [ FormsModule, MatButtonModule, MatFormFieldModule,
             MatInputModule, MatIconModule, MatTableModule,
             MatDialogModule, MatSnackBarModule ],
  templateUrl: './fleet-management.component.html',
  styleUrl: './fleet-management.component.scss',
})
```

**Signal state pattern** (onboarding-fleet.component.ts lines 181–183):
```typescript
isLoading = this.vehicleService.isLoading;
error = signal<string | null>(null);
vehicles = this.vehicleService.vehicles;
```

**Form submit pattern** (onboarding-fleet.component.ts lines 184–209):
```typescript
async onSubmit(): Promise<void> {
  if (!this.formData.nickname.trim()) return;
  this.error.set(null);
  try {
    const result = await this.vehicleService.createVehicle(this.formData);
    if (result) {
      // success feedback
    } else {
      this.error.set(this.vehicleService.error() ?? 'Failed to create vehicle');
    }
  } catch (err) {
    this.error.set((err as Error).message ?? 'Failed to create vehicle');
  }
}
```

**Material template pattern** — `appearance="outline"` on all form fields, error banner with `mat-icon` + `role="alert"`, `@if` control flow (not `*ngIf`):
```html
@if (error()) {
  <div class="error-banner" role="alert">
    <mat-icon>error_outline</mat-icon>
    <span>{{ error() }}</span>
  </div>
}
<mat-form-field appearance="outline">
  <mat-label>Vehicle Nickname</mat-label>
  <input matInput [(ngModel)]="nickname" name="nickname" required />
</mat-form-field>
```

**Soft-delete pattern** (D-26) — fleet.service.ts uses hard delete. Vehicle soft-delete instead:
```typescript
// In vehicle.service.ts deleteVehicle():
const { error } = await this.supabase.client
  .from('vehicles')
  .update({ status: 'inactive', updated_at: new Date().toISOString() })
  .eq('id', id);
```

---

### `packages/dashboard/src/app/core/services/vehicle.service.ts` (service, CRUD)

**Analog:** `packages/dashboard/src/app/core/services/fleet.service.ts` — exact structural copy.

**Full service structure to copy** (fleet.service.ts lines 1–275):
```typescript
import { Injectable, signal, computed, inject } from "@angular/core";
import { SupabaseService } from "./supabase.service";

@Injectable({ providedIn: "root" })
export class VehicleService {
  private readonly supabase = inject(SupabaseService);

  readonly vehicles = signal<Vehicle[]>([]);
  readonly selectedVehicle = signal<Vehicle | null>(null);
  readonly isLoading = signal(false);
  readonly error = signal<string | null>(null);

  async loadVehicles(fleetId: string): Promise<void> { ... }
  async createVehicle(data: CreateVehicleDto): Promise<Vehicle | null> { ... }
  async updateVehicle(id: string, updates: Partial<Vehicle>): Promise<boolean> { ... }
  async deleteVehicle(id: string): Promise<boolean> { ... }  // soft delete via status:'inactive'
}
```

**CRUD try/catch pattern** — copy from fleet.service.ts lines 42–55 for every method:
```typescript
try {
  const { data, error } = await this.supabase.client
    .from('vehicles')
    .select('*')
    .eq('fleet_id', fleetId)
    .order('nickname');
  if (error) throw error;
  this.vehicles.set(data ?? []);
} catch (err: unknown) {
  this.error.set((err as Error).message ?? 'Failed to load vehicles');
} finally {
  this.isLoading.set(false);
}
```

**Do NOT use `supabase.client.auth.getUser()`** (fleet.service.ts lines 69, 143 — pitfall 5). Use `authService.currentUser()?.id` instead for owner_id fields.

---

## Shared Patterns

### Signal State (Angular)
**Source:** `packages/dashboard/src/app/core/services/fleet.service.ts` lines 13–16
**Apply to:** All new Angular services and components
```typescript
readonly vehicles = signal<Vehicle[]>([]);
readonly isLoading = signal(false);
readonly error = signal<string | null>(null);
```

### Supabase CRUD Try/Catch
**Source:** `packages/dashboard/src/app/core/services/organization.service.ts` lines 14–31
**Apply to:** All new dashboard services (vehicle.service.ts, invite service)
```typescript
this.isLoading.set(true);
this.error.set(null);
try {
  const { data, error } = await this.supabase.client.from('table').select('*');
  if (error) throw error;
  this.signal.set(data ?? []);
} catch (err: unknown) {
  this.error.set((err as Error).message ?? 'Fallback message');
} finally {
  this.isLoading.set(false);
}
```

### NestJS Controller Error Handling
**Source:** `packages/auth-service/src/auth/auth.controller.ts` lines 10–16
**Apply to:** Any new NestJS controller endpoints
```typescript
try {
  return await this.service.method(dto);
} catch (error) {
  throw new UnauthorizedException('Human-readable message');
}
```

### NestJS Service Supabase CRUD
**Source:** `packages/auth-service/src/users/users.service.ts` lines 24–35
**Apply to:** Any new NestJS service methods touching Supabase
```typescript
const { data, error } = await this.supabase
  .from('table')
  .select('*')
  .eq('column', value)
  .single();
if (error || !data) return null;
return this.mapToEntity(data);
```

### Angular Material Form Field
**Source:** `packages/dashboard/src/app/pages/onboarding/fleet/onboarding-fleet.component.ts` lines 40–54
**Apply to:** All new form components (vehicle registration, invite, fleet creation)
```html
<mat-form-field appearance="outline" class="onboarding-field">
  <mat-label>Label Text</mat-label>
  <input matInput type="text" [(ngModel)]="fieldName" name="fieldName"
         required aria-required="true" placeholder="e.g., hint text" />
  <mat-icon matPrefix aria-hidden="true">icon_name</mat-icon>
</mat-form-field>
```

### Angular Route Guard Async Pattern
**Source:** `packages/dashboard/src/app/core/guards/auth.guard.ts` lines 7–20
**Apply to:** Any new guards
```typescript
const authCheck = async (): Promise<boolean | UrlTree> => {
  const auth = inject(AuthService);
  const router = inject(Router);
  while (auth.isLoading()) {
    await new Promise(resolve => setTimeout(resolve, 50));
  }
  if (!auth.isAuthenticated()) {
    return router.createUrlTree(['/login']);
  }
  return true;
};
```

### SQL Migration Structure
**Source:** `packages/supabase/migrations/006_orgs_and_fleets.sql` lines 13, 719
**Apply to:** Migration 009
```sql
BEGIN;
-- numbered sections with comments
-- ALTER TABLE before CREATE TABLE
-- RLS policies using EXISTS() not IN()
-- CREATE INDEX IF NOT EXISTS after table creation
COMMIT;
```

### RLS Policy (EXISTS pattern)
**Source:** `packages/supabase/migrations/006_orgs_and_fleets.sql` lines 246–254
**Apply to:** All new RLS policies in migration 009
```sql
CREATE POLICY "description" ON table_name
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM fleet_members fm
            WHERE fm.user_id = auth.uid()
            AND fm.organization_id = table_name.organization_id
        )
    );
```

---

## No Analog Found

| File | Role | Data Flow | Reason |
|---|---|---|---|
| `packages/dashboard/src/app/features/settings/invite/invite.component.ts` | component | event-driven | No invite/link-generation UI exists anywhere in codebase. Closest structural analog is `onboarding-fleet.component.ts` for the Angular Material form shell, but the token generation + clipboard copy logic (`navigator.clipboard.writeText()`) and Supabase `org_invites` insert are new patterns with no existing analog. |

---

## Anti-Patterns Flagged (Do NOT copy these existing patterns)

| Existing Code | Location | Why Not to Copy |
|---|---|---|
| `computed(() => firstValueFrom(observable$))` | auth.service.ts line 47 | Returns Promise not boolean — always truthy. Use `toSignal()` instead. |
| `fetch('http://localhost:3001/auth/exchange', ...)` | auth.service.ts line 89 | Hardcoded localhost. Use `environment.authServiceUrl`. |
| `supabase.client.auth.getUser()` for owner_id | organization.service.ts line 66, fleet.service.ts line 69 | Returns null for Auth0 users not in auth.users. Use `authService.currentUser()`. |
| `roles: ['owner']` hardcoded | users.service.ts line 106 | Must read from `dbRow.roles` column. |
| `invitations` table insert | organization.service.ts line 149 | Phase 2 invite flow uses `org_invites` (token-based), not `invitations` (email-based). |
| `allowlistGuard` in canActivate | app.routes.ts lines 81, 90, 99, 108, 113 | D-13: remove entirely. Replace with `onboardingGuard`. |
| `orgId: 'default', roles: ['owner']` in JWT payload | auth.service.ts lines 28–29 | D-06: JWT must contain only sub + email. |

---

## Metadata

**Analog search scope:** `packages/auth-service/src/`, `packages/dashboard/src/app/core/`, `packages/dashboard/src/app/pages/onboarding/`, `packages/supabase/migrations/`
**Files read:** 14
**Pattern extraction date:** 2026-05-12
