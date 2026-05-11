# Codebase Concerns

**Analysis Date:** 2026-05-07

---

## Security Concerns

### CRITICAL: Supabase anon key hardcoded in development environment config

**File:** `packages/dashboard/src/environments/environment.development.ts`

The production Supabase URL (`odwctmlawibhaclptsew.supabase.co`) and a fully valid JWT anon key are committed to source code in the development environment file. This token grants read access to all Supabase tables and would be embedded in every dev build.

```typescript
supabaseUrl: 'https://odwctmlawibhaclptsew.supabase.co',
supabaseAnonKey: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
```

**Impact:** Anyone who clones the repo or gains access to it can query the production database directly, reading all vehicle telemetry, alerts, fleet data, and user information. RLS policies provide some protection but the anon key bypasses none of them -- an attacker with the key can enumerate all data accessible to the lowest-privilege user.

**Fix:** Move these values to environment variables. The `.env.example` pattern already exists in other packages but is missing from `dashboard/src/environments/`. Generate a new anon key immediately as this one should be considered exposed.

### CRITICAL: JWT secret hardcoded as fallback with weak default

**Files:**
- `packages/auth-service/src/app.module.ts` (line 12)
- `packages/auth-service/src/auth/jwt.strategy.ts` (line 11)

```typescript
secret: process.env.JWT_SECRET || 'vitalsdrive-secret-key',
```

The fallback secret `'vitalsdrive-secret-key'` is trivially guessable and is committed in source code. If `JWT_SECRET` is not set in the auth-service environment (a common first-run scenario), all JWT tokens can be forged.

**Impact:** Complete auth bypass. Any attacker who reads the source can mint valid tokens with arbitrary `orgId`, `roles`, and `userId` values.

**Fix:** Remove the fallback entirely. The service must fail to start if `JWT_SECRET` is not provided.

### CRITICAL: Production auth-service URL hardcoded in dashboard

**File:** `packages/dashboard/src/app/core/services/auth.service.ts` (line 89)

```typescript
const response = await fetch('http://localhost:3001/auth/exchange', {
```

The token exchange endpoint is hardcoded to `localhost:3001`. This will never work in production. Additionally, the URL uses `http` (unencrypted) which would expose tokens in transit even if the host were correct.

**Fix:** Make the auth-service base URL configurable via an environment variable or Angular environment config.

### HIGH: Auth0 configuration hardcoded with real credentials

**File:** `packages/dashboard/src/app/app.config.ts` (lines 28-29)

```typescript
domain: 'ronbiter.auth0.com',
clientId: 'vlGLhmcqPYQWjWrHzC49fwYnJ54Segmk',
```

The Auth0 domain and clientId are embedded directly in `appConfig`. While clientIds in Auth0 are not technically secrets, this creates a tight coupling to a specific personal Auth0 tenant (`ronbiter.auth0.com`) and makes it impossible to run against a different tenant without code changes.

**Fix:** Move to environment configuration.

### HIGH: Device SMS credentials stored in plaintext in database

**File:** `packages/supabase/migrations/007_create_users_clerk.sql` (lines 45-60)

The `devices.sms_login` and `devices.sms_password` columns store device authentication credentials in plaintext. No hashing is applied.

**Impact:** Database compromise exposes all device credentials, enabling device impersonation attacks.

**Fix:** At minimum, hash the `sms_password` field (e.g., bcrypt/argon2). If the OBD2 devices require plaintext comparison, consider encrypting at rest using Supabase's encryption features.

### HIGH: `users` table RLS policy allows all reads

**File:** `packages/supabase/migrations/007_create_users_clerk.sql` (line 25)

```sql
CREATE POLICY "Users can read own record" ON users
  FOR SELECT USING (true);
```

The `USING (true)` clause means any authenticated user can read all user records -- not just their own. This contradicts the policy name "Users can read own record."

**Impact:** Any logged-in user can enumerate all users in the system, including emails, display names, and organization associations.

**Fix:** Change to `USING (id = auth.uid())` or at minimum scope to users within the same organization.

### HIGH: Service role policy in migration 006 uses `auth.jwt()->>'role'`

**File:** `packages/supabase/migrations/006_orgs_and_fleets.sql` (lines 209, 370)

```sql
CREATE POLICY "Service role can manage organizations" ON organizations
    FOR ALL USING (auth.jwt()->>'role' = 'service_role');
```

