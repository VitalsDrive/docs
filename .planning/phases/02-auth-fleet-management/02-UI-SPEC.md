---
phase: 2
slug: auth-fleet-management
status: approved
shadcn_initialized: false
preset: none
created: 2026-05-12
---

# Phase 2 — UI Design Contract

> Visual and interaction contract for Phase 2: Auth & Fleet Management.
> Design system: VitalsDrive CSS custom properties (dark, warm amber palette).
> Framework: Angular 21 + Angular Material.

---

## Design System

| Property | Value |
|----------|-------|
| Tool | Angular Material 21 (no shadcn — Angular project) |
| Preset | VitalsDrive design tokens (CSS custom properties) |
| Component library | Angular Material (MatDialog, MatTable, MatCard, MatForm) |
| Icon library | Material Symbols / mat-icon |
| Font display | Space Grotesk (Latin), Rubik (Hebrew/RTL) |
| Font body | Inter (Latin), Heebo (Hebrew/RTL) |
| Font mono | JetBrains Mono (VINs, IMEI codes, metrics) |
| Theme | Dark — deep umber backgrounds, warm cream text, amber brand |

---

## Spacing Scale

Sourced from `--vd-space-*` tokens (4px base):

| Token | Value | Usage |
|-------|-------|-------|
| --vd-space-1 | 4px | Icon gaps, chip padding |
| --vd-space-2 | 8px | Compact element spacing, input internal padding |
| --vd-space-3 | 12px | Badge padding, small gaps |
| --vd-space-4 | 16px | Default element spacing, form field gaps |
| --vd-space-6 | 24px | Section padding (--vd-content-pad) |
| --vd-space-8 | 32px | Card internal padding, layout gaps |
| --vd-space-10 | 40px | Section breaks |
| --vd-space-12 | 48px | Major section breaks |
| --vd-space-16 | 64px | Header height (--vd-header-h) |

Exceptions:
- `12px` (--vd-space-3): Badge and chip padding requires tighter density than 16px — part of canonical VitalsDrive token set
- `40px` (--vd-space-10): Section break midpoint between card padding (32px) and section break (48px) — part of canonical VitalsDrive token set

---

## Typography

Collapsed to 4 sizes. All from `--vd-text-*` tokens:

| Role | Token | Size | Weight | Line Height | Font |
|------|-------|------|--------|-------------|------|
| Page title | --vd-text-2xl | 28px | 600 | 1.15 (tight) | Space Grotesk |
| Section/card heading | --vd-text-xl | 22px | 600 | 1.3 (snug) | Inter |
| Body / form input | --vd-text-md | 15px | 400 | 1.5 (base) | Inter |
| Meta / labels / code | --vd-text-sm | 12px | 400 or 600 | 1.5 (base) | Inter or JetBrains Mono |

**Meta/label variants at 12px** (differentiated by weight + case, not size):
- Secondary body: 400, Inter, `--vd-fg-2`
- Eyebrow label: 600, Inter, uppercase, `letter-spacing: 0.08em`, `--vd-fg-3`
- Code/VIN/IMEI: 600, JetBrains Mono, tabular-nums, `--vd-fg-1`

Letter spacing: `--vd-track-tight` (-0.02em) on page title, `--vd-track-eyebrow` (0.08em) on uppercase labels.

---

## Color

| Role | Token | Value | Usage |
|------|-------|-------|-------|
| Dominant (60%) | --vd-bg-base | #0e0a07 | Page background |
| Surface | --vd-bg-surface | #1a130d | Cards, panels |
| Elevated | --vd-bg-elevated | #241a12 | Modals, popovers, hovered cards |
| Sunken | --vd-bg-sunken | #080604 | Input wells |
| Sidebar | --vd-bg-sidebar | #100b07 | Nav sidebar |
| Header | --vd-bg-header | #15100a | Top header bar |
| Primary text | --vd-fg-1 | #f5e6d3 | Headings, primary body |
| Secondary text | --vd-fg-2 | #c4a986 | Body copy, descriptions |
| Muted | --vd-fg-3 | #8a7058 | Labels, placeholders |
| Disabled | --vd-fg-4 | #5a4530 | Disabled text, offline state |
| Brand / Accent (10%) | --vd-brand | #f59e0b | Primary CTA buttons, focus rings, active nav |
| Brand hover | --vd-brand-hover | #d97706 | Button hover state |
| Brand active | --vd-brand-active | #b45309 | Button pressed state |
| Ember accent | --vd-ember | #ea580c | Info badges, secondary highlights |
| Destructive | --vd-critical | #ef4444 | Remove/delete actions, error states |
| Healthy | --vd-healthy | #84cc16 | Active/healthy vehicle status |
| Warning | --vd-warning | #eab308 | Warning vehicle status |
| Offline | --vd-offline | #5a4530 | Offline/inactive vehicles |
| Border default | --vd-border | #3a2a1d | Card borders, dividers |
| Border subtle | --vd-border-subtle | #261b12 | Row dividers |
| Border strong | --vd-border-strong | #503624 | Selected/focus track |

