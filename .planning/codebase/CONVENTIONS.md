# Coding Conventions

**Analysis Date:** 2026-05-07

## Naming Conventions

### File naming

- **Dashboard (Angular)**: `kebab-case.component.ts` for components, `kebab-case.service.ts` for services, `kebab-case.model.ts` for model interfaces, `kebab-case.guard.ts` for route guards. Example: `vehicle-grid.component.ts`, `telemetry.service.ts`, `auth.guard.ts`.
- **Parser / Simulator**: `camelCase` or `kebab-case` for source files under `src/`. Single entry point `index.ts`.
- **Auth Service (NestJS)**: NestJS convention: `auth.controller.ts`, `auth.service.ts`, `jwt.strategy.ts`.

### Class / Component naming

- **Angular components**: `PascalCaseComponent` — `DashboardComponent`, `VehicleCardComponent`, `HealthGaugeComponent`.
- **Angular services**: `PascalCaseService` — `TelemetryService`, `VehicleService`, `AlertService`.
- **Parser / Simulator classes**: `PascalCase` — `OBD2Parser` (`packages/parser/src/index.ts:33`), `GhostFleetSimulator` (`packages/simulator/src/index.ts:25`).
- **NestJS**: Standard NestJS naming — `AuthService`, `JwtStrategy`, `UsersService`.

### Interface / Type naming

- **Interfaces**: `PascalCase` — `TelemetryPacket` (`packages/parser/src/index.ts:5`), `Vehicle` (`packages/dashboard/src/app/core/models/vehicle.model.ts:3`), `SimulatedData` (`packages/simulator/src/index.ts:15`).
- **Type aliases**: `PascalCase` — `VehicleStatus`, `AlertSeverity`, `ConnectionStatus`, `GaugeSize`.
- **Model interfaces** use `interface` keyword (not `type`) for object shapes.

### Variable / property naming

- **Local variables / params**: `camelCase` — `vehicleId`, `remoteAddress`, `baseLat`.
- **Angular signals**: `camelCase` — `isLoading`, `error`, `currentUser`, `showPassword`.
- **Angular inputs**: `camelCase` with `input()` — `value`, `size`, `showLabel` (`HealthGaugeComponent`).
- **Private fields**: Prefixed with underscore in some cases (`_alerts` in `AlertService`) or `private readonly` without underscore (`supabase` in `VehicleService`). No consistent convention across services.
- **Constants**: `UPPER_SNAKE_CASE` — `MIN_TEMP`, `MAX_TEMP`, `CRITICAL_DTC_CODES`, `MAX_TELEMETRY_HISTORY`.

### Function naming

- **Public methods**: `camelCase` — `loadVehicles()`, `processBatch()`, `subscribeToFleet()`.
- **Private methods**: `camelCase` — `calculateHealthScore()`, `mapPayload()`, `buildPacket()`.
- **Guards**: `camelCase` ending in `Guard` — `authGuard`, `adminGuard`, `fleetAdminGuard`, `orgOwnerGuard`.

## Code Style

### TypeScript config

| Package | strict | target | module | decorator support |
|---------|--------|--------|--------|-------------------|
| **parser** | `true` | ES2022 | commonjs | No |
| **simulator** | `true` | ES2022 | commonjs | No |
| **dashboard** | `true` | ES2022 | ES2022 (bundler) | Yes (`experimentalDecorators`) |
| **auth-service** | `false` (noImplicitAny: false) | ES2021 | commonjs | Yes (`experimentalDecorators`, `emitDecoratorMetadata`) |

**Dashboard angularCompilerOptions** (strict mode):
- `strictTemplates: true`
- `strictInjectionParameters: true`
- `strictInputAccessModifiers: true`
- `noImplicitOverride: true`
- `noPropertyAccessFromIndexSignature: true`

**Auth-service** is the most permissive: `strictNullChecks: false`, `noImplicitAny: false`, `forceConsistentCasingInFileNames: false`.

### ESLint / linting

- **Dashboard**: `ng lint` configured via Angular CLI. No local `.eslintrc` found — uses Angular Schematics defaults.
- **Parser**: `eslint src/**/*.ts` in `package.json`. No config file present — uses global/workspace config or defaults.
- **Simulator / Auth-service**: No lint script or config detected.

No Prettier config files detected anywhere. No `.prettierrc`, `biome.json`, or `.editorconfig`.

### Import ordering

- **Dashboard components**: Angular framework imports first, then third-party, then local relative imports. No explicit blank-line separation enforced.
  ```
  import { ChangeDetectionStrategy, Component, inject } from '@angular/core';
  import { MatButtonModule } from '@angular/material/button';
  import { AuthService } from '../../core/services/auth.service';
  ```
- **Parser / Simulator**: Node built-in first (`net`), then npm packages (`@supabase/supabase-js`), then local interfaces/classes.
- **Dashboard paths alias**: `@env` maps to `./src/environments/environment` in `tsconfig.json`. No other path aliases.

## Architectural Conventions

### Angular (Dashboard)

**Component style**: Standalone components exclusively. All components use `standalone: true` implicitly via the `imports` array (Angular 21 default). No NgModules used for component declarations.

**Component decorator patterns**:
- `selector: 'app-{name}'` — always prefixed with `app-`.
- `templateUrl` + `styleUrl` — external template and CSS (not inline). The root `App` component uses inline template (`template: '<router-outlet />'`).
- `changeDetection: ChangeDetectionStrategy.OnPush` — used consistently on all components observed.
- `imports: [...]` — lists standalone dependencies.

**Component file co-location**: Each component has its own directory with `.component.ts`, `.component.html`, `.component.css`. Example:
```
packages/dashboard/src/app/pages/dashboard/dashboard.component.ts
packages/dashboard/src/app/pages/dashboard/dashboard.component.html
packages/dashboard/src/app/pages/dashboard/dashboard.component.css
```

**Signal-based state management**: All services and components use Angular signals (`signal()`, `computed()`, `input()`) rather than RxJS `BehaviorSubject` for reactive state. RxJS `Subject` is used only for teardown (`destroy$`) and stream subscriptions from Supabase Realtime.

**Dependency injection**: Uses `inject()` function exclusively. No constructor injection for dependencies:
```
private readonly vehicleService = inject(VehicleService);
private readonly alertService = inject(AlertService);
```

**Service pattern**: `@Injectable({ providedIn: 'root' })` — all services are root-scoped singletons. No module-level providers. Services implement `OnDestroy` and use `Subject<void>` + `takeUntil` pattern for cleanup.

**Routing**: Route-based lazy loading via `loadComponent` for all page components. No eager-loaded page routes. Routes defined in `packages/dashboard/src/app/app.routes.ts`.

**Route guards hierarchy** (`packages/dashboard/src/app/core/guards/`):
| Guard | Purpose | File |
|-------|---------|------|
| `authGuard` | Redirects to `/login` if unauthenticated | `auth.guard.ts` |
| `authGuardChild` | Same as authGuard for child routes | `auth.guard.ts` |
| `allowlistGuard` | Redirects to `/onboarding` if no org membership | `auth.guard.ts` |
| `onboardingGuard` | Redirects to `/onboarding` if setup incomplete | `auth.guard.ts` |
| `onboardingStepGuard` | Prevents skipping onboarding steps | `auth.guard.ts` |
| `adminGuard` | Requires `owner` or `admin` role in org | `admin.guard.ts` |
| `ownerGuard` | Requires `owner` role only | `admin.guard.ts` |
| `orgAdminGuard` | Requires `owner` or `admin` role | `org-admin.guard.ts` |
| `orgOwnerGuard` | Requires `owner` role only | `org-owner.guard.ts` |
| `fleetAdminGuard` | Requires `owner` or `admin` for specific fleet | `fleet-admin.guard.ts` |

Guard pattern: function-based `CanActivateFn` / `CanActivateChildFn` using `inject()` for DI. All guards are async and return `boolean | UrlTree`. Guards poll `auth.isLoading()` with a 50ms setTimeout loop until auth state resolves — consider replacing with proper async stream.

**App config**: `ApplicationConfig`-based providers in `app.config.ts`. No `AppModule`. Auth0 configured via `provideAuth0()` with hardcoded `domain` and `clientId` (should be moved to environment).

**Interceptor**: `AuthInterceptor` (`core/interceptors/auth.interceptor.ts`) attaches `Bearer` token from `AuthService.getInternalToken()` to all HTTP requests. Uses class-based `HttpInterceptor` interface (not functional).

