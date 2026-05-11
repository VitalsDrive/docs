# Codebase Structure

**Analysis Date:** 2026-05-07

## Monorepo Layout

```
VitalsDrive/
├── packages/
│   ├── parser/             # Node.js + TypeScript TCP ingestion server (:5050)
│   ├── dashboard/          # Angular 21 + Material UI SPA (:4200)
│   ├── simulator/          # Node.js + TypeScript Ghost Fleet simulator
│   ├── supabase/           # PostgreSQL schema & migrations (local Docker)
│   └── auth-service/       # NestJS JWT exchange & validation service (:3001)
├── docs/                   # PRD & architecture specifications
├── package.json            # Root npm workspaces (convenience scripts only)
└── VitalsDrive.code-workspace
```

Each package has its own `package.json`, `tsconfig.json`, `.env.example`. Packages are independent — no shared code between them.

## Directory Purposes

**packages/parser/:**
- Purpose: TCP ingestion server for 4G OBD2 device data
- Contains: Single-file Node.js/TypeScript server (`src/index.ts`)
- Key files: `src/index.ts` (all parser logic)

**packages/dashboard/:**
- Purpose: Real-time fleet health Angular SPA
- Contains: Full Angular application with pages, services, guards, models, components
- Key files: `src/app/app.ts`, `src/app/app.routes.ts`, `src/app/app.config.ts`

**packages/simulator/:**
- Purpose: Emulates OBD2 devices for testing the parser
- Contains: Single-file simulator (`src/index.ts`)
- Key files: `src/index.ts`

**packages/supabase/:**
- Purpose: Database schema, migrations, seed data
- Contains: SQL migrations and seed scripts
- Key files: `migrations/` (numbered SQL files), `seed/001_seed_data.sql`

**packages/auth-service/:**
- Purpose: Auth0 → internal JWT token exchange
- Contains: NestJS application with auth module
- Key files: `src/auth/auth.service.ts`, `src/auth/auth.controller.ts`

## Key File Locations

**Entry Points:**
- `packages/parser/src/index.ts`: `new OBD2Parser().start()` — TCP server on port 5050
- `packages/dashboard/src/main.ts`: Angular bootstrap (`bootstrapApplication(App, appConfig)`)
- `packages/simulator/src/index.ts`: `new GhostFleetSimulator(config).start()`
- `packages/auth-service/src/main.ts`: NestJS bootstrap

**Configuration:**
- `packages/dashboard/src/app/app.config.ts`: Angular application config (Auth0, router, animations, HTTP interceptors)
- `packages/dashboard/src/app/app.routes.ts`: Route definitions (189 lines, lazy-loaded components)
- `packages/dashboard/src/environments/environment.ts`: Dev environment (local Supabase URL)
- `packages/dashboard/src/environments/environment.development.ts`: Dev environment overrides
- `packages/parser/package.json`: `@vitalsdrive/parser` — scripts: dev, build, start, test, lint
- `packages/auth-service/package.json`: `@vitalsdrive/auth-service` — NestJS scripts

**Core Logic:**
- `packages/parser/src/index.ts` — OBD2Parser class (337 lines): TCP server, device auth, binary protocol parsing, Supabase writes
- `packages/simulator/src/index.ts` — GhostFleetSimulator class (226 lines): TCP client, login packet builder, binary telemetry generator
- `packages/auth-service/src/auth/auth.service.ts` — Auth0 JWKS verification, internal JWT issuance
- `packages/dashboard/src/app/core/services/vehicle.service.ts` — Vehicle loading, telemetry subscription, health scoring
- `packages/dashboard/src/app/core/services/telemetry.service.ts` — Supabase Realtime channel management
- `packages/dashboard/src/app/core/services/supabase.service.ts` — Supabase client singleton, auth state management
- `packages/dashboard/src/app/core/services/auth.service.ts` — Auth0 integration, token exchange

**Testing:**
- `packages/parser/`: Jest configured (see `package.json` — `"test": "jest"`)
- `packages/simulator/`: Jest configured
- `packages/dashboard/`: Karma + Jasmine (`"test": "ng test"`)

