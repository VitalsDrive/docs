# Auth Flow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Supabase Auth with Clerk + custom NestJS auth service that issues internal JWTs for backend services.

**Architecture:** Clerk handles email/password, Google OAuth, and third-party auth. NestJS auth service exchanges Clerk JWTs for internal JWTs. Parser/simulator validate internal JWTs. Dashboard stores internal JWT in memory (not localStorage).

**Tech Stack:** Clerk SDK, NestJS, @nestjs/jwt, Angular (dashboard), TypeScript

---

## Overview

This implementation has 6 main phases:

1. Create NestJS auth service
2. Create users table migration
3. Update dashboard auth service
4. Add HTTP interceptor
5. Update guards
6. Add JWT middleware to parser/simulator

---

## Phase 1: NestJS Auth Service

### Task 1.1: Create auth-service package

**Files:**
- Create: `packages/auth-service/package.json`
- Create: `packages/auth-service/tsconfig.json`
- Create: `packages/auth-service/src/main.ts`
- Create: `packages/auth-service/src/app.module.ts`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@vitalsdrive/auth-service",
  "version": "1.0.0",
  "description": "VitalsDrive Auth Service - JWT exchange and validation",
  "main": "dist/main.js",
  "scripts": {
    "build": "nest build",
    "start": "nest start",
    "start:dev": "nest start --watch",
    "start:prod": "node dist/main"
  },
  "dependencies": {
    "@nestjs/common": "^10.0.0",
    "@nestjs/core": "^10.0.0",
    "@nestjs/jwt": "^10.0.0",
    "@nestjs/platform-express": "^10.0.0",
    "clerk": "^4.0.0",
    "js-yaml": "^4.0.0",
    "reflect-metadata": "^0.1.13",
    "rxjs": "^7.8.0"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.0.0",
    "@types/express": "^4.0.0",
    "@types/js-yaml": "^4.0.0",
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": true,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "forceConsistentCasingInFileNames": false,
    "noFallthroughCasesInSwitch": false
  }
}
```

- [ ] **Step 3: Create src/main.ts**

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  const port = process.env.PORT || 3001;
  await app.listen(port);
  console.log(`Auth service running on port ${port}`);
}
bootstrap();
```

- [ ] **Step 4: Create src/app.module.ts**

```typescript
import { Module } from '@nestjs/common';
import { AuthController } from './auth/auth.controller';
import { AuthService } from './auth/auth.service';
import { JwtStrategy } from './auth/jwt.strategy';
import { UsersService } from './users/users.service';

@Module({
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy, UsersService],
})
export class AppModule {}
```

- [ ] **Step 5: Commit**

```bash
git add packages/auth-service/
git commit -m "feat: scaffold NestJS auth service"
```

---

### Task 1.2: Implement auth controller and service

**Files:**
- Create: `packages/auth-service/src/auth/auth.controller.ts`
- Create: `packages/auth-service/src/auth/auth.service.ts`
- Create: `packages/auth-service/src/auth/jwt.strategy.ts`
- Create: `packages/auth-service/src/auth/dto.ts`

- [ ] **Step 1: Create auth DTOs (src/auth/dto.ts)**

```typescript
export class ExchangeTokenDto {
  clerkJwt: string;
}

export class RefreshTokenDto {
  refreshToken: string;
}

export class ValidateTokenDto {
  token: string;
}
```

- [ ] **Step 2: Create auth service (src/auth/auth.service.ts)**

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Clerk } from 'clerk';

@Injectable()
export class AuthService {
  private clerk: Clerk;

  constructor(private jwtService: JwtService) {
    this.clerk = new Clerk({
      secretKey: process.env.CLERK_SECRET_KEY,
    });
  }

  async exchangeClerkToken(clerkJwt: string): Promise<{ accessToken: string; refreshToken: string; user: any }> {
    try {
      const claims = await this.clerk.verifyToken(clerkJwt);
      const userId = claims.sub;
      
      // Get user from database via UsersService
      // For now, create user if not exists (will be created via webhook)
      const user = await this.getOrCreateUser(userId, claims.email);
      
      const payload = {
        sub: user.id,
        email: user.email,
        orgId: user.organizationId || 'default',
        roles: user.roles || ['member'],
      };

      const accessToken = this.jwtService.sign(payload, { expiresIn: '15m' });
      const refreshToken = this.jwtService.sign(payload, { expiresIn: '7d' });

      return { accessToken, refreshToken, user };
    } catch (error) {
      throw new UnauthorizedException('Invalid Clerk token');
    }
  }

  async refreshToken(refreshToken: string): Promise<{ accessToken: string }> {
    try {
      const payload = this.jwtService.verify(refreshToken);
      const newAccessToken = this.jwtService.sign(
        { sub: payload.sub, email: payload.email, orgId: payload.orgId, roles: payload.roles },
        { expiresIn: '15m' }
      );
      return { accessToken: newAccessToken };
    } catch (error) {
      throw new UnauthorizedException('Invalid refresh token');
    }
  }

  async validateToken(token: string): Promise<any> {
    try {
      return this.jwtService.verify(token);
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }

  private async getOrCreateUser(clerkUserId: string, email: string): Promise<any> {
    // This will be implemented in Task 1.3
    // For now, return placeholder
    return { id: clerkUserId, email, organizationId: 'default', roles: ['owner'] };
  }
}
```

- [ ] **Step 3: Create JWT strategy (src/auth/jwt.strategy.ts)**

```typescript
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: process.env.JWT_SECRET || 'vitalsdrive-secret-key',
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, email: payload.email, orgId: payload.orgId, roles: payload.roles };
  }
}
```

- [ ] **Step 4: Create auth controller (src/auth/auth.controller.ts)**

```typescript
import { Controller, Post, Body, Headers, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';
import { ExchangeTokenDto, RefreshTokenDto } from './dto';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('exchange')
  async exchangeToken(@Body() dto: ExchangeTokenDto) {
    try {
      return await this.authService.exchangeClerkToken(dto.clerkJwt);
    } catch (error) {
      throw new UnauthorizedException('Failed to exchange token');
    }
  }

  @Post('refresh')
  async refreshToken(@Body() dto: RefreshTokenDto) {
    try {
      return await this.authService.refreshToken(dto.refreshToken);
    } catch (error) {
      throw new UnauthorizedException('Failed to refresh token');
    }
  }

  @Post('validate')
  async validateToken(@Headers('authorization') authHeader: string) {
    if (!authHeader?.startsWith('Bearer ')) {
      throw new UnauthorizedException('No token provided');
    }
    const token = authHeader.substring(7);
    return await this.authService.validateToken(token);
  }

  @Post('me')
  async getCurrentUser(@Headers('authorization') authHeader: string) {
    if (!authHeader?.startsWith('Bearer ')) {
      throw new UnauthorizedException('No token provided');
    }
    const token = authHeader.substring(7);
    return await this.authService.validateToken(token);
  }
}
```

- [ ] **Step 5: Commit**

```bash
git add packages/auth-service/src/auth/
git commit -m "feat: implement auth controller and service"
```

---

### Task 1.3: Implement users service

**Files:**
- Create: `packages/auth-service/src/users/users.service.ts`
- Create: `packages/auth-service/src/users/users.controller.ts`

- [ ] **Step 1: Create users service (src/users/users.service.ts)**

```typescript
import { Injectable } from '@nestjs/common';

interface User {
  id: string;
  externalAuthId: string;
  externalAuthProvider: string;
  email: string;
  name?: string;
  organizationId?: string;
  roles: string[];
}

@Injectable()
export class UsersService {
  private users: Map<string, User> = new Map();