### Backend (Parser)

**Pattern**: Single class `OBD2Parser` with constructor-based Supabase client initialization. No dependency injection. Stateful socket connection management via per-connection `ConnectionState` objects. All in one file (`src/index.ts` ~337 lines). Entry point is IIFE-style instantiation at module bottom.

### Simulator

**Pattern**: Class `GhostFleetSimulator` with config object injected via constructor. Each simulation cycle creates a new TCP socket, sends login, then telemetry, then disconnects. Configured via environment variables.

### Auth Service (NestJS)

**Pattern**: Standard NestJS module/controller/service/strategy structure.
- `AuthModule` with `JwtStrategy` using `passport-jwt`.
- Token exchange endpoint: `POST /auth/exchange` — accepts Auth0 token, validates via JWKS, returns internal JWT.
- Hardcoded fallback JWT secret: `'vitalsdrive-secret-key'` in `jwt.strategy.ts:11`.

### Model / data interfaces

Models live in `packages/dashboard/src/app/core/models/`. They are pure TypeScript interfaces with no methods. Helper functions co-located (e.g., `getVehicleDisplayName` in `vehicle.model.ts`).

```
core/models/
  alert.model.ts        — Alert, AlertSeverity, AlertStatus, AlertType
  device.model.ts       — Device
  dtc.model.ts          — DtcEntry
  fleet.model.ts        — Fleet
  organization.model.ts — Organization
  telemetry.model.ts    — TelemetryRecord, ConnectionStatus
  user.model.ts         — User
  vehicle.model.ts      — Vehicle, VehicleWithHealth, VehicleStatus
```

## Environment Configuration

### Backend packages (Parser, Simulator)

- `.env.example` files present in `packages/parser/` and `packages/simulator/`.
- `dotenv` loaded via `import 'dotenv/config'` at top of entry files.

**Parser env vars** (`packages/parser/.env.example`):
```
SUPABASE_URL
SUPABASE_SERVICE_ROLE_KEY
PARSER_PORT
LOG_LEVEL
```

**Simulator env vars** (`packages/simulator/.env.example`):
```
PARSER_HOST
PARSER_PORT
VEHICLE_ID
SEND_INTERVAL_MS
LATITUDE
LONGITUDE
```

### Dashboard

- Two environment files: `environment.ts` (default, contains production Supabase config) and `environment.development.ts` (local, points to `localhost:54321`).
- Structure: `{ production: boolean, supabase: { url: string, anonKey: string } }`.
- Note: `environment.ts` contains the production anon key — this is checked into git. Consider using environment-specific injection for production builds.
- Auth0 configuration is **hardcoded** in `app.config.ts` (domain, clientId, audience). Should be moved to `environment.ts`.

### Deployment

- **Parser**: Railway deployment via Railway CLI. Configured via Railway dashboard environment variables.
- **Dashboard**: Vercel deployment. Environment variables configured via Vercel dashboard.
- No `railway.json`, `vercel.json`, or Dockerfiles committed.

### Auth Service

- Hardcoded fallbacks for Auth0 domain (`ronbiter.auth0.com`) and audience.
- `JWT_SECRET` from env var with hardcoded fallback (security risk).
- Auth exchange URL hardcoded in `auth.service.ts`: `http://localhost:3001/auth/exchange`.

## CSS / Styling Conventions (Dashboard)

- **Global styles**: `packages/dashboard/src/styles.css` — comprehensive design token system using CSS custom properties.
- **Theme**: Dark theme only (`:root`). Light theme reserved via `[data-theme='light']` placeholder.
- **Fonts**: Inter (sans-serif) + JetBrains Mono (code).
- **Component styles**: External `.css` files co-located with each component. No SCSS/SASS/LESS.
- **CSS naming**: BEM-like utility classes (`.text-primary`, `.badge-healthy`, `.card`, `.flex`, `.gap-2`).
- **Semantic color tokens**: `--color-healthy`, `--color-warning`, `--color-critical`, `--color-info`, `--color-predictive`.
- **Spacing**: `--space-N` tokens (4px increments from 1 to 12).
- **Angular Material overrides**: Extensive Material component overrides in `styles.css` using `!important` for theming.

---

*Convention analysis: 2026-05-07*