## Parser Source Structure

```
packages/parser/
├── package.json
├── src/
│   └── index.ts          # OBD2Parser class (only source file)
└── (no separate types/, services/, etc. directories)
```

**Entry point:** `src/index.ts:333` — instantiates and starts `OBD2Parser`
**Exports:** `OBD2Parser`, `TelemetryPacket` interface

## Dashboard Source Structure

```
packages/dashboard/src/
├── main.ts                           # Angular bootstrap
├── app/
│   ├── app.ts                        # Root component (<router-outlet />)
│   ├── app.routes.ts                 # Route definitions
│   ├── app.config.ts                 # ApplicationConfig (providers)
│   ├── app.spec.ts                   # Root component test
│   │
│   ├── core/
│   │   ├── guards/
│   │   │   ├── auth.guard.ts         # authGuard, authGuardChild, allowlistGuard, onboardingGuard, onboardingStepGuard
│   │   │   ├── admin.guard.ts        # Admin role check
│   │   │   ├── org-admin.guard.ts    # Organization admin check
│   │   │   ├── org-owner.guard.ts    # Organization owner check
│   │   │   └── fleet-admin.guard.ts  # Fleet admin check
│   │   │
│   │   ├── models/
│   │   │   ├── vehicle.model.ts      # Vehicle, VehicleWithHealth interfaces
│   │   │   ├── telemetry.model.ts    # TelemetryRecord, ConnectionStatus
│   │   │   ├── alert.model.ts        # Alert model
│   │   │   ├── dtc.model.ts          # DTC (diagnostic trouble code) model
│   │   │   ├── user.model.ts         # User model
│   │   │   ├── device.model.ts       # Device model
│   │   │   ├── fleet.model.ts        # Fleet model
│   │   │   ├── organization.model.ts # Organization model
│   │   │   └── user.model.ts         # User model
│   │   │
│   │   └── services/
│   │       ├── supabase.service.ts   # Supabase client, auth state (signals)
│   │       ├── auth.service.ts       # Auth0 integration, token exchange
│   │       ├── vehicle.service.ts    # Vehicle CRUD, health scoring, telemetry subscription
│   │       ├── telemetry.service.ts  # Supabase Realtime channel management
│   │       ├── fleet.service.ts      # Fleet operations
│   │       ├── device.service.ts     # Device management
│   │       ├── organization.service.ts # Organization operations
│   │       ├── alert.service.ts      # Alert creation, threshold evaluation
│   │       └── dtc-translation.service.ts # DTC code lookup
│   │
│   ├── layout/
│   │   ├── shell/
│   │   │   └── shell.component.ts    # Authenticated layout wrapper
│   │   ├── header/
│   │   │   └── header.component.ts   # Top bar
│   │   └── sidebar/
│   │       └── sidebar.component.ts  # Navigation sidebar
│   │
│   ├── pages/
│   │   ├── login/
│   │   │   └── login.component.ts    # Login page
│   │   ├── signup/
│   │   │   └── signup.component.ts   # Signup page
│   │   ├── pending/
│   │   │   └── pending.component.ts  # Account pending screen
│   │   │
│   │   ├── dashboard/
│   │   │   ├── dashboard.component.ts       # Main fleet dashboard
│   │   │   └── vehicle-grid/
│   │   │       ├── vehicle-grid.component.ts    # Grid of vehicles
│   │   │       └── vehicle-card/
│   │   │           └── vehicle-card.component.ts  # Individual vehicle card
│   │   │
│   │   ├── fleet-map/
│   │   │   ├── fleet-map.component.ts           # Leaflet map view
│   │   │   └── vehicle-detail-panel/
│   │   │       └── vehicle-detail-panel.component.ts  # Side panel for selected vehicle
│   │   │
│   │   ├── vehicle-detail/
│   │   │   ├── vehicle-detail.component.ts      # Single vehicle detail page
│   │   │   ├── battery-analysis/
│   │   │   │   └── battery-analysis.component.ts
│   │   │   └── dtc-list/
│   │   │       └── dtc-list.component.ts
│   │   │
│   │   ├── alerts/
│   │   │   └── alerts.component.ts              # Active alerts list
│   │   │
│   │   ├── onboarding/
│   │   │   ├── organization/
│   │   │   │   └── onboarding-organization.component.ts  # Create org step
│   │   │   ├── fleet/
│   │   │   │   └── onboarding-fleet.component.ts        # Create fleet step
│   │   │   └── complete/
│   │   │       └── onboarding-complete.component.ts     # Onboarding complete
│   │   │
│   │   └── backoffice/
│   │       ├── fleet-list/
│   │       │   └── fleet-list.component.ts
│   │       ├── fleet-detail/
│   │       │   └── fleet-detail.component.ts
│   │       ├── vehicle-list/
│   │       │   └── vehicle-list.component.ts
│   │       ├── vehicle-edit/
│   │       │   └── vehicle-edit.component.ts
│   │       ├── device-list/
│   │       │   └── device-list.component.ts
│   │       ├── device-detail/
│   │       │   └── device-detail.component.ts
│   │       └── user-management/
│   │           └── user-management.component.ts
│   │
│   ├── shared/
│   │   ├── components/
│   │   │   ├── health-gauge/
│   │   │   │   └── health-gauge.component.ts
│   │   │   ├── battery-status/
│   │   │   │   └── battery-status.component.ts
│   │   │   ├── dtc-indicator/
│   │   │   │   └── dtc-indicator.component.ts
│   │   │   ├── toast/
│   │   │   │   └── toast.component.ts
│   │   │   ├── alert-banner/
│   │   │   │   └── alert-banner.component.ts
│   │   │   ├── loading-spinner/
│   │   │   │   └── loading-spinner.component.ts
│   │   │   └── vehicle-info/
│   │   │       └── vehicle-info.component.ts
│   │   │
│   │   └── data/
│   │       └── dtc-database.ts          # DTC code lookup table
│   │
│   └── core/interceptors/
│       └── auth.interceptor.ts          # HTTP interceptor for auth headers
│
└── environments/
    ├── environment.ts                   # Base environment config
    └── environment.development.ts       # Dev overrides (local Supabase)
```