**Split: 60% background (`--vd-bg-*` tokens), 30% surface/elevated (`--vd-bg-surface`, `--vd-bg-elevated`, `--vd-bg-sidebar`, `--vd-bg-header`), 10% brand accent (`--vd-brand` #f59e0b).**

**Brand reserved for:** primary CTA buttons, active navigation item, focus outline/glow, step progress indicator active state, form field focus ring.

---

## Border Radius

| Token | Value | Usage |
|-------|-------|-------|
| --vd-radius-xs | 4px | Chips, badges, status dots |
| --vd-radius-sm | 6px | Inputs, text fields |
| --vd-radius-md | 10px | Buttons, list rows, snackbar |
| --vd-radius-lg | 16px | Cards (vehicle cards, onboarding steps) |
| --vd-radius-xl | 20px | Modals (MatDialog panels) |
| --vd-radius-full | 9999px | Avatar circles, step number badges |

---

## Motion

| Token | Value | Usage |
|-------|-------|-------|
| --vd-ease-out | cubic-bezier(.22,1,.36,1) | Entrances (slide in, expand) |
| --vd-ease-in-out | cubic-bezier(.65,0,.35,1) | State transitions |
| --vd-dur-fast | 120ms | Hover states, button feedback |
| --vd-dur-base | 220ms | Route transitions, card expand |
| --vd-dur-slow | 400ms | Modal open/close, skeleton fade |

Skeleton shimmer animation: 1.4s loop, ease-in-out, gradient sweep from `--vd-bg-surface` to `--vd-bg-elevated`.

---

## Shadows & Glow

| Token | Usage |
|-------|-------|
| --vd-shadow-sm | Inline badges, chips |
| --vd-shadow-card | Default vehicle cards |
| --vd-shadow-elevated | Hovered vehicle cards |
| --vd-shadow-modal | MatDialog panels |
| --vd-glow-amber | Active/selected card, focused input |
| --vd-glow-critical | Critical alert card border |

---

## Layout

| Constant | Token | Value |
|----------|-------|-------|
| Sidebar width | --vd-sidebar-w | 240px |
| Sidebar collapsed | --vd-sidebar-w-collapsed | 64px |
| Header height | --vd-header-h | 64px |
| Content padding | --vd-content-pad | 24px |

---

## Screens Delivered by Phase 2

| Screen | Route | Notes |
|--------|-------|-------|
| Auth callback / redirect | /auth/callback | Handled by Auth0 SDK — no custom UI |
| Onboarding — Create Org | /onboarding/org | Step 1 of 3 |
| Onboarding — Create Fleet | /onboarding/fleet | Step 2 of 3 |
| Onboarding — Add Vehicle | /onboarding/vehicle | Step 3 of 3 (skippable) |
| Fleet Management | /fleet-management | Vehicle cards + fleet switcher |
| Invite Members | /settings/invite | Invite link generator |

---

## Onboarding Flow

**Step badge:** Pill badge — `--vd-radius-full`, `--vd-brand` fill for active, `--vd-bg-elevated` for pending, `--vd-fg-3` for upcoming. Text: "Step N of 3".

**Step labels:**
- Step 1 of 3: "Create Your Organization"
- Step 2 of 3: "Create Your First Fleet"
- Step 3 of 3: "Add Your First Vehicle" — skippable

**Skip link (Step 3):** "Skip — add vehicles from Fleet Management" — `--vd-fg-3`, underline on hover, positioned below primary CTA.

**Card container:** `--vd-bg-surface`, `--vd-radius-lg`, `--vd-shadow-card`, max-width 480px, centered. Padding: `--vd-space-8` (32px).

**Progress connector:** Horizontal line between step badges, `--vd-border` for incomplete, `--vd-brand` for completed.

---

## Fleet Management Page

**Layout:** Card grid — CSS grid, `repeat(auto-fill, minmax(280px, 1fr))`, gap `--vd-space-6`.

**Vehicle card:**
- Background: `--vd-bg-surface`
- Border: `1px solid --vd-border`
- Radius: `--vd-radius-lg`
- Shadow: `--vd-shadow-card` (default), `--vd-shadow-elevated` on hover
- Hover: border → `--vd-border-strong`, shadow elevated, `transition: 220ms ease-out`
- Status border-left: 4px solid — healthy=`--vd-healthy`, warning=`--vd-warning`, critical=`--vd-critical`, offline=`--vd-offline`
- Content: Vehicle nickname (h3, Space Grotesk 600), make/model/year (`--vd-fg-2`), plate badge (mono, `--vd-bg-elevated`, `--vd-radius-xs`), fleet label (eyebrow), action buttons (edit icon, remove icon)

**Fleet switcher:** `MatSelect` or tab row above grid — filters cards by fleet. "All Fleets" as default option.

**Empty state (no vehicles):**
- Heading: "No vehicles yet"
- Body: "Add your first vehicle to start monitoring fleet health."
- CTA: "Add Vehicle" button (primary amber)

**Add vehicle button:** Fixed at top-right of page header, amber primary button, "+ Add Vehicle" label.

---

## Vehicle Removal Dialog (MatDialog)

```
Title: "Remove [Vehicle Nickname]?"
Body:  "This vehicle will be deactivated and removed from your fleet.
        Historical telemetry data is preserved."
Actions:
  - [Cancel]  — outlined/ghost, --vd-fg-2
  - [Remove Vehicle]  — filled, --vd-critical (#ef4444), --vd-fg-inverse text
```

Dialog panel: `--vd-bg-elevated`, `--vd-radius-xl`, `--vd-shadow-modal`, max-width 400px.

---

## App Init Loading (Skeleton Shimmer)

Displayed while `/auth/me` resolves. Layout mirrors the authenticated shell:

- Sidebar: shimmer blocks for nav items (3 rows, `--vd-radius-md`, 160px wide)
- Header: shimmer block 200px wide right side
- Content: 2 rows of 3 card shimmer placeholders (`--vd-radius-lg`, full card dimensions)

Shimmer color: gradient sweep `--vd-bg-surface` → `--vd-bg-elevated` → `--vd-bg-surface`, 1.4s loop.

Fade-out: `opacity: 0, transition: 400ms (--vd-dur-slow)` when `/auth/me` resolves.

---

## Invite Flow

**Invite type selector:** `MatButtonToggle` group above link display. Options: "Single-use link" / "Multi-use link (7 days)". Label: "Invite type" (eyebrow style, `--vd-fg-3`). Default: "Single-use link".

**Invite link display:** Read-only `MatFormField` with monospace font, full-width. Button: "Copy Invite Link" — outlined variant, `--vd-brand` border and text.

**Copy feedback:** Snackbar "Link copied to clipboard" — `--vd-bg-elevated`, `--vd-fg-1`, `--vd-radius-md`, 2s auto-dismiss.

**Invite page empty state:**
- Heading: "Invite your team"
- Body: "Share this link to give teammates access to your fleet."

---

## Copywriting Contract

| Element | Copy |
|---------|------|
| Onboarding Step 1 CTA | "Create Organization" |
| Onboarding Step 2 CTA | "Create Fleet" |
| Onboarding Step 3 CTA | "Add Vehicle" |
| Onboarding Step 3 skip link | "Skip — add vehicles from Fleet Management" |
| Fleet page empty state heading | "No vehicles yet" |
| Fleet page empty state body | "Add your first vehicle to start monitoring fleet health." |
| Fleet page primary CTA | "+ Add Vehicle" |
| Vehicle removal dialog title | "Remove [Vehicle Nickname]?" |
| Vehicle removal dialog body | "This vehicle will be deactivated and removed from your fleet. Historical telemetry data is preserved." |
| Vehicle removal confirm button | "Remove Vehicle" |
| Invite type selector label | "Invite type" |
| Invite type option 1 | "Single-use link" |
| Invite type option 2 | "Multi-use link (7 days)" |
| Invite copy button | "Copy Invite Link" |
| Vehicle card edit button (aria-label) | "Edit [Vehicle Nickname]" |
| Vehicle card remove button (aria-label) | "Remove [Vehicle Nickname]" |
| Copy success snackbar | "Link copied to clipboard" |
| Auth loading (aria-label) | "Loading your fleet dashboard…" |
| Unauthenticated redirect (aria-label) | "Redirecting to login…" |
| Org name placeholder | "e.g. Acme Logistics" |
| Fleet name placeholder | "e.g. Main Fleet" |
| Vehicle nickname placeholder | "e.g. Truck 01" |

---

## Form Validation

All forms use Angular Material `MatFormField` with `matError`. Error messages appear inline below the field on blur or submit attempt.

| Field | Validation | Error copy |
|-------|-----------|------------|
| Org name | Required, 2–80 chars | "Organization name is required" / "Must be 2–80 characters" |
| Fleet name | Required, 2–60 chars | "Fleet name is required" |
| Vehicle nickname | Required, 2–40 chars | "Nickname is required" |
| Vehicle make/model | Optional | — |
| Vehicle year | Optional, 4-digit int | "Enter a valid year (e.g. 2020)" |
| Vehicle plate | Optional | — |

Input focus ring: `--vd-glow-amber` (`box-shadow: 0 0 0 1px rgba(245,158,11,0.4), 0 0 18px rgba(245,158,11,0.25)`).

---

## Registry Safety

| Registry | Components Used | Safety Gate |
|----------|----------------|-------------|
| Angular Material | MatDialog, MatCard, MatFormField, MatInput, MatSelect, MatButton, MatIconButton, MatSnackBar, MatProgressSpinner | Not required — official library |
| Google Fonts | Space Grotesk, Inter, JetBrains Mono, Rubik, Heebo | Not required — CDN import |
| Auth0 Angular SDK | @auth0/auth0-angular | Not required — official SDK |

No third-party component registries used.

---

## Checker Sign-Off

- [x] Dimension 1 Copywriting: PASS
- [x] Dimension 2 Visuals: PASS
- [x] Dimension 3 Color: PASS
- [x] Dimension 4 Typography: PASS
- [x] Dimension 5 Spacing: PASS
- [x] Dimension 6 Registry Safety: PASS

**Approval:** approved 2026-05-12