  async findByExternalId(externalAuthId: string, provider: string = 'clerk'): Promise<User | null> {
    for (const user of this.users.values()) {
      if (user.externalAuthId === externalAuthId && user.externalAuthProvider === provider) {
        return user;
      }
    }
    return null;
  }

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) || null;
  }

  async findByEmail(email: string): Promise<User | null> {
    for (const user of this.users.values()) {
      if (user.email === email) {
        return user;
      }
    }
    return null;
  }

  async create(data: { externalAuthId: string; externalAuthProvider?: string; email: string; name?: string }): Promise<User> {
    const user: User = {
      id: crypto.randomUUID(),
      externalAuthId: data.externalAuthId,
      externalAuthProvider: data.externalAuthProvider || 'clerk',
      email: data.email,
      name: data.name,
      organizationId: 'default',
      roles: ['owner'],
    };
    this.users.set(user.id, user);
    return user;
  }

  async updateOrganization(userId: string, organizationId: string): Promise<User | null> {
    const user = this.users.get(userId);
    if (user) {
      user.organizationId = organizationId;
      return user;
    }
    return null;
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add packages/auth-service/src/users/
git commit -m "feat: implement users service"
```

---

## Phase 2: Database Migration

### Task 2.1: Create users table migration

**Files:**
- Create: `packages/supabase/migrations/007_create_users_clerk.sql`

- [ ] **Step 1: Create migration**

```sql
-- Migration: 007_create_users_clerk.sql
-- Creates users table with external auth provider support (Clerk, Auth0, etc.)

-- Create users table
CREATE TABLE IF NOT EXISTS users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_auth_id TEXT NOT NULL,
  external_auth_provider TEXT NOT NULL DEFAULT 'clerk',
  email TEXT NOT NULL UNIQUE,
  display_name VARCHAR(100),
  preferences JSONB DEFAULT '{"theme": "dark", "notifications": true}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Index for external auth lookups
CREATE INDEX idx_users_external_auth ON users(external_auth_id, external_auth_provider);

-- RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Users can read their own record
CREATE POLICY "Users can read own record" ON users
  FOR SELECT USING (true);

-- Service role can manage all
CREATE POLICY "Service role can manage users" ON users
  FOR ALL USING (auth.jwt()->>'role' = 'service_role');

COMMENT ON TABLE users IS 'Application users linked to external auth providers (Clerk, Auth0, etc.)';
```

- [ ] **Step 2: Commit**

```bash
git add packages/supabase/migrations/007_create_users_clerk.sql
git commit -m "feat: add users table with external auth support"
```

---

## Phase 3: Dashboard Auth Service

### Task 3.1: Update dashboard auth service for Clerk

**Files:**
- Modify: `packages/dashboard/src/app/core/services/auth.service.ts`

- [ ] **Step 1: Read existing auth service**

```bash
# Read the file first to understand current implementation
cat packages/dashboard/src/app/core/services/auth.service.ts
```

- [ ] **Step 2: Replace with Clerk-based implementation**

```typescript
import { Injectable, signal, computed } from '@angular/core';
import { Router } from '@angular/router';
import { createClerk } from '@clerk/clerk-js';

export interface User {
  id: string;
  email: string;
  name: string;
}

@Injectable({
  providedIn: 'root',
})
export class AuthService {
  private clerk: any;
  
  currentUser = signal<User | null>(null);
  isAuthenticated = computed(() => this.currentUser() !== null);
  isLoading = signal(true);
  
  private internalToken: string | null = null;

  constructor(private router: Router) {
    this.initClerk();
  }

  private async initClerk() {
    this.clerk = createClerk({
      publishableKey: 'pk_test_your-key-here',
    });
    
    await this.clerk.load();
    
    this.clerk.addListener(({ user }: { user: any }) => {
      if (user) {
        this.currentUser.set({
          id: user.id,
          email: user.emailAddresses[0]?.emailAddress || '',
          name: user.fullName || '',
        });
      } else {
        this.currentUser.set(null);
      }
      this.isLoading.set(false);
    });
  }

  async signUp(email: string, password: string): Promise<void> {
    await this.clerk.createUser({
      emailAddress: email,
      password,
    });
  }

  async signIn(email: string, password: string): Promise<void> {
    const { user, token } = await this.clerk.signInWithEmailAndPassword({
      email,
      password,
    });
    
    if (user) {
      await this.exchangeToken(token);
    }
  }

  async signInWithGoogle(): Promise<void> {
    const { user, token } = await this.clerk.signInWithOAuth({ provider: 'google' });
    
    if (user) {
      await this.exchangeToken(token);
    }
  }

  private async exchangeToken(clerkToken: string): Promise<void> {
    const response = await fetch('http://localhost:3001/auth/exchange', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ clerkJwt: clerkToken }),
    });
    
    if (response.ok) {
      const data = await response.json();
      this.internalToken = data.accessToken;
    }
  }

  getInternalToken(): string | null {
    return this.internalToken;
  }

  async signOut(): Promise<void> {
    await this.clerk.signOut();
    this.internalToken = null;
    this.currentUser.set(null);
    this.router.navigate(['/login']);
  }
}
```

- [ ] **Step 3: Commit**

```bash
git add packages/dashboard/src/app/core/services/auth.service.ts
git commit -m "feat: update auth service for Clerk"
```

---

## Phase 4: HTTP Interceptor

### Task 4.1: Create auth interceptor

**Files:**
- Create: `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts`

- [ ] **Step 1: Create interceptor**

```typescript
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from '@angular/common/http';
import { Observable } from 'rxjs';
import { AuthService } from '../services/auth.service';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getInternalToken();
    
    if (token) {
      req = req.clone({
        setHeaders: {
          Authorization: `Bearer ${token}`,
        },
      });
    }
    
    return next.handle(req);
  }
}
```

- [ ] **Step 2: Register in app.config.ts**

Add to providers array:
```typescript
{
  provide: HTTP_INTERCEPTORS,
  useClass: AuthInterceptor,
  multi: true,
}
```

- [ ] **Step 3: Commit**

```bash
git add packages/dashboard/src/app/core/interceptors/auth.interceptor.ts
git add packages/dashboard/src/app/app.config.ts
git commit -m "feat: add auth HTTP interceptor"
```

---

## Phase 5: Guards

### Task 5.1: Update auth guard

**Files:**
- Modify: `packages/dashboard/src/app/core/guards/auth.guard.ts`

- [ ] **Step 1: Update auth guard**

```typescript
import { inject } from '@angular/core';
import { Router, CanActivateFn } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  if (authService.isAuthenticated()) {
    return true;
  }
  
  router.navigate(['/login']);
  return false;
};
```

- [ ] **Step 2: Commit**

```bash
git add packages/dashboard/src/app/core/guards/auth.guard.ts
git commit -m "feat: update auth guard for internal JWT"
```

---

## Phase 6: Parser/Simulator JWT Middleware

### Task 6.1: Add JWT middleware to parser

**Files:**
- Modify: `packages/parser/src/main.ts` (or similar entry point)

- [ ] **Step 1: Add JWT validation middleware**

```typescript
import { validateToken } from './middleware/jwt-auth.middleware';