This checks the JWT `role` claim for `service_role`. However, Supabase's service_role key automatically bypasses RLS -- this policy is redundant for the service role but creates a potential bypass: if a user JWT ever includes `role: "service_role"` in its claims (which the auth-service does with `roles: ['owner']` in the JWT payload at `auth-service/src/auth/auth.service.ts` line 29), they could trigger this policy check.

**Note:** The auth-service embeds `roles` (plural) in the JWT, not `role` (singular), so the check technically won't match. But the naming inconsistency is fragile and could lead to false security.

### MEDIUM: No auth check on invitations insert policy

**File:** `packages/supabase/migrations/006_orgs_and_fleets.sql` (lines 342-348)

```sql
CREATE POLICY "Anyone can create invitations" ON invitations
    FOR INSERT WITH CHECK (
```

Despite the policy name saying "Anyone," the CHECK clause does restrict to org admins. However, the name is misleading and should match the actual logic.

### MEDIUM: Parser uses `as any` casts to suppress TypeScript errors

**Files:**
- `packages/parser/src/index.ts` (line 269) -- `this.supabase.from('devices') as any`
- `packages/parser/src/index.ts` (line 325) -- `as any` on telemetry insert

These casts suppress type checking, meaning the parser can silently write data with wrong column names or types.

---

## Technical Debt

### CRITICAL: Auth-service migration from Auth0 to Supabase Auth is incomplete

The codebase is in a transitional state between Auth0 and Supabase Auth. The dashboard still uses `@auth0/auth0-angular` as its primary auth library (`app.config.ts` line 11), but the database schema has been migrated for Supabase Auth (`001_initial_schema.sql` references `auth.users`).

**Evidence:**
- `auth.service.ts` imports `Auth0Service` and uses `auth0.loginWithRedirect()`, `auth0.getAccessTokenSilently()`
- `app.config.ts` configures `provideAuth0({ domain: 'ronbiter.auth0.com', ... })`
- `auth.interceptor.ts` attaches tokens from the auth service
- But `supabase.service.ts` uses Supabase's own auth (`this._client.auth.getSession()`)
- Two competing auth systems running simultaneously

**Impact:** This is the single largest debt item. The system will not function correctly in production until one auth provider is chosen and the other is fully removed.

**Fix:** Choose one auth provider. If Supabase Auth is the target:
1. Replace `@auth0/auth0-angular` with `@supabase/auth-js` in the dashboard
2. Remove the auth-service entirely (Supabase Auth handles JWT directly)
3. Remove `packages/auth-service/` or repurpose it
4. Update all guards and services to use Supabase auth signals

### HIGH: Mock/stub implementations in auth service

**File:** `packages/dashboard/src/app/core/services/auth.service.ts`

Several methods return hardcoded mock values:

```typescript
async initializeUserState(): Promise<UserState> {
  const state: UserState = {
    isAllowlisted: true,
    isOnboardingComplete: true,
    hasOrganization: true,
    hasFleet: true,
    organizationId: "mock-org-1",
  };
  ...
}

async checkUserRoles(): Promise<void> {
  this.isAdmin.set(true);
  this.isOwner.set(true);
}

async completeOnboarding(fleetName?: string): Promise<{ success: boolean; error: string | null }> {
  await this.initializeUserState();
  await this.checkUserRoles();
  return { success: true, error: null };
}
```

**Impact:** Onboarding, role checks, and user initialization are completely stubbed. The dashboard will show all users as admins/owners with complete onboarding regardless of actual database state.

### HIGH: Hardcoded `orgId: 'default'` and `roles: ['owner']` in auth-service

**File:** `packages/auth-service/src/auth/auth.service.ts` (lines 26-30)

Every user who exchanges an Auth0 token receives the same hardcoded payload:

```typescript
const payload = {
  sub: userId,
  email: email,
  orgId: 'default',
  roles: ['owner'],
};
```

No database lookup is performed to determine the user's actual organization or roles. Every user is granted owner-level access to the default org.

**Fix:** The auth-service should query the `users` table (via `UsersService`) to retrieve actual org membership and roles before signing the JWT.

### HIGH: TODO comment in users service

**File:** `packages/auth-service/src/users/users.service.ts` (line 106)

```typescript
roles: ['owner'], // TODO: implement proper role lookup
```

Same issue as above -- hardcoded role. This is the server-side equivalent of the dashboard mock.

### MEDIUM: Two competing `users` table schemas

