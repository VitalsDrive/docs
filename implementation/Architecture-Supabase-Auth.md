# VitalsDrive — Architecture: Supabase Auth
**Version:** 1.0  
**Status:** Draft  
**Last Updated:** April 2026  
**Document Owner:** VitalsDrive Engineering  

---

## 1. Executive Summary

### 1.1 Purpose

This document defines the new authentication architecture for VitalsDrive, replacing Auth0 with Supabase Auth. The migration simplifies the codebase by removing the auth-service package entirely and leveraging Supabase's built-in authentication and Row Level Security (RLS) for all access control.

### 1.2 Architecture Changes

| Component | Before (Auth0) | After (Supabase Auth) |
|---|---|---|
| **Authentication** | Auth0 (external) | Supabase Auth (built-in) |
| **Token Exchange** | auth-service (NestJS) | Removed |
| **Internal JWT** | Custom JWT from auth-service | Supabase session token |
| **Access Control** | Custom middleware | Supabase RLS policies |
| **Dashboard Auth** | @auth0/auth0-angular | Supabase Auth directly |
| **Real-time** | Supabase (anon key) | Supabase (session token) |

---

## 2. New Auth Flow

### 2.1 Authentication Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    VitalsDrive Supabase Auth Flow                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐                                                       │
│   │  Dashboard  │                                                       │
│   │   Browser   │                                                       │
│   └─────┬──────┘                                                       │
│         │                                                                │
│         │ 1. signIn(email/password or OAuth)                                  │
│         │                                                                │
│         ▼                                                                │
│   ┌──────────────────────────────────────────┐                          │
│   │      Supabase Auth (auth.v1/...)        │                          │
│   │                                          │                          │
│   │   - Email/Password registration           │                          │
│   │   - OAuth (Google, GitHub)              │                          │
│   │   -Magic link (optional)             │                          │
│   │                                          │                          │
│   │   Returns: session.access_token         │                          │
│   └──────────────────┬───────────────────┘                          │
│                      │                                                │
│                      │ 2. session.access_token                         │
│                      │                                                │
│                      ▼                                                │
│   ┌──────────────────────────────────────────┐                          │
│   │      Supabase Client (authenticated)     │                          │
│   │                                          │                          │
│   │   • Uses session token for all requests  │                          │
│   │   • RLS evaluates auth.uid()            │                          │
│   │   • Realtime subscribes with token        │                          │
│   │                                          │                          │
│   └──────────────────┬───────────────────┘                          │
│                      │                                                │
│                      │ 3. REST API calls                             │
│                      │    (SELECT, INSERT, UPDATE)                   │
│                      ▼                                                │
│   ┌──────────────────────────────────────────┐                          │
│   │    Supabase PostgreSQL + RLS              │                          │
│   │                                          │                          │
│   │   RLS Policies:                         │                          │
│   │   ─────────────                         │                          │
│   │   ✓ users: auth.uid = user_id          │                          │
│   │   ✓ vehicles: user is fleet_member   │                          │
│   │   ✓ telemetry_logs: user is fleet_member │                        │
│   │   ✓ alerts: user is fleet_member     │                          │
│   │   ✓ fleets: owner_id = auth.uid     │                          │
│   │                                          │                          │
│   └──────────────────────────────────────────┘                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Token Management

```typescript
// Supabase Auth Session
interface Session {
  access_token: string;    // JWT with auth claims
  refresh_token: string;   // For session refresh
  expires_in: number;    // Seconds until expiry
  expires_at: number;   // Unix timestamp
  token_type: 'bearer';
  user: User;
}

// Supabase User (from auth.users)
interface User {
  id: string;                    // UUID (auth.uid)
  email: string;
  email_confirmed_at: string;
  created_at: string;
  last_sign_in_at: string;
  app_metadata: {
    provider: string;            // 'email', 'google', 'github'
    [key: string]: any;
  };
  user_metadata: {
    [key: string]: any;       // Custom user fields
  };
}
```

---

## 3. Package Changes

### 3.1 Removed: auth-service Package

The entire `packages/auth-service` package is removed:

| File/Path | Action |
|---|---|
| `packages/auth-service/` | Delete |
| `packages/auth-service/src/auth/` | Delete |
| `packages/auth-service/src/users/` | Delete |
| `packages/auth-service/package.json` | Delete |
| `packages/auth-service/tsconfig.json` | Delete |
| `packages/auth-service/Dockerfile` | Delete |

### 3.2 Package Dependencies

**Before:**
```json
{
  "dependencies": {
    "@auth0/auth0-angular": "^2.8.1",
    "@auth0/auth0-spa-js": "^2.19.2"
  }
}
```

**After:**
```json
{
  "dependencies": {
    "@supabase/supabase-js": "^2.47.0"
  }
}
```

---

## 4. Database Schema Updates

### 4.1 Users Table Integration

The `users` table in Supabase aligns with auth.users:

```sql
-- Public profile table (mirrors auth.users)
CREATE TABLE public.profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
 created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- RLS policies
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile"
  ON public.profiles FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "Users can update own profile"
  ON public.profiles FOR UPDATE
  USING (auth.uid() = id);
```

### 4.2 Fleet Membership Table

```sql
-- Fleet membership links users to fleets
CREATE TABLE public.fleet_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  fleet_id UUID NOT NULL REFERENCES public.fleets(id) ON DELETE CASCADE,
  role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, fleet_id)
);

-- RLS: members can view their fleets
CREATE POLICY "Fleet members can view fleets"
  ON public.fleet_members FOR SELECT
  USING (
    user_id = auth.uid() OR
    EXISTS (
      SELECT 1 FROM public.fleet_members fm
      WHERE fm.fleet_id = fleet_members.fleet_id
      AND fm.user_id = auth.uid()
    )
  );
```

### 4.3 RLS Policy Reference

| Table | Policy | Expression |
|---|---|---|
| `profiles` | SELECT (own) | `auth.uid() = id` |
| `profiles` | UPDATE (own) | `auth.uid() = id` |
| `fleets` | SELECT (member) | `auth.uid() IN (SELECT user_id FROM fleet_members WHERE fleet_id = fleets.id)` |
| `fleets` | INSERT (new) | `auth.uid() = owner_id` |
| `fleets` | UPDATE (owner) | `auth.uid() = owner_id` |
| `vehicles` | SELECT (member) | `auth.uid() IN (SELECT user_id FROM fleet_members fm JOIN fleets f ON f.id = fm.fleet_id WHERE f.id = vehicles.fleet_id)` |
| `vehicles` | INSERT (fleet owner) | `auth.uid() IN (SELECT owner_id FROM fleets WHERE id = vehicles.fleet_id)` |
| `telemetry_logs` | SELECT (member) | `auth.uid() IN (SELECT user_id FROM fleet_members fm JOIN fleets f ON f.id = fm.fleet_id JOIN vehicles v ON v.fleet_id = f.id WHERE v.id = telemetry_logs.vehicle_id)` |
| `telemetry_logs` | INSERT (device) | `device_imei IN (SELECT imei FROM devices WHERE fleet_id IN (SELECT fleet_id FROM vehicles WHERE id = telemetry_logs.vehicle_id))` |
| `alerts` | SELECT (member) | `auth.uid() IN (SELECT user_id FROM fleet_members fm JOIN fleets f ON f.id = fm.fleet_id JOIN vehicles v ON v.fleet_id = f.id WHERE v.id = alerts.vehicle_id)` |

---

## 5. Supabase Client Configuration

### 5.1 Dual Client Pattern

```typescript
// core/supabase.ts
import { createClient, SupabaseClient } from '@supabase/supabase-js';

export interface Environment {
  supabase: {
    url: string;
    anonKey: string;
  };
}

let supabaseAnon: SupabaseClient | null = null;
let supabaseAuth: SupabaseClient | null = null;

export function getSupabaseAnon(): SupabaseClient {
  if (!supabaseAnon) {
    supabaseAnon = createClient(
      environment.supabase.url,
      environment.supabase.anonKey,
      {
        realtime: { params: { apikey: environment.supabase.anonKey } },
        global: { fetch: fetch }
      }
    );
  }
  return supabaseAnon;
}

export function getSupabaseAuth(): SupabaseClient {
  if (!supabaseAuth) {
    supabaseAuth = createClient(
      environment.supabase.url,
      environment.supabase.anonKey,
      {
        auth: {
          persistSession: true,
          autoRefreshToken: true,
          detectSessionInUrl: true
        },
        realtime: { params: { apikey: environment.supabase.anonKey } },
        global: { fetch: fetch }
      }
    );
  }
  return supabaseAuth;
}

export function setAuthSession(accessToken: string): void {
  supabaseAuth?.auth.setSession(accessToken);
}

export function clearAuthSession(): void {
  supabaseAuth?.auth.signOut();
}
```

### 5.2 Environment Configuration

```typescript
// environment.ts
export const environment = {
  production: false,
  supabase: {
    url: 'https://[project].supabase.co',
    anonKey: '[anon-key]'
  },
  supabaseAuth: {
    redirectUrl: 'http://localhost:4200/auth/callback'
  }
};

// environment.prod.ts
export const environment = {
  production: true,
  supabase: {
    url: 'https://[project].supabase.co',
    anonKey: '[anon-key]'
  },
  supabaseAuth: {
    redirectUrl: 'https://vitalsdrive.com/auth/callback'
  }
};
```

---

## 6. Auth Guards

### 6.1 Auth Guard

```typescript
// core/guards/auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { getSupabaseAuth } from '../supabase';

export const authGuard: CanActivateFn = () => {
  const router = inject(Router);
  const supabase = getSupabaseAuth();
  
  const { data: { session } } = supabase.auth.getSession();
  
  if (session) {
    return true;
  }
  
  router.navigate(['/login']);
  return false;
};
```

### 6.2 Guest Guard (unauthenticated routes)

```typescript
// core/guards/guest.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { getSupabaseAuth } from '../supabase';

export const guestGuard: CanActivateFn = () => {
  const router = inject(Router);
  const supabase = getSupabaseAuth();
  
  const { data: { session } } = supabase.auth.getSession();
  
  if (!session) {
    return true;
  }
  
  router.navigate(['/dashboard']);
  return false;
};
```

### 6.3 Role Guard

```typescript
// core/guards/role.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router, ActivatedRouteSnapshot } from '@angular/router';
import { getSupabaseAuth } from '../supabase';

export const roleGuard: CanActivateFn = (route: ActivatedRouteSnapshot) => {
  const router = inject(Router);
  const supabase = getSupabaseAuth();
  const requiredRole = route.data['role'];
  
  const { data: { session } } = supabase.auth.getSession();
  if (!session) {
    router.navigate(['/login']);
    return false;
  }
  
  // TODO: Check user's role in fleet_members
  // For now, all authenticated users have access
  return true;
};
```

---

## 7. HTTP Interceptor

### 7.1 Auth Interceptor

```typescript
// core/interceptors/auth.interceptor.ts
import { HttpInterceptorFn, HttpRequest, HttpHandlerFn } from '@angular/common/http';
import { getSupabaseAuth } from '../supabase';

export const authInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => {
  // Only add token for API requests to Supabase
  if (!req.url.includes('.supabase.co')) {
    return next(req);
  }
  
  const supabase = getSupabaseAuth();
  const { data: { session } } = supabase.auth.getSession();
  
  if (session?.access_token) {
    const authReq = req.clone({
      setHeaders: {
        Authorization: `Bearer ${session.access_token}`
      }
    });
    return next(authReq);
  }
  
  return next(req);
};
```

### 7.2 Register in app.config.ts

```typescript
// app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './core/interceptors/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor])
    ),
    // ... other providers
  ]
};
```

---

## 8. User Profile Management

### 8.1 Profile Service

```typescript
// core/services/profile.service.ts
import { Injectable, signal, inject } from '@angular/core';
import { getSupabaseAuth } from '../supabase';

export interface Profile {
  id: string;
  email: string;
  full_name: string | null;
  avatar_url: string | null;
  created_at: string;
}

@Injectable({ providedIn: 'root' })
export class ProfileService {
  private supabase = getSupabaseAuth();
  
  readonly profile = signal<Profile | null>(null);
  readonly isLoading = signal(false);
  
  async loadProfile(): Promise<Profile | null> {
    const { data: { session } } = this.supabase.auth.getSession();
    if (!session?.user) return null;
    
    const { data, error } = await this.supabase
      .from('profiles')
      .select('*')
      .eq('id', session.user.id)
      .single();
    
    if (error) {
      console.error('Error loading profile:', error);
      return null;
    }
    
    this.profile.set(data);
    return data;
  }
  
  async updateProfile(updates: Partial<Profile>): Promise<Profile | null> {
    const { data: { session } } = this.supabase.auth.getSession();
    if (!session?.user) return null;
    
    const { data, error } = await this.supabase
      .from('profiles')
      .update(updates)
      .eq('id', session.user.id)
      .select()
      .single();
    
    if (error) {
      console.error('Error updating profile:', error);
      return null;
    }
    
    this.profile.set(data);
    return data;
  }
  
  async signOut(): Promise<void> {
    await this.supabase.auth.signOut();
    this.profile.set(null);
  }
}
```

### 8.2 Profile Router Integration

```typescript
// After successful sign-in, redirect to callback
// auth/callback.component.ts
@Component({
  template: `<p>Signing you in...</p>`
})
export class AuthCallbackComponent {
  private router = inject(Router);
  private profileService = inject(ProfileService);
  
  constructor() {
    this.handleAuthCallback();
  }
  
  private async handleAuthCallback(): Promise<void> {
    const { data: { session } } = await this.supabase.auth.getSession();
    
    if (session) {
      await this.profileService.loadProfile();
      this.router.navigate(['/dashboard']);
    } else {
      this.router.navigate(['/login']);
    }
  }
}
```

---

## 9. Login Page Implementation

### 9.1 Login Component

```typescript
// pages/login/login.component.ts
@Component({
  selector: 'app-login',
  template: `
    <div class="login-container">
      <h1>Sign in to VitalsDrive</h1>
      
      @if (isOAuthEnabled()) {
        <button (click)="signInWithGoogle()">Continue with Google</button>
      }
      
      <div class="divider">or</div>
      
      <form (ngSubmit)="signInWithEmail()">
        <input
          type="email"
          [(ngModel)]="email"
          name="email"
          placeholder="Email"
          required
        />
        <input
          type="password"
          [(ngModel)]="password"
          name="password"
          placeholder="Password"
          required
        />
        <button type="submit" [disabled]="isLoading()">
          {{ isLoading() ? 'Signing in...' : 'Sign in' }}
        </button>
      </form>
      
      @if (error()) {
        <p class="error">{{ error() }}</p>
      }
      
      <p class="signup-link">
        Don't have an account? <a (click)="signUp()">Sign up</a>
      </p>
    </div>
  `
})
export class LoginComponent {
  private supabase = getSupabaseAuth();
  
  email = '';
  password = '';
  isLoading = signal(false);
  error = signal<string | null>(null);
  isOAuthEnabled = signal(true);
  
  async signInWithEmail(): Promise<void> {
    this.isLoading.set(true);
    this.error.set(null);
    
    try {
      const { error } = await this.supabase.auth.signInWithPassword({
        email: this.email,
        password: this.password
      });
      
      if (error) throw error;
      
      // Redirect to dashboard
      window.location.href = '/dashboard';
    } catch (err: any) {
      this.error.set(err.message || 'Sign in failed');
    } finally {
      this.isLoading.set(false);
    }
  }
  
  async signInWithGoogle(): Promise<void> {
    await this.supabase.auth.signInWithOAuth({
      provider: 'google',
      options: {
        redirectTo: window.location.origin + '/auth/callback'
      }
    });
  }
  
  async signUp(): Promise<void> {
    // Navigate to signup page or show signup form
  }
}
```

### 9.2 Sign Up Component

```typescript
// pages/signup/signup.component.ts
@Component({
  selector: 'app-signup',
  template: `
    <div class="signup-container">
      <h1>Create your account</h1>
      
      <form (ngSubmit)="signUp()">
        <input
          type="email"
          [(ngModel)]="email"
          name="email"
          placeholder="Email"
          required
        />
        <input
          type="password"
          [(ngModel)]="password"
          name="password"
          placeholder="Password"
          minlength="8"
          required
        />
        <button type="submit" [disabled]="isLoading()">
          {{ isLoading() ? 'Creating account...' : 'Create account' }}
        </button>
      </form>
    </div>
  `
})
export class SignupComponent {
  private supabase = getSupabaseAuth();
  
  email = '';
  password = '';
  isLoading = signal(false);
  error = signal<string | null>(null);
  
  async signUp(): Promise<void> {
    this.isLoading.set(true);
    this.error.set(null);
    
    try {
      const { error } = await this.supabase.auth.signUp({
        email: this.email,
        password: this.password,
        options: {
          emailRedirectTo: window.location.origin + '/auth/callback'
        }
      });
      
      if (error) throw error;
      
      // Show success message
      alert('Check your email to confirm your account!');
    } catch (err: any) {
      this.error.set(err.message || 'Sign up failed');
    } finally {
      this.isLoading.set(false);
    }
  }
}
```

