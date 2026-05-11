# Technology Stack

**Analysis Date:** 2026-05-07

## Parser (`packages/parser/`)

- **Runtime:** Node.js (ES2022)
- **Language:** TypeScript 5.3
- **TypeScript config:** `packages/parser/tsconfig.json` — `commonjs` modules, `strict: true`, `declaration: true`, `sourceMap: true`
- **Build tool:** `tsc` (output to `dist/`)
- **Dev server:** `ts-node-dev` 2.0 (hot reload with `--transpile-only`)
- **Key dependencies:**
  - `@supabase/supabase-js` ^2.39.0 — Supabase client for DB writes + device lookups
  - `dotenv` ^16.3.1 — environment loading
- **Entry point:** `packages/parser/src/index.ts` — single `OBD2Parser` class using Node.js `net` server
- **Protocol:** Teltonika Codec 8 Extended binary TCP protocol (0x8E codec ID, CRC-16)
- **Listening port:** 5050 (configurable via `PARSER_PORT`)
- **Tests:** Jest (configured but no test files detected)
- **Linting:** ESLint (`eslint src/**/*.ts`)

## Dashboard (`packages/dashboard/`)

- **Framework:** Angular 21.0
- **Language:** TypeScript 5.9
- **UI framework:** Angular Material 21.0
- **Build:** Angular CLI (`ng serve` / `ng build`)
- **Testing:** Karma + Jasmine (`ng test`)
- **Linting:** ESLint via `ng lint`
- **State management:** Angular signals (`signal`, `computed`) — no NgRx
- **Routing:** Angular Router with lazy-loaded components (`loadComponent`)
- **Key dependencies:**
  - `@auth0/auth0-angular` ^2.8.1 — Auth0 OIDC integration (`domain: ronbiter.auth0.com`)
  - `@clerk/clerk-js` ^6.0.0 — installed but not currently used in app config
  - `@supabase/supabase-js` ^2.39.0 — database client + Realtime subscriptions
  - `leaflet` ^1.9.4 — map rendering
  - `rxjs` ^7.8.1 — reactive streams
  - `zone.js` ~0.15.0 — Angular change detection
- **Entry point:** `packages/dashboard/src/app/app.ts` — `AppComponent` with inline template `<router-outlet />`, `ChangeDetectionStrategy.OnPush`
- **Routes:** `packages/dashboard/src/app/app.routes.ts` — public routes (`/login`, `/signup`, `/pending`), protected routes (`/dashboard`, `/map`, `/vehicle/:id`, `/alerts`, `/backoffice/**`), onboarding flow (`/onboarding/**`)
- **Guards:** `authGuard`, `authGuardChild`, `allowlistGuard`, `onboardingGuard`, `onboardingStepGuard`, `adminGuard`, `fleetAdminGuard`, `orgAdminGuard`, `orgOwnerGuard`
- **Architecture:** Standalone components only, no NgModules. Services use `providedIn: "root"`
- **Environment config:**
  - Local: `packages/dashboard/src/environments/environment.ts` — `http://localhost:54321`
  - Dev: `packages/dashboard/src/environments/environment.development.ts` — production Supabase (`odwctmlawibhaclptsew.supabase.co`)

## Simulator (`packages/simulator/`)

- **Runtime:** Node.js (ES2022)
- **Language:** TypeScript 5.3
- **TypeScript config:** `packages/simulator/tsconfig.json` — `commonjs` modules, `strict: true`
- **Build tool:** `tsc` (output to `dist/`)
- **Dev server:** `ts-node-dev` 2.0
- **Key dependencies:**
  - `dotenv` ^16.3.1 — environment loading
  - `@types/readable-stream` ^2.3.15 — type declarations
  - No external SDKs — uses Node.js `net.Socket` directly
- **Entry point:** `packages/simulator/src/index.ts` — `GhostFleetSimulator` class
- **How it works:** Connects to parser via TCP on port 5050, sends IMEI login, then sends binary Teltonika Codec 8 Extended AVL record packets with configurable GPS coordinates. Supports SMS login/password authentication. Interval: configurable (default 5000ms).
- **Tests:** Jest (configured but no test files detected)

## Supabase (`packages/supabase/`)

