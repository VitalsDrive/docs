# Integrations

**Analysis Date:** 2026-05-07

## External Services

### Supabase (Primary Backend)

- **Production project:** `odwctmlawibhaclptsew`
- **Production URL:** `https://odwctmlawibhaclptsew.supabase.co`
- **Local URL:** `http://localhost:54321` (Supabase CLI Docker)
- **Authentication:**
  - Parser uses `SUPABASE_SERVICE_ROLE_KEY` (full access, bypasses RLS) — env: `packages/parser/.env.example`
  - Dashboard uses anon key (RLS-scoped) — env: `packages/dashboard/src/environments/environment.development.ts`
  - Auth Service uses `SUPABASE_SERVICE_ROLE_KEY` for user lookups — env: `packages/auth-service` (not detected in repo)
- **Tables accessed:**
  - Parser writes: `telemetry_logs` (INSERT), `devices` (SELECT for IMEI lookup, UPDATE for last_seen)
  - Dashboard reads: all tables via Supabase JS client with Realtime subscriptions on `telemetry_logs`
  - Auth Service reads/writes: `users` table (findByExternalId, create, updateOrganization)
- **Realtime channels:** Dashboard subscribes to `postgres_changes` on `telemetry_logs` (INSERT events), both fleet-wide (`fleet-telemetry` channel) and per-vehicle (`vehicle-{id}` channels) — see `packages/dashboard/src/app/core/services/telemetry.service.ts`

### Auth0 (Identity Provider)

- **Domain:** `ronbiter.auth0.com`
- **Client ID:** `vlGLhmcqPYQWjWrHzC49fwYnJ54Segmk` (configured in `packages/dashboard/src/app/app.config.ts`)
- **Audience:** `https://ronbiter.auth0.com/api/v2/`
- **Scopes:** `openid profile email`
- **Token storage:** `localstorage` with `useRefreshTokens: true`
- **Redirect URI:** `window.location.origin`
- **Flow:** Auth0 login -> access token -> exchange with Auth Service (`POST /auth/exchange`) -> internal JWT -> API requests
- **JWKS endpoint:** `https://ronbiter.auth0.com/.well-known/jwks.json` — used by Auth Service to validate Auth0 tokens (RS256)
- **Note:** Auth0 is currently used for dashboard login. The auth-service performs token exchange but does not yet write to Supabase users table during exchange (hardcoded `orgId: 'default'`, `roles: ['owner']`)

### 4G OBD2 Hardware Devices (TCP Data Source)

- **Protocol:** Teltonika Codec 8 Extended binary TCP
- **Connection:** Raw TCP to parser on port 5050
- **Authentication:** Device sends raw 15-digit IMEI as ASCII, server responds `0x01`
- **Telemetry packet format (Codec 8 Extended):**
  - `[0-3]` Preamble: `0x00 0x00 0x00 0x00`
  - `[4-7]` Data length (uint32 BE)
  - `[8]` Codec ID: `0x8E`
  - `[9-10]` Num records (uint16 BE)
  - `...` AVL records with IO elements
  - `[N..N+1]` Num records repeat
  - `[N+2..N+3]` CRC-16 (IBM, polynomial 0x1021)
  - Server ACK: 4-byte big-endian record count
  - `[18-19]` CRC (uint16 LE)
  - `[20-21]` Stop: `0x0D 0x0A`

## Internal Service Communication

### Parser <-> Supabase

- **Direction:** Parser -> Supabase (writes only)
- **Client:** `@supabase/supabase-js` ^2.39.0 with service role key
- **Flow:** TCP connection -> IMEI auth -> binary packet parse -> IMEI lookup in `devices` table -> write to `telemetry_logs` -> update `last_seen` on `devices`
- **File:** `packages/parser/src/index.ts`
- **No response path** — parser does not read Supabase for anything other than device lookup/validation

### Dashboard <-> Supabase

- **Direction:** Dashboard <-> Supabase (bidirectional)
- **Client:** `@supabase/supabase-js` ^2.39.0 with anon key
- **Services:**
  - `packages/dashboard/src/app/core/services/supabase.service.ts` — creates Supabase client, manages auth state
  - `packages/dashboard/src/app/core/services/telemetry.service.ts` — Realtime subscriptions to `telemetry_logs`
- **Pattern:** Supabase RLS policies gate all data access; dashboard uses anon key with authenticated sessions

### Dashboard <-> Auth Service

- **Direction:** Dashboard -> Auth Service (HTTP POST)
- **Endpoint:** `POST http://localhost:3001/auth/exchange` (hardcoded in `packages/dashboard/src/app/core/services/auth.service.ts:89`)
- **Flow:** Auth0 login -> get access token -> POST to auth-service -> receive internal JWT -> attach to HTTP requests via `AuthInterceptor`
- **Note:** URL is hardcoded to localhost:3001 — no environment-based configuration. Will break in production without change.

### Auth Service <-> Supabase

- **Direction:** Auth Service -> Supabase (user lookups/writes)
- **Client:** `@supabase/supabase-js` ^2.104.1 with service role key
- **File:** `packages/auth-service/src/users/users.service.ts`
- **Operations:** `findByExternalId`, `findById`, `findByEmail`, `create`, `updateOrganization` on `users` table

### Simulator <-> Parser

- **Direction:** Simulator -> Parser (TCP only)
- **Connection:** TCP socket to `PARSER_HOST:PARSER_PORT` (default `localhost:5050`)
- **Protocol:** Same Teltonika Codec 8 Extended format as hardware devices
- **File:** `packages/simulator/src/index.ts`
- **No Supabase access** — simulator only sends data to parser

## Deployment & Hosting

| Service | Platform | Port | Notes |
|---------|----------|------|-------|
| Parser | Railway | 5050 | TCP ingestion server |
| Dashboard | Vercel | 4200 (dev) | Angular SPA, production build |
| Supabase | Supabase Cloud | 54321 (local) | Project: `odwctmlawibhaclptsew` |
| Simulator | Local only | N/A | Not deployed — dev testing tool |
| Auth Service | Not deployed | 3001 | Local development only |

### Environment Management

- **Parser:** `.env` from `.env.example` — `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `PARSER_PORT`, `LOG_LEVEL`
- **Simulator:** `.env` from `.env.example` — `PARSER_HOST`, `PARSER_PORT`, `VEHICLE_ID`, `DEVICE_IMEI`, `DEVICE_SMS_LOGIN`, `DEVICE_SMS_PASSWORD`, `SEND_INTERVAL_MS`, `LATITUDE`, `LONGITUDE`
- **Dashboard:** Angular environments — `environment.ts` (local), `environment.development.ts` (dev/prod Supabase URL + anon key)
- **Auth Service:** `AUTH0_DOMAIN`, `AUTH0_AUDIENCE`, `JWT_SECRET`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `PORT` (defaults present, no `.env.example` detected)
- **Supabase:** `PROJECT_REF` file for linking to remote project

## Production Gaps

- **Auth Service production URL** — dashboard hardcodes `localhost:3001` for token exchange. Needs environment-based URL configuration before production deployment.
- **Auth Service deployment target** — no deployment platform configured yet.
- **CI/CD pipelines** — no GitHub Actions or other CI detected across any repository.
- **@clerk/clerk-js dependency** — installed in dashboard but not wired into app config. Auth0 is the active provider. Dead dependency.

---

*Integration audit: 2026-05-07*