---

## 10. Route Configuration

### 10.1 Updated Routes

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './core/guards/auth.guard';
import { guestGuard } from './core/guards/guest.guard';

export const routes: Routes = [
  {
    path: '',
    canActivate: [authGuard],
    children: [
      {
        path: 'dashboard',
        loadComponent: () => import('./pages/dashboard/dashboard.component')
          .then(m => m.DashboardComponent)
      },
      {
        path: 'map',
        loadComponent: () => import('./pages/fleet-map/fleet-map.component')
          .then(m => m.FleetMapComponent)
      },
      {
        path: 'vehicle/:id',
        loadComponent: () => import('./pages/vehicle-detail/vehicle-detail.component')
          .then(m => m.VehicleDetailComponent)
      },
      {
        path: 'settings',
        loadComponent: () => import('./pages/settings/settings.component')
          .then(m => m.SettingsComponent)
      }
    ]
  },
  {
    path: 'login',
    canActivate: [guestGuard],
    loadComponent: () => import('./pages/login/login.component')
      .then(m => m.LoginComponent)
  },
  {
    path: 'signup',
    canActivate: [guestGuard],
    loadComponent: () => import('./pages/signup/signup.component')
      .then(m => m.SignupComponent)
  },
  {
    path: 'auth/callback',
    loadComponent: () => import('./pages/auth-callback/auth-callback.component')
      .then(m => m.AuthCallbackComponent)
  },
  {
    path: '**',
    redirectTo: 'dashboard'
  }
];
```

---

## 11. Supabase Auth Configuration

### 11.1 Email Settings (Supabase Dashboard)

| Setting | Value |
|---|---|
| **Enable email signup** | ON |
| **Confirm email** | ON (optional, recommended OFF for faster onboarding) |
| **Min password length** | 8 characters |
| **Enable passwordless** | OFF (MVP) |

### 11.2 OAuth Providers

| Provider | Status | Configuration |
|---|---|---|
| **Google** | Enable | Create project in Google Cloud Console |
| **GitHub** | Disable | Optional for MVP |

### 11.3 Site URL Configuration

In Supabase Dashboard → Authentication → URL Configuration:

| Setting | Value |
|---|---|
| **Site URL** | `https://vitalsdrive.com` |
| **Redirect URLs** | `http://localhost:4200/auth/callback` |

---

## 12. Migration Benefits

### 12.1 Simplified Architecture

| Benefit | Before | After |
|---|---|---|
| **Dependencies** | 2 packages | 1 package |
| **Token exchange** | HTTP round-trip to auth-service | Built-in |
| **RLS policies** | Custom middleware | Native PostgreSQL |
| **Session management** | Internal JWT + refresh | Supabase handles |

### 12.2 Reduced Complexity

- Remove auth-service package entirely
- No token exchange HTTP endpoints
- No JWT validation middleware
- No custom auth guards for auth-service
- Single authentication flow

### 12.3 Better Security

- Supabase handles token refresh automatically
- RLS enforced at database level
- No custom security middleware to maintain
- Supabase security team manages auth infrastructure

---

## Appendix A: Environment Variables

### Dashboard (.env)

```
SUPABASE_URL=https://[project].supabase.co
SUPABASE_ANON_KEY=[anon-key]
SUPABASE_REDIRECT_URL=http://localhost:4200/auth/callback
```

### Supabase (SQL Editor)

```sql
-- Enable RLS on all tables
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.fleets ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.fleet_members ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.vehicles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.telemetry_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.alerts ENABLE ROW LEVEL SECURITY;

-- Create indexes for RLS performance
CREATE INDEX idx_fleet_members_user_id ON public.fleet_members(user_id);
CREATE INDEX idx_fleet_members_fleet_id ON public.fleet_members(fleet_id);
CREATE INDEX idx_vehicles_fleet_id ON public.vehicles(fleet_id);
```

---

## Appendix B: Testing Checklist

- [ ] Login with email/password works
- [ ] Login with OAuth (Google) works
- [ ] Sign up flow works
- [ ] Auth guard protects dashboard routes
- [ ] Guest guard protects login route
- [ ] Profile loads after login
- [ ] Profile updates persist
- [ ] Sign out clears session
- [ ] RLS policies enforce access control
- [ ] Realtime subscriptions authenticated

---

*Document Owner: VitalsDrive Engineering  
Next Review: Post-migration*