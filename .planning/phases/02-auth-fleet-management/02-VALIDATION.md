---
phase: 2
slug: auth-fleet-management
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-05-12
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest (auth-service), Karma/Jasmine (dashboard) |
| **Config file** | `packages/auth-service/jest.config.*` / `packages/dashboard/karma.conf.js` |
| **Quick run command** | `npm run test` in each package |
| **Full suite command** | `npm run test` at workspace root |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test` in the affected package
- **After every plan wave:** Run `npm run test` at workspace root
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 2-01-01 | 01 | 1 | AUTH-03 | — | Exchange returns signed JWT, not plaintext creds | Unit | `npm run test` in `packages/auth-service` | ❌ W0 | ⬜ pending |
| 2-01-02 | 01 | 1 | AUTH-03 | — | User auto-provisioned in Supabase on first exchange | Unit | `npm run test` in `packages/auth-service` | ❌ W0 | ⬜ pending |
| 2-02-01 | 02 | 2 | AUTH-04 | — | Unauthenticated route access redirects to /login | Unit | `npm run test` in `packages/dashboard` | ❌ W0 | ⬜ pending |
| 2-02-02 | 02 | 2 | AUTH-01 | — | `isAuthenticated` signal correct (non-Promise) | Unit | `npm run test` in `packages/dashboard` | ❌ W0 | ⬜ pending |
| 2-02-03 | 02 | 2 | AUTH-02 | — | Token persists across refresh | Manual | Browser: reload page, check localStorage | — | ⬜ pending |
| 2-04-01 | 04 | 3 | AUTH-04 | T-02-04-01 | VehicleService soft-delete sets status='inactive' (not hard-delete) | Unit | `npm run test -- --watch=false` in `packages/dashboard` | Created by Plan 04 Task 1 | ⬜ pending |
| 2-04-02 | 04 | 3 | AUTH-04 | T-02-04-01 | /fleet-management route compiles and is accessible to authenticated users | Build | `npm run build -- --configuration=development` in `packages/dashboard` | — | ⬜ pending |
| 2-05-01 | 05 | 3 | AUTH-01 | T-02-05-01 | Invite token inserted into org_invites via Supabase; clipboard copy works | Build | `npm run build -- --configuration=development` in `packages/dashboard` | — | ⬜ pending |
| 2-05-02 | 05 | 3 | AUTH-02 | T-02-05-03 | Skip link navigates to /dashboard without creating a vehicle | Build | `npm run build -- --configuration=development` in `packages/dashboard` | — | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

> Wave 0 spec files are created during plan execution as TDD RED tasks (before implementation). They do not exist pre-execution — this is expected.

- [ ] `packages/auth-service/src/auth/auth.service.spec.ts` — stubs for AUTH-03 exchange + user provisioning — **created by Plan 02, Task 1**
- [ ] `packages/dashboard/src/app/core/guards/auth.guard.spec.ts` — AUTH-04 guard redirect behavior — **created by Plan 03, Task 1**
- [ ] `packages/dashboard/src/app/core/services/auth.service.spec.ts` — `isAuthenticated` signal correctness — **created by Plan 03, Task 1**

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Auth0 login redirect for unauthenticated user | AUTH-01 | OAuth redirect flow cannot be automated in unit tests | Open browser, navigate to `/dashboard`, verify redirect to Auth0 login page |
| Token persists after browser refresh | AUTH-02 | Requires real browser localStorage + page reload | Login, refresh page, verify user remains logged in without re-auth prompt |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