- **Database:** PostgreSQL (via Supabase)
- **Management tool:** Supabase CLI ^1.226.4
- **Extensions:** `uuid-ossp` (for UUID generation)
- **Migration count:** 8 migrations (`001_initial_schema.sql` through `008_add_org_and_roles_to_users.sql`)
- **Tables defined:**
  - `telemetry_logs` — BIGSERIAL id, vehicle_id FK, lat/lng, temp, voltage, rpm, dtc_codes (TEXT[]), timestamp
  - `vehicles` — UUID id, fleet_id FK, vin (unique), make, model, year, license_plate, status, metadata
  - `fleets` — UUID id, organization_id FK, name, settings (JSONB), timestamps
  - `organizations` — UUID id, name, plan (starter/professional/enterprise), can_add_devices, max_devices, owner_id
  - `users` — UUID id, external_auth_id, external_auth_provider, email, display_name, preferences (JSONB), organization_id, roles (TEXT[])
  - `fleet_members` — composite PK (fleet_id, user_id), role, organization_id
  - `devices` — IMEI, imei (unique), fleet_id, vehicle_id, status (unassigned/active/inactive), sms_login, sms_password, last_seen
  - `device_assignments` — history table for device-vehicle assignments
  - `alerts` — BIGSERIAL id, vehicle_id, fleet_id, severity, code, message, dtc_codes, lat/lng, acknowledged fields
  - `telemetry_rules` — fleet-level alert threshold rules
  - `scheduled_maintenance` — vehicle maintenance tracking
  - `invitations` — org invite system
- **RLS:** Row Level Security enabled on all tables with fleet-scoped, role-aware policies
- **Functions:** `assign_device_to_vehicle`, `unassign_device`, `create_organization_and_first_fleet`, `check_user_login_status`, `is_user_org_allowlisted`, `get_user_org_id`, `get_user_org_role`, `org_can_add_device`, `create_invitation`, `accept_invitation`, `update_organization`, `get_user_primary_org`
- **Local development:** Docker via `supabase start` (port 54321)

## Auth Service (`packages/auth-service/`)

- **Framework:** NestJS 10.0
- **Language:** TypeScript 5.0
- **TypeScript config:** `packages/auth-service/tsconfig.json` — `commonjs`, `emitDecoratorMetadata`, `experimentalDecorators`, `ES2021` target, `strictNullChecks: false`
- **Build:** NestJS CLI (`nest build`)
- **Key dependencies:**
  - `@nestjs/common` / `@nestjs/core` / `@nestjs/platform-express` ^10.0.0
  - `@nestjs/jwt` ^10.0.0 — JWT signing/verification
  - `@nestjs/passport` ^10.0.0 — Passport integration
  - `passport` ^0.7.0, `passport-jwt` ^4.0.1 — JWT auth strategy
  - `jsonwebtoken` ^9.0.2 — token decode/verify
  - `jwks-rsa` ^4.0.1 — Auth0 JWKS endpoint for public key retrieval
  - `@supabase/supabase-js` ^2.104.1 — Supabase client for user lookups
  - `axios` ^1.6.0 — HTTP client (dependency but not currently imported in source)
  - `rxjs` ^7.8.0
- **Entry point:** `packages/auth-service/src/main.ts` — listens on port 3001 (configurable via `PORT`)
- **Auth strategy:** Auth0 token exchange — receives Auth0 access token, validates via JWKS (RS256), exchanges for internal JWT (15min access, 7-day refresh)
- **Endpoints:**
  - `POST /auth/exchange` — exchange Auth0 token for internal JWT
  - `POST /auth/refresh` — refresh internal access token
  - `POST /auth/validate` — validate internal JWT
  - `POST /auth/me` — get current user from token
- **JWT strategy:** Bearer token from `Authorization` header, extracts userId/email/orgId/roles from payload
- **Users service:** `packages/auth-service/src/users/users.service.ts` — queries `users` table in Supabase, supports Auth0 external auth

## Shared / Infra

- **Package manager:** npm with workspaces (`packages/*`)
- **Root config:** `package.json` — convenience scripts for dev/build across packages
- **CI/CD:** Not detected — no `.github/` workflows found
- **Deployment:**
  - Parser: Railway (port 5050)
  - Dashboard: Vercel
  - Supabase: Managed Supabase cloud (project: `odwctmlawibhaclptsew`) + local Docker for dev
  - Auth Service: Not yet deployed (localhost:3001 only, no deployment config detected)
- **Git:** 5 separate repositories (one per package + docs) under VitalsDrive organization

---

*Stack analysis: 2026-05-07*