**Files:**
- `packages/supabase/migrations/001_initial_schema.sql` -- defines `users` with `id UUID REFERENCES auth.users(id)`, `role VARCHAR(20) CHECK (role IN ('admin', 'editor', 'viewer'))`
- `packages/supabase/migrations/007_create_users_clerk.sql` -- defines `users` with `id UUID DEFAULT gen_random_uuid()`, `external_auth_id TEXT`, `external_auth_provider TEXT`
- `packages/supabase/migrations/008_add_org_and_roles_to_users.sql` -- adds `roles TEXT[]` and `organization_id UUID`

Migration `007` uses `CREATE TABLE IF NOT EXISTS`, which means it may or may not apply depending on migration order. The `users` table schema is ambiguous and likely inconsistent between environments.

**Impact:** The parser queries `devices` table by IMEI (correct), but other services query `users` with incompatible column assumptions.

### MEDIUM: `environment.ts` (production config) points to local Supabase

**File:** `packages/dashboard/src/environments/environment.ts`

```typescript
export const environment = {
  production: false,
  supabaseUrl: 'http://localhost:54321',
  supabaseAnonKey: 'your-local-anon-key'
};
```

This is the default Angular environment file (used when building without `--configuration=development`). It contains placeholder values that will cause the app to fail silently if someone runs `npm run build` without specifying a configuration.

### MEDIUM: `ng serve` vs `npm start` naming inconsistency

The CLAUDE.md and dashboard README say `npm start` runs the Angular dev server, but Angular's default is `ng serve`. The root `package.json` uses `npm run dev:dashboard`. Three different command names for the same operation across the codebase.

### MEDIUM: `devices` table referenced but never created in migrations

The parser (`packages/parser/src/index.ts` line 198) queries `this.supabase.from('devices')`, but no migration file creates the `devices` table. Migration `005_device_provisioning.sql` assumes the table exists ("NOTE: devices table ... was already applied via Supabase MCP"). This means the devices table was likely created manually via the Supabase dashboard, not through the migration system.

**Impact:** Running migrations on a fresh database will leave the `devices` table missing, causing the parser to fail on first connection.

### LOW: Conflicting migration ordering issues

**File:** `packages/supabase/migrations/006_orgs_and_fleets.sql`

Migration 006 drops the `organizations` table at line 74 (`DROP TABLE IF EXISTS organizations CASCADE;`) then recreates it. This is destructive and will fail if other objects depend on it. The migration also attempts to `INSERT INTO organizations ... SELECT ... FROM organizations` after dropping the table -- this query will always return zero rows in the second pass.

---

## Production Readiness Gaps

### CRITICAL: No logging framework -- all logging uses `console.log`

**Files:** `packages/parser/src/index.ts`, `packages/simulator/src/index.ts`, `packages/auth-service/src/auth/*.ts`

No structured logging library (winston, pino, etc.) is in use. Parser logs raw telemetry packets to stdout including vehicle IDs, IMEIs, and parsed values. No log levels, no correlation IDs, no log rotation.

**Impact:** In production, logs will be unstructured, unsearchable, and will grow without bound. No way to trace a single vehicle's data through the pipeline.

**Fix:** Add `pino` (parser/auth-service) with request IDs. Include IMEI/vehicle_id in every log line for that connection.

### HIGH: No monitoring, alerting, or health checks

None of the three server packages (parser, auth-service, simulator) expose a health check endpoint. No metrics (Prometheus, DataDog) are configured. No uptime monitoring.

**Fix:** Add `GET /health` endpoints to parser and auth-service. Wire up Supabase dashboard alerts for device connectivity drops.

### HIGH: No telemetry deduplication

**File:** `packages/parser/src/index.ts` (lines 258-261)

Every telemetry packet received is written to `telemetry_logs` with no deduplication. If a device retransmits (common with poor cellular connectivity), duplicate rows will accumulate.

**Fix:** Add a unique constraint on `(vehicle_id, timestamp)` or implement an upsert pattern.

### HIGH: Parser connection state leaks

**File:** `packages/parser/src/index.ts` (lines 55-79)

The parser creates a new `ConnectionState` per socket but has no connection limit. A flood of connections (intentional or from device retry storms) will exhaust memory. The `net.createServer()` has no `maxConnections` configured.

**Fix:** Set `server.maxConnections` to a reasonable limit (e.g., 1000). Add a connection tracking map to reject duplicate IMEIs connecting simultaneously.

