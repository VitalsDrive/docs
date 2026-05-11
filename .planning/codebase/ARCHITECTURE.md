<!-- refreshed: 2026-05-07 -->
# Architecture

**Analysis Date:** 2026-05-07

## System Overview

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                        VitalsDrive Architecture                          │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐    ┌────────────────────────┐    ┌──────────────────┐ │
│  │ 4G OBD2      │───▶│ Parser                 │───▶│ Supabase         │ │
│  │ Devices      │    │ (Node.js TCP server)   │    │ PostgreSQL       │ │
│  │ (TCP client) │    │ packages/parser/       │    │ packages/supabase│ │
│  └──────────────┘    │ :5050                  │    │ Docker local     │ │
│                      └──────────┬─────────────┘    └────────┬─────────┘ │
│                               │                              │           │
│                      ┌────────▼─────────────┐    ┌───────────▼────────┐ │
│                      │ Ghost Fleet           │    │ Dashboard          │ │
│                      │ Simulator             │    │ (Angular 21 SPA)   │ │
│                      │ packages/simulator/   │    │ packages/dashboard/│ │
│                      │ (test device emulator)│    │ Vercel prod        │ │
│                      └──────────────────────┘    └────────────────────┘ │
│                                                                          │
│  ┌──────────────────────────────┐    ┌──────────────────────────────┐   │
│  │ Auth Service (NestJS)        │◀──▶│ Auth0 (External)              │   │
│  │ packages/auth-service/       │    │ External identity provider    │   │
│  │ :3001                        │    └──────────────────────────────┘   │
│  └──────────────────────────────┘                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| **OBD2Parser** | TCP binary protocol parsing, device auth, telemetry ingestion | `packages/parser/src/index.ts` |
| **GhostFleetSimulator** | Emulates OBD2 device TCP connections for testing | `packages/simulator/src/index.ts` |
| **Supabase Client** | Database tables, RLS, realtime subscriptions | `packages/supabase/migrations/` |
| **Dashboard (Angular)** | Fleet visualization, vehicle health, alerts, backoffice CRUD | `packages/dashboard/src/app/` |
| **Auth Service** | Auth0 token exchange → internal JWT (15min access, 7d refresh) | `packages/auth-service/src/auth/` |

## Pattern Overview

**Overall:** Microservices with shared database. Each package is independently deployable with its own `package.json` and `tsconfig.json`.

**Key Characteristics:**
- Parser and Simulator use the same binary protocol (Teltonika Codec 8 Extended format)
- Dashboard uses Supabase Realtime for live telemetry streaming (no polling)
- Auth uses Auth0 SDK → internal JWT exchange pattern
- Each package has a single entry point — flat source structure
- Angular uses signals-based reactivity, not RxJS `BehaviorSubject`
- Routes use `loadComponent()` lazy loading throughout

## Layers

**Device Layer (TCP):**
- Purpose: OBD2 devices connect over TCP to Parser on port 5050
- Location: `packages/parser/src/index.ts`, `packages/simulator/src/index.ts`
- Contains: Binary protocol parsing, device authentication via IMEI+credentials
- Depends on: Node.js `net` module, Supabase for device lookup
- Used by: Physical OBD2 devices + Simulator

**Storage Layer (Supabase):**
- Purpose: Persistent storage + real-time pub/sub
- Location: `packages/supabase/migrations/`
- Contains: Tables (`telemetry_logs`, `vehicles`, `fleets`, `alerts`, `users`, `devices`, etc.), RLS policies, indexes
- Depends on: PostgreSQL (local Docker or Supabase cloud)
- Used by: Parser (writes), Dashboard (reads + realtime subscriptions)

**Presentation Layer (Angular):**
- Purpose: Fleet health dashboard, vehicle detail, backoffice administration
- Location: `packages/dashboard/src/app/`
- Contains: Pages, services, guards, models, shared components
- Depends on: Supabase client, Auth0, Angular 21 + Material
- Used by: Fleet managers, admins

**Auth Layer (NestJS + Auth0):**
- Purpose: Identity verification and token management
- Location: `packages/auth-service/src/auth/`
- Contains: Token exchange, refresh, validation endpoints
- Depends on: Auth0 JWKS, `@nestjs/jwt`
- Used by: Dashboard (exchange on login), route guards

## Data Flow

### Primary Telemetry Path

1. OBD2 device sends TCP connection, then raw IMEI as 15 ASCII bytes (`packages/parser/src/index.ts:55`) — `handleConnection(socket)`
2. Parser responds with `0x01` ACK, marks device authenticated (`packages/parser/src/index.ts:100`) — login handling
3. Device sends Codec 8 Extended packets (0x00000000 preamble, codec 0x8E, AVL records), parser validates CRC-16, parses IO elements (`packages/parser/src/index.ts:140`) — `processBuffer()`
4. Parser looks up `device → vehicle_id` mapping, rejects inactive/unassigned devices (`packages/parser/src/index.ts:264`) — `processTelemetry()`
5. Parser inserts into `telemetry_logs` table (`packages/parser/src/index.ts:315`) — `writeToSupabase()`
6. Supabase triggers Realtime `postgres_changes` event on INSERT
7. Dashboard `TelemetryService` receives via Supabase Realtime channel (`packages/dashboard/src/app/core/services/telemetry.service.ts:33`) — `subscribeToFleet()`
8. `VehicleService` processes batch, computes health scores, triggers alert checks (`packages/dashboard/src/app/core/services/vehicle.service.ts:223`) — `processBatch()`

### Auth Flow

1. User navigates to dashboard → Auth0 redirect (`packages/dashboard/src/app/core/services/auth.service.ts:73`) — `signIn()`
2. After Auth0 login, `getAccessTokenSilently()` obtains Auth0 token
3. Token sent to Auth Service `POST /auth/exchange` (`packages/dashboard/src/app/core/services/auth.service.ts:89`)
4. Auth Service verifies Auth0 token via JWKS (`packages/auth-service/src/auth/auth.service.ts:19`) — `exchangeAuth0Token()`
5. Auth Service returns internal JWT access token (15min) + refresh token (7d)
6. Dashboard stores internal token for API calls
7. Route guards check `auth.isAuthenticated$`, `isAllowlisted`, `isOnboardingComplete` (`packages/dashboard/src/app/core/guards/auth.guard.ts`)

### Simulator → Parser Path

1. Simulator connects to Parser TCP port (`packages/simulator/src/index.ts:51`)
2. Sends login packet: `IMEI:xxxxx\n` or `imei,login,password\n` (`packages/simulator/src/index.ts:100`)
3. Parser responds with AUTHORIZED/OK
4. Simulator sends binary telemetry packet every `SEND_INTERVAL_MS` (`packages/simulator/src/index.ts:45`)
5. Same ingestion path as physical devices

### Onboarding Flow

1. First-time user → `onboardingStepGuard` blocks access to dashboard
2. Creates organization → `packages/dashboard/src/app/pages/onboarding/organization/`
3. Creates fleet → `packages/dashboard/src/app/pages/onboarding/fleet/`
4. Redirected to dashboard

## Key Abstractions

**TelemetryRecord:**
- Purpose: Unified telemetry data model across parser, database, and dashboard
- Examples: `packages/dashboard/src/app/core/models/telemetry.model.ts`, `packages/parser/src/index.ts` (TelemetryPacket)
- Pattern: Interface with `vehicle_id`, GPS coords, rpm, temp, voltage, dtc_codes

**VehicleWithHealth:**
- Purpose: Vehicle model enriched with computed health score and alert count
- Examples: `packages/dashboard/src/app/core/models/vehicle.model.ts`
- Pattern: Health score calculated from voltage thresholds, coolant temp, DTC count (`packages/dashboard/src/app/core/services/vehicle.service.ts:253`)

## Entry Points

**Parser:**
- Location: `packages/parser/src/index.ts:334`
- Triggers: OBD2 device TCP connection on port 5050
- Responsibilities: Device auth, binary protocol parsing, Supabase write

**Dashboard:**
- Location: `packages/dashboard/src/main.ts` (Angular bootstrap)
- Triggers: Browser navigation
- Responsibilities: Fleet visualization, vehicle management, alerts

**Simulator:**
- Location: `packages/simulator/src/index.ts:218`
- Triggers: Manual `npm start` (configured via env vars)
- Responsibilities: Emulate OBD2 device for testing

