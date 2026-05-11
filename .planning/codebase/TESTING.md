# Testing Patterns

**Analysis Date:** 2026-05-07

## Test Infrastructure

### Parser (`packages/parser/`)

- **Runner**: Jest (declared in `package.json`: `"test": "jest"`)
- **Config**: No `jest.config.js`, `jest.config.ts`, or Jest section in `package.json` detected. Uses Jest defaults.
- **Dev dependencies**: No `jest` or `@types/jest` listed in `package.json`. Jest script will fail unless installed globally or inherited from workspace.
- **Actual tests**: **None**. Zero `.test.ts` or `.spec.ts` files found in `packages/parser/`.

### Simulator (`packages/simulator/`)

- **Runner**: Jest (declared in `package.json`: `"test": "jest"`)
- **Config**: No Jest config file detected.
- **Dev dependencies**: No `jest` or `@types/jest` listed in `package.json`.
- **Actual tests**: **None**. Zero `.test.ts` or `.spec.ts` files found in `packages/simulator/`.

### Dashboard (`packages/dashboard/`)

- **Runner**: Angular CLI (`ng test`) — Karma + Jasmine (default Angular test runner)
- **Config**: No `karma.conf.js` detected. May be using Angular 21 defaults.
- **Dev dependencies**: Karma and Jasmine are not explicitly listed in `package.json` — likely included via `@angular-devkit/build-angular`.
- **Actual tests**: **One test file** found: `packages/dashboard/src/app/app.spec.ts`

### Auth Service (`packages/auth-service/`)

- **Runner**: Not configured. No `test` script in `package.json`.
- **Dev dependencies**: No Jest, no testing library.
- **Actual tests**: **None**.

## Existing Tests

### Dashboard — `app.spec.ts`

Location: `packages/dashboard/src/app/app.spec.ts`

```typescript
import { TestBed } from '@angular/core/testing';
import { App } from './app';

describe('App', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [App],
    }).compileComponents();
  });

  it('should create the app', () => {
    const fixture = TestBed.createComponent(App);
    const app = fixture.componentInstance;
    expect(app).toBeTruthy();
  });

  it('should render title', async () => {
    const fixture = TestBed.createComponent(App);
    await fixture.whenStable();
    const compiled = fixture.nativeElement as HTMLElement;
    expect(compiled.querySelector('h1')?.textContent).toContain('Hello, dashboard');
  });
});
```

This is the default Angular scaffold test. The second test (`should render title`) references an `h1` with "Hello, dashboard" which does not exist in the actual `App` component (which only renders `<router-outlet />`). This test will fail.

### Coverage

| Package | Test Files | Lines Covered | Notes |
|---------|------------|---------------|-------|
| parser | 0 | 0% | No tests at all |
| simulator | 0 | 0% | No tests at all |
| dashboard | 1 (default scaffold) | ~0% | Only tests root App component (scaffold) |
| auth-service | 0 | 0% | No test script configured |

**No coverage target is enforced.** No `--coverage` flags or coverage thresholds found.

## Test Patterns

### Observed patterns from `app.spec.ts`

**Test suite structure** (Jasmine-style):
```typescript
describe('App', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [App],
    }).compileComponents();
  });

  it('should create the app', () => { ... });
});
```

- Uses `TestBed.configureTestingModule` with `imports` array for standalone component testing.
- Uses `TestBed.createComponent` for component instantiation.
- Uses `fixture.componentInstance` for accessing component instance.
- Uses `fixture.nativeElement` for DOM assertions.
- No mocking pattern demonstrated.

### Missing test infrastructure

**Parser / Simulator critical untested areas**:
- Binary packet parsing logic (`OBD2Parser.parsePacket`) — complex byte-level protocol parsing with no tests.
- IMEI credential validation (`OBD2Parser.validateCredentials`) — Supabase query logic untested.
- Simulator packet builder (`GhostFleetSimulator.buildPacket`) — binary packet construction untested.
- Simulator data generation (`GhostFleetSimulator.generateSimulatedData`) — random scenario logic untested.

**Dashboard untested areas**:
- All guards (`authGuard`, `adminGuard`, `fleetAdminGuard`, `orgOwnerGuard`, `orgAdminGuard`, `onboardingGuard`, `onboardingStepGuard`)
- All services (`AuthService`, `VehicleService`, `TelemetryService`, `AlertService`, `SupabaseService`, `FleetService`, `DeviceService`, `DtcTranslationService`, `OrganizationService`)
- All components (15+ components)
- Alert processing logic (linear regression in `AlertService.predictVoltageIn2Hours`)
- Health score calculation (`VehicleService.calculateHealthScore`)
- DTC translation

**Auth Service untested areas**:
- Token exchange flow
- JWT validation
- Refresh token logic
- Auth0 JWKS verification

### Recommended mocking approach

**For Dashboard services (Jasmine + Karma)**:
```typescript
// Mock SupabaseService
const supabaseMock = {
  client: {
    from: jasmine.createSpy('from').and.returnValue({
      select: jasmine.createSpy('select').and.returnValue({
        eq: jasmine.createSpy('eq').and.returnValue({
          single: jasmine.createSpy('single').and.returnValue(Promise.resolve({ data: null, error: null })),
        }),
      }),
    }),
  },
};

TestBed.configureTestingModule({
  imports: [ComponentUnderTest],
  providers: [
    { provide: SupabaseService, useValue: supabaseMock },
  ],
});
```

**For Parser / Simulator (Jest)**:
```typescript
// Mock Supabase client
jest.mock('@supabase/supabase-js', () => ({
  createClient: jest.fn(() => ({
    from: jest.fn().mockReturnThis(),
    select: jest.fn().mockReturnThis(),
    eq: jest.fn().mockReturnThis(),
    insert: jest.fn().mockReturnValue({ data: null, error: null }),
  })),
}));
```

## Run Commands

```bash
# Dashboard (Karma/Jasmine)
npm test -w @vitalsdrive/dashboard    # Run all tests
ng test                               # Run all tests (from dashboard/)
ng test --watch=false                 # Single run (CI mode)
ng test --code-coverage               # Coverage report

# Parser (Jest)
npm test -w @vitalsdrive/parser       # Run all tests (if jest installed)
npx jest                              # Run all tests (from parser/)
npx jest --coverage                   # Coverage report

# Simulator (Jest)
npm test -w @vitalsdrive/simulator    # Run all tests (if jest installed)
npx jest                              # Run all tests (from simulator/)
```

## Test File Organization (Recommended)

**Dashboard** (following Angular conventions):
- Co-located `*.spec.ts` next to source files
- Pattern: `dashboard.component.ts` -> `dashboard.component.spec.ts`

**Parser / Simulator** (Jest conventions):
- Co-located `*.test.ts` or `*.spec.ts` next to source files
- Pattern: `index.ts` -> `index.test.ts` or `obd2-parser.test.ts`

## Test Types

**Unit tests**: Only approach used. No integration or E2E tests exist.

**Integration tests**: None. Parser-Supabase integration, Dashboard-Realtime integration, Auth Service token flow — all untested.

**E2E tests**: Not configured. No Cypress, Playwright, or Angular Protractor/Cypress setup detected.

---

*Testing analysis: 2026-05-07*