### MEDIUM: `telemetry_logs` table will grow unbounded

**File:** `packages/supabase/migrations/001_initial_schema.sql`

The `telemetry_logs` table uses `BIGSERIAL` with no partitioning, archiving, or automatic cleanup. With 5-second reporting intervals per vehicle, a fleet of 100 vehicles generates 1.7 million rows per day.

The `cleanup_old_telemetry()` function exists in migration `003` but nothing schedules it. No cron job, no pg_cron extension, no Supabase scheduled function.

**Fix:** Enable Supabase's Edge Functions scheduler or set up pg_cron to call `cleanup_old_telemetry()` daily. Consider table partitioning by `timestamp`.

### MEDIUM: No API rate limiting

The auth-service has no rate limiting on the `/auth/exchange` or `/auth/refresh` endpoints. An attacker could brute-force token exchange or flood the token refresh endpoint.

**Fix:** Add `express-rate-limit` or NestJS throttler guard.

### MEDIUM: Simulator does not handle connection failures gracefully

**File:** `packages/simulator/src/index.ts` (lines 81-97)

When the `close` event fires, the simulator reschedules `runSimulation()` but creates a new socket each time without closing old connections. If the parser is down for extended periods, the simulator will accumulate connection state in memory.

### LOW: No Docker or deployment configuration

No Dockerfiles, docker-compose files, or deployment manifests exist for any package. Deployment relies on manual `npm run build && npm run start` commands.

---

## Code Quality Issues

### MEDIUM: Duplicate `is_authenticated` logic in parser and simulator

Both `packages/parser/src/index.ts` and `packages/simulator/src/index.ts` implement their own authentication state machines. The protocol details (packet format, login sequence) are duplicated between the two packages. If the protocol changes, both must be updated.

**Fix:** Extract the protocol handling into a shared package or at least share the packet builder/parser.

### MEDIUM: `alert.service.ts` creates a new instance of `DtcTranslationService`

**File:** `packages/dashboard/src/app/core/services/alert.service.ts` (line 16)

```typescript
private readonly dtcService = new DtcTranslationService();
```

Rather than injecting via Angular's DI, the alert service directly instantiates `DtcTranslationService`. This breaks DI and makes testing harder.

### MEDIUM: `auth.interceptor.ts` uses old HTTP interceptor pattern

**File:** `packages/dashboard/src/app/core/interceptors/auth.interceptor.ts`

Uses the deprecated `HttpInterceptor` class-based interface rather than the newer functional `HttpInterceptorFn` introduced in Angular 15+. The interceptor is registered in `app.config.ts` using `HTTP_INTERCEPTORS` provider.

### MEDIUM: `auth.guard.ts` uses polling to wait for auth state

**File:** `packages/dashboard/src/app/core/guards/auth.guard.ts` (lines 11-13)

```typescript
while (auth.isLoading()) {
  await new Promise(resolve => setTimeout(resolve, 50));
}
```

This busy-wait polling pattern blocks route navigation and wastes CPU cycles. Should use an `Observable` or `Promise`-based approach instead.

### LOW: `generateId()` uses a global counter that resets on page reload

**File:** `packages/dashboard/src/app/core/services/alert.service.ts` (lines 9-12)

```typescript
let alertIdCounter = 0;
function generateId(): string {
  return `alert-${Date.now()}-${++alertIdCounter}`;
```

If two alerts are generated in the same millisecond after a reload, the counter starts at 0 again, creating potential ID collisions.

### LOW: Inconsistent error handling patterns

- Parser: `console.error` + return silently (lines 188-189, 263-264)
- Auth-service: `throw new UnauthorizedException` (consistent)
- Dashboard services: `try/catch` + set error signal (consistent within package)

Each package handles errors differently. No cross-package error handling convention.

---

## Scalability Concerns

### HIGH: `telemetry_logs` has no partitioning or time-series optimization

With `BIGSERIAL` primary key and simple B-tree indexes on `(vehicle_id, timestamp DESC)`, queries for "latest telemetry per vehicle" must scan large portions of the index. At scale (>1M rows), this degrades.

**Fix:** Implement PostgreSQL table partitioning by month on the `timestamp` column.

### MEDIUM: Subquery-heavy RLS policies

Multiple RLS policies use nested subqueries against `fleet_members`:

```sql
SELECT fleet_id FROM fleet_members WHERE user_id = auth.uid()
```