## Angular Module Structure

**Configuration:** `packages/dashboard/src/app/app.config.ts` uses `ApplicationConfig` pattern (no NgModule). Providers include:
- `provideRouter(routes, withViewTransitions())` — lazy-loaded routes with view transitions
- `provideAuth0({ domain: 'ronbiter.auth0.com', ... })` — Auth0 SDK with refresh tokens
- `HTTP_INTERCEPTORS: [AuthInterceptor]` — attaches auth headers to HTTP requests
- `provideAnimationsAsync()` — Angular Material animations

**Routing:** `packages/dashboard/src/app/app.routes.ts`
- Public routes: `/login`, `/signup`, `/pending`
- Protected shell (`/` with `authGuard` + `authGuardChild`):
  - `/dashboard` — fleet overview grid
  - `/map` — Leaflet map view
  - `/vehicle/:id` — single vehicle detail
  - `/alerts` — active alerts
  - `/onboarding/*` — org/fleet creation flow (with `onboardingStepGuard`)
  - `/backoffice/*` — admin CRUD (fleets, vehicles, devices, users)
- Wildcard redirects to `/dashboard`

## Simulator Source Structure

```
packages/simulator/
├── package.json
├── src/
│   └── index.ts          # GhostFleetSimulator class (only source file)
└── (no subdirectories)
```

**Entry point:** `src/index.ts:218` — instantiates with env-based config and starts
**Exports:** `GhostFleetSimulator` class

## Supabase Migration Structure