**Auth Service:**
- Location: `packages/auth-service/src/main.ts`
- Triggers: HTTP POST to `/auth/exchange`, `/auth/refresh`, `/auth/validate`, `/auth/me`
- Responsibilities: Auth0 token verification, internal JWT issuance

**Supabase:**
- Location: `packages/supabase/` (Docker Compose)
- Triggers: `supabase start` (local), Supabase Cloud (prod)
- Responsibilities: PostgreSQL database, realtime subscriptions, row-level security

## Architectural Constraints

- **Threading:** Single-threaded Node.js event loop in Parser. Each TCP connection is handled by a callback; no worker threads or concurrency primitives.
- **Global state:** `packages/parser/src/index.ts:333` — `OBD2Parser` singleton created at module load, calls `start()`. Per-connection state held in closure (`ConnectionState`).
- **Circular imports:** Not detected in current codebase. Parser is a single file. Dashboard services use `providedIn: 'root'` (tree-shakable, no barrel files).
- **Auth service hardcodes defaults:** `packages/auth-service/src/auth/auth.service.ts:6` — `AUTH0_DOMAIN` defaults to `ronbiter.auth0.com` (not suitable for multi-tenant).
- **Parser is a single file:** All logic in one 337-line class — `OBD2Parser` in `packages/parser/src/index.ts`.

## Anti-Patterns

### Parser is a monolithic class

**What happens:** All 337 lines of parsing logic live in `packages/parser/src/index.ts` — connection handling, auth, protocol parsing, database writes.
**Why it's wrong:** Difficult to test, difficult to extend with new protocols, no separation of concerns.
**Do this instead:** Split into `ConnectionHandler`, `ProtocolParser`, `DeviceAuth`, and `TelemetryWriter` modules. See `packages/dashboard/src/app/` for a well-structured service pattern.

### Dashboard auth service has hardcoded localhost URL

**What happens:** `packages/dashboard/src/app/core/services/auth.service.ts:89` — `fetch('http://localhost:3001/auth/exchange', ...)` is hardcoded.
**Why it's wrong:** Cannot work in production (Vercel deployment). The `environment.ts` file defines `supabaseUrl` but not the auth service URL.
**Do this instead:** Add `authServiceUrl` to `environment.ts` and `environment.development.ts`, read from there.

### Dashboard guard polling pattern

**What happens:** `packages/dashboard/src/app/core/guards/auth.guard.ts:11` — `while (auth.isLoading()) { await new Promise(resolve => setTimeout(resolve, 50)); }` — busy-wait polling in route guards.
**Why it's wrong:** Wastes CPU, can cause infinite loops if `isLoading` never becomes false.
**Do this instead:** Convert `isLoading` to an RxJS observable or Promise, use `firstValueFrom` or `async/await` on a single promise.

### Dashboard auth service uses mock/stub defaults

**What happens:** `packages/dashboard/src/app/core/services/auth.service.ts:167-179` — `initializeUserState()` returns hardcoded `true` for all flags. `packages/dashboard/src/app/core/services/auth.service.ts:182-185` — `checkUserRoles()` hardcodes `admin` and `owner` to `true`.
**Why it's wrong:** All guards pass regardless of actual user state. No real authorization checks.
**Do this instead:** Query Supabase `users` table for role membership; use Supabase RLS for data-level authorization.

## Error Handling

**Strategy:** Try-catch with console logging, no centralized error tracking.

**Patterns:**
- Parser: `try/catch` with `console.error` for each async operation (`packages/parser/src/index.ts:188`, `packages/parser/src/index.ts:263`)
- Dashboard services: `signal<string | null>` for error state, set on catch (`packages/dashboard/src/app/core/services/vehicle.service.ts:117`)
- Auth service: NestJS `UnauthorizedException` for auth failures (`packages/auth-service/src/auth/auth.controller.ts:14`)
- No error boundary components in Angular app
- No Sentry/Datadog integration

## Cross-Cutting Concerns

**Logging:** `console.log` and `console.error` throughout. No structured logging framework.
**Validation:** Device credentials validated in parser. Dashboard guards validate auth/onboarding state. No input validation middleware.
**Authentication:** Auth0 SDK → NestJS JWT exchange → Supabase session. Dashboard uses `@auth0/auth0-angular` for UI, internal JWT for API calls.

---

*Architecture analysis: 2026-05-07*
