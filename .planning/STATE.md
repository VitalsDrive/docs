---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: in_progress
last_updated: "2026-05-13T10:10:00.000Z"
progress:
  total_phases: 5
  completed_phases: 1
  total_plans: 4
  completed_plans: 4
  percent: 100
current_phase: 02-auth-fleet-management
current_plan: "02-06"
stopped_at: "Plan 02-05 complete. OnboardingVehicleComponent (Step 3 of 3, skippable) at /onboarding/vehicle. InviteComponent at /settings/invite with type selector, role selector, org_invites insert, clipboard copy. Both routes registered in app.routes.ts. Build passes. Ready for 02-06."
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-05-11)

**Core value:** Prevent expensive vehicle failures — DTC codes, dead batteries, and overheating engines — before they cause breakdowns. Fleet Health, Simplified.
**Current focus:** Phase 2: Auth & Fleet Management (in progress — plan 02-02 complete)
**Last completed:** Phase 2-05 — Onboarding Step 3 vehicle form + invite link generator
**Decisions:** FMC003 IO IDs corrected (7040/7044/7045/7059/7038); TelemetryWriter uses injected SupabaseClient; jest.config.js rootDir override for tests/; D-07: dtc_codes=[] deferred to Phase 4; CRC-16/IBM (0xA001 reflected) replaces CCITT 0x1021; known test vector [0x8E,0x00,0x01]=0xEBA1 (plan stated 0x4B37 was wrong); Phase 2-01: auth.jwt()->>'sub' not auth.uid() for Auth0 RLS; ALL RLS policies updated across all tables; organizations/fleets/fleet_members owner/user columns converted to TEXT; vehicles.make/model made nullable; Auth0 JWT provider configured in Supabase; D-06: JWT payload = {sub,email} only — orgId/roles removed from auth-service; D-04: user auto-provisioned in users table on first /auth/exchange call; D-13: allowlistGuard removed — onboardingGuard queries real Supabase org+fleet state; toSignal(auth0.isAuthenticated$) replaces broken computed(firstValueFrom) — real boolean signal; Jest+ts-jest added as dashboard test runner (no Karma/Vitest was configured); /auth/me validation on app init validates stored JWT before Supabase queries (D-07); 02-04: VehicleService CRUD added to existing telemetry service (not replaced); vehicle.model.ts nickname/make/model/vin made nullable + CreateVehicleDto added; deleteVehicle soft-deletes via status:inactive

## Active Phase

**Phase:** 2 (Auth & Fleet Management) — IN PROGRESS
**Status:** Plan 02-05 complete. OnboardingVehicleComponent at /onboarding/vehicle (Step 3 of 3, skippable). InviteComponent at /settings/invite: type selector, role selector, crypto.randomUUID token, org_invites Supabase insert, clipboard copy with snackbar. Build passes. Ready for plan 02-06.
**Resume:** `.planning/phases/02-auth-fleet-management/02-06-PLAN.md`
**Roadmap:** .planning/ROADMAP.md

## Sub-Repositories

This project uses 5 separate git repos:

- `docs/` — Documentation
- `packages/dashboard/` — Angular dashboard
- `packages/parser/` — TCP ingestion server
- `packages/simulator/` — Ghost fleet simulator
- `packages/supabase/` — Database schema & migrations
- `packages/auth-service/` — JWT exchange service (no .git repo initialized)

## Notes

- Brownfield project — core infrastructure already exists
- Config loaded from .planning/config.json
- Codebase analysis available in .planning/codebase/

---
*Initialized: 2026-05-11*