```
packages/supabase/
├── migrations/
│   ├── 001_initial_schema.sql       # Core tables: telemetry_logs, fleets, users, vehicles, alerts, etc.
│   ├── 002_rls_policies.sql         # Row-level security policies
│   ├── 003_functions_triggers.sql   # Database functions and triggers
│   ├── 004_fleet_provisioning.sql   # Fleet provisioning tables
│   ├── 005_device_provisioning.sql  # Device provisioning tables
│   ├── 006_orgs_and_fleets.sql      # Organizations and fleet relationships
│   ├── 007_create_users_clerk.sql   # Users table (clerk integration — may be legacy)
│   └── 008_add_org_and_roles_to_users.sql  # Org/role additions to users
└── seed/
    └── 001_seed_data.sql            # Initial seed data
```

## Auth Service Source Structure

```
packages/auth-service/
├── package.json
├── src/
│   ├── main.ts                      # NestJS bootstrap
│   ├── app.module.ts                # AppModule (JwtModule, imports)
│   ├── auth/
│   │   ├── auth.controller.ts       # POST /auth/exchange, /refresh, /validate, /me
│   │   ├── auth.service.ts          # Auth0 JWKS verification, JWT signing
│   │   ├── jwt.strategy.ts          # Passport JWT strategy
│   │   └── dto.ts                   # ExchangeTokenDto, RefreshTokenDto
│   └── users/
│       └── users.service.ts         # User lookup operations
```

**App Module:** `packages/auth-service/src/app.module.ts` — `JwtModule.register({ global: true, secret: process.env.JWT_SECRET, expiresIn: '15m' })`

## Naming Conventions

**Files:**
- Angular components: `kebab-case.component.ts` (e.g., `vehicle-grid.component.ts`)
- Services: `kebab-case.service.ts` (e.g., `vehicle.service.ts`)
- Models: `kebab-case.model.ts` (e.g., `vehicle.model.ts`)
- Guards: `kebab-case.guard.ts` (e.g., `auth.guard.ts`)
- NestJS: `kebab-case.controller.ts`, `kebab-case.service.ts`, `kebab-case.module.ts`
- Parser/Simulator: Single `index.ts` in each

**Directories:** `kebab-case` (e.g., `vehicle-grid/`, `battery-analysis/`)

## Where to Add New Code

**New Parser Feature (new protocol, new metric):**
- Currently `packages/parser/src/index.ts` is a single file. Best practice: create `packages/parser/src/parser/` for protocol logic, `packages/parser/src/auth/` for device auth.
- Tests: `packages/parser/src/__tests__/` or co-located `*.test.ts`

**New Dashboard Page:**
- Component: `packages/dashboard/src/app/pages/{feature-name}/{feature-name}.component.ts`
- Route: Add to `packages/dashboard/src/app/app.routes.ts` (use `loadComponent()` lazy loading)
- Service: `packages/dashboard/src/app/core/services/{feature}.service.ts`
- Model: `packages/dashboard/src/app/core/models/{feature}.model.ts`
- Test: Colocated `*.spec.ts` (Karma + Jasmine)

**New Shared Component:**
- Implementation: `packages/dashboard/src/app/shared/components/{component-name}/`
- Data: `packages/dashboard/src/app/shared/data/`

**New Database Migration:**
- File: `packages/supabase/migrations/{NNN}_description.sql` (next sequential number)
- RLS policies: `packages/supabase/migrations/` (add new file, don't modify existing)

**New Auth Endpoint:**
- Controller: `packages/auth-service/src/auth/auth.controller.ts` (add `@Post()` method)
- Service: `packages/auth-service/src/auth/auth.service.ts` (add business logic)
- DTO: `packages/auth-service/src/auth/dto.ts` (add input validation DTO)

**New Simulator Feature:**
- Currently `packages/simulator/src/index.ts` is a single file. For new protocols, consider creating `packages/simulator/src/devices/` for device profiles.

## Special Directories

**docs/:**
- Purpose: Product requirements documents and architecture specifications
- Contains: `VitalsDrive_PRD.md`, `Architecture-Overview.md`, PRDs for each layer
- Not generated, committed manually

**packages/supabase/migrations/:**
- Purpose: Sequential SQL migrations for Supabase
- Generated: No, written manually
- Committed: Yes

**.env files:**
- Each package has `.env.example` — copy to `.env` before running
- Never commit `.env` (secrets)

---

*Structure analysis: 2026-05-07*