While indexed, these subqueries execute on every row access. For tables with millions of rows (e.g., `telemetry_logs`), the `vehicle_id IN (SELECT v.id FROM vehicles v WHERE v.fleet_id IN (SELECT ...))` pattern in the telemetry SELECT policy becomes expensive.

**Fix:** The `get_user_fleet_ids()` helper function already exists in migration `002` but is not consistently used. Update all RLS policies to use this function.

### MEDIUM: `get_fleet_statistics` joins across 3 large tables

**File:** `packages/supabase/migrations/003_functions_triggers.sql` (lines 125-150)

```sql
SELECT COUNT(DISTINCT v.id) ...
FROM vehicles v
LEFT JOIN alerts a ON v.id = a.vehicle_id
LEFT JOIN telemetry_logs t ON v.id = t.vehicle_id
WHERE v.fleet_id = p_fleet_id;
```

This query joins `vehicles`, `alerts`, and `telemetry_logs` without a `LIMIT`. For fleets with many vehicles and high telemetry volume, this will be extremely slow.

### LOW: Simulator creates a new TCP connection per telemetry burst

**File:** `packages/simulator/src/index.ts` (line 48)

Each `runSimulation()` call creates a new `net.Socket()`, connects, sends one packet, and ends the connection. This is wasteful and doesn't match real device behavior (which maintains a persistent TCP connection).

---

## Schema Inconsistencies

### HIGH: `telemetry_logs` columns don't match parser writes

**Parser writes** (`packages/parser/src/index.ts` lines 314-324):
```typescript
.insert({
  vehicle_id, lat, lng, temp, voltage, rpm, dtc_codes, timestamp
})
```

**Dashboard expects** (`packages/dashboard/src/app/core/services/telemetry.service.ts` lines 98-114):
```typescript
{ vehicle_id, timestamp, rpm, speed, engine_on, coolant_temp, voltage, latitude, longitude, dtc_codes, fuel_level, signal_strength, raw_packet }
```

Column name mismatches:
- Parser writes `temp`, dashboard reads `coolant_temp`
- Parser writes `lat`, dashboard reads `latitude`
- Parser writes `lng`, dashboard reads `longitude`
- Dashboard expects `speed`, `engine_on`, `fuel_level`, `signal_strength`, `raw_packet` -- none of which the parser writes

**Impact:** The real-time dashboard will show NaN or undefined for most fields because the column names don't match between what the parser inserts and what the telemetry service maps.

---

## Missing Features

### CRITICAL: No alert generation for real-time alerts in the database

The `generate_alert_if_needed()` trigger in migration `003` creates alerts based on `telemetry_rules`, but no default rules exist in the seed data. The dashboard's `AlertService` does its own client-side alert evaluation, meaning alerts are local to the browser session and are not persisted or shared across users.

### HIGH: No device provisioning API

Devices are supposed to self-register via the `register_device_and_vehicle()` RPC function, but no API endpoint calls this function. The parser handles IMEI authentication but doesn't call the provisioning RPC.

### MEDIUM: No user management in auth-service

The `UsersService` exists but is never called by the auth controller or service. The `/users` endpoint is missing. User lookup during token exchange always returns hardcoded values.

---

## Test Coverage Gaps

### HIGH: No parser tests

**Directory:** `packages/parser/`

No test files exist (`*.test.ts`, `*.spec.ts`). The parser is the most critical component -- it handles untrusted TCP input from external devices and writes to the database. No packet parsing tests, no auth validation tests, no buffer handling tests.

### HIGH: No auth-service tests

**Directory:** `packages/auth-service/`

No test files exist. JWT validation, token exchange, and refresh logic are completely untested.

### HIGH: Simulator test gap

**Directory:** `packages/simulator/`

No test files exist. Packet building and data generation are untested.

### MEDIUM: Dashboard has one spec file

**File:** `packages/dashboard/src/app/app.spec.ts`

Only `app.spec.ts` exists with a basic component instantiation test. No tests for services, guards, interceptors, components, or alert processing logic.

---

## Dependencies at Risk

### MEDIUM: Auth0 integration being deprecated but still primary

The project plans to migrate to Supabase Auth (per CLAUDE.md: "Initial commit with Auth0 setup (to be replaced with Supabase Auth)"), but the migration has not been completed. Continued reliance on Auth0 creates:
- Dependency on a service that may be deprecated
- Cost per active user that scales with fleet size
- Latency from token exchange through auth-service

---

*Concerns audit: 2026-05-07*