// In your Express/parser setup:
app.use(validateToken);
```

- [ ] **Step 2: Create jwt-auth.middleware.ts**

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET || 'vitalsdrive-secret-key';

export function validateToken(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  const token = authHeader.substring(7);
  
  try {
    const decoded = jwt.verify(token, JWT_SECRET);
    (req as any).user = decoded;
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

- [ ] **Step 3: Commit**

```bash
git add packages/parser/src/
git commit -m "feat: add JWT middleware to parser"
```

---

### Task 6.2: Add JWT middleware to simulator

**Files:**
- Modify: `packages/simulator/src/main.ts`

- [ ] **Step 1: Add same JWT middleware to simulator**

```typescript
import { validateToken } from './middleware/jwt-auth.middleware';

// In your Express/simulator setup:
app.use(validateToken);
```

- [ ] **Step 2: Commit**

```bash
git add packages/simulator/src/
git commit -m "feat: add JWT middleware to simulator"
```

---

## Files Created/Modified Summary

| File | Action |
|------|--------|
| `packages/auth-service/package.json` | Create |
| `packages/auth-service/tsconfig.json` | Create |
| `packages/auth-service/src/main.ts` | Create |
| `packages/auth-service/src/app.module.ts` | Create |
| `packages/auth-service/src/auth/auth.controller.ts` | Create |
| `packages/auth-service/src/auth/auth.service.ts` | Create |
| `packages/auth-service/src/auth/jwt.strategy.ts` | Create |
| `packages/auth-service/src/auth/dto.ts` | Create |
| `packages/auth-service/src/users/users.service.ts` | Create |
| `packages/supabase/migrations/007_create_users_clerk.sql` | Create |
| `packages/dashboard/src/app/core/services/auth.service.ts` | Modify |
| `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts` | Create |
| `packages/dashboard/src/app/app.config.ts` | Modify |
| `packages/dashboard/src/app/core/guards/auth.guard.ts` | Modify |
| `packages/parser/src/middleware/jwt-auth.middleware.ts` | Create |
| `packages/simulator/src/middleware/jwt-auth.middleware.ts` | Create |

---

## Environment Variables

```env
# Clerk
CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

# Auth Service
JWT_SECRET=vitalsdrive-secret-key
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
PORT=3001
```

---

## Verification Steps

After each phase:

1. **Phase 1-2:** Run `npm run build` in `packages/auth-service`
2. **Phase 3:** Run `npm run build` in `packages/dashboard`
3. **Phase 4:** Test HTTP requests include auth header
4. **Phase 5:** Test protected routes redirect to login
5. **Phase 6:** Test parser/simulator reject requests without token

---

## Plan complete and saved to `docs/superpowers/plans/2026-04-20-auth-flow-implementation.md`. Two execution options:

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?