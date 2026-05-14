# VitalsDrive — Customer Management PRD

**Version:** 1.0 | **Status:** Draft | **Updated:** March 2026

---

## 1. Overview

Customer Management covers everything from the moment a prospect signs up to the full lifecycle of their account — onboarding, fleet configuration, user roles, and account health. As a solo operator, the goal is to make self-service the default so customers can set themselves up without requiring your time for routine tasks.

---

## 2. Scope

| In Scope                                 | Out of Scope                                                          |
| ---------------------------------------- | --------------------------------------------------------------------- |
| Account registration & onboarding flow   | CRM / sales pipeline (use a spreadsheet until you have 20+ customers) |
| Fleet & vehicle setup                    | Customer success automation                                           |
| User roles & permissions                 | SSO / enterprise identity                                             |
| Account settings & profile               | Multi-language support                                                |
| Admin panel (your view of all customers) | API access for customers                                              |

---

## 3. Data Model

### accounts

Top-level customer account (a business).

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Account identifier |
| company_name | TEXT | NOT NULL | Business name |
| email | TEXT | UNIQUE, NOT NULL | Primary billing contact |
| phone | TEXT | | Contact phone number |
| plan_tier | TEXT | DEFAULT 'foundation' | foundation, diagnostic_plus, or predictive |
| status | TEXT | DEFAULT 'trial' | trial, active, suspended, or churned |
| trial_ends_at | TIMESTAMPTZ | | When the 14-day trial expires |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Account creation time |

### users

Users who can log into the dashboard (1 account → many users).

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | User identifier |
| account_id | UUID | FK → accounts | Parent account |
| email | TEXT | UNIQUE, NOT NULL | User email |
| role | TEXT | DEFAULT 'viewer' | owner, manager, or viewer |
| full_name | TEXT | | Display name |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | User creation time |

### vehicles

Vehicles belonging to an account.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Vehicle identifier |
| account_id | UUID | FK → accounts | Parent account |
| display_name | TEXT | NOT NULL | e.g. "Van #3 - Moshe" |
| make, model, year | TEXT/INTEGER | | Vehicle details |
| license_plate | TEXT | | License plate number |
| vin | TEXT | | Auto-populated from OBD on first connection |
| status | TEXT | DEFAULT 'pending' | pending, active, offline, or retired |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Vehicle creation time |

---

## 4. User Roles

| Role        | Capabilities                                                                       |
| ----------- | ---------------------------------------------------------------------------------- |
| **Owner**   | Full access. Manages billing, users, vehicles, and settings. One per account.      |
| **Manager** | Can add/remove vehicles, view all data, acknowledge alerts. Cannot manage billing. |
| **Viewer**  | Read-only. Can view dashboard and alerts but cannot make changes.                  |

> **MVP simplification:** Ship Owner + Viewer only. Manager role can be added when a customer explicitly asks for it.

---

## 5. Onboarding Flow

The onboarding sequence must get a new customer to "first data seen" with zero hand-holding from you.

```
[Sign Up] → [Verify Email] → [Create Account] → [Add First Vehicle]
         → [Pair Device] → [Confirm Data Flowing] → [Dashboard]
```

### Step-by-Step

**Step 1 — Sign Up**

- Email + password via Supabase Auth
- No OAuth for MVP (adds complexity, low ROI at this stage)

**Step 2 — Account Setup**

- Company name, phone number
- Auto-starts a 14-day free trial of Diagnostic+ (gives them the best first impression)

**Step 3 — Add First Vehicle**

- Form: Display name, make, model, year, license plate
- VIN is optional at this stage — it will be auto-populated from OBD data on first connection

**Step 4 — Device Pairing**

- Customer is shown their unique device lookup key (based on the IMEI they enter)
- Step-by-step SMS configuration guide shown in-app:
  ```
  Send this SMS to your device SIM:
  adminip[password] [your-server-host] [port]
  ```
- A "Waiting for device..." live indicator polls Supabase for the first telemetry packet from that IMEI
- On first packet received → green checkmark, vehicle status flips to `active`

**Step 5 — Dashboard**

- Redirected to their live fleet dashboard
- If no data yet, a clear empty state with troubleshooting steps (not a blank screen)

---

## 6. Admin Panel (Your View)

A simple internal-only Angular route (`/admin`, protected by a hardcoded admin user role in Supabase) giving you visibility across all accounts.

| View               | What It Shows                                                                  |
| ------------------ | ------------------------------------------------------------------------------ |
| **Account List**   | All accounts, plan tier, status, vehicle count, created date                   |
| **Account Detail** | Users, vehicles, device IMEI list, last telemetry timestamp                    |
| **Health Monitor** | Accounts where no telemetry has been received in >24h (device offline warning) |
| **Trial Expiry**   | Accounts whose trial ends in <3 days (so you can manually reach out)           |

> This is not a fancy CRM — it's a read-only ops view. Keep it simple.

---

## 7. Account Lifecycle States

```
trial → active (on payment)
trial → churned (trial expired, no payment)
active → suspended (payment failed)
suspended → active (payment resolved)
suspended → churned (no resolution after 7 days)
active → churned (customer cancels)
churned → active (reactivation)
```

When an account is `suspended`, the dashboard remains accessible (read-only) but live telemetry ingestion is paused — the parser checks account status before writing to `telemetry_logs`.

---

## 8. MVP Sequencing

| Priority | Feature                                               | Rationale                               |
| -------- | ----------------------------------------------------- | --------------------------------------- |
| P0       | Sign up, email verify, account creation               | Can't do anything without this          |
| P0       | Add vehicle + device pairing flow                     | Core activation moment                  |
| P0       | Admin account list view                               | You need visibility as operator         |
| P1       | User invite (owner invites another user)              | Needed when a customer has a dispatcher |
| P1       | Account status lifecycle (trial → active → suspended) | Needed before taking payments           |
| P2       | Manager role                                          | Only when a customer asks for it        |
| P2       | Vehicle retire/archive                                | Housekeeping, not urgent                |

---

## 9. Open Questions

1. Should trial accounts require a credit card upfront (reduces churn, may reduce signups) or card-on-file after trial?
2. What happens to historical telemetry data when an account churns — retained for 30 days? Deleted immediately?
3. Self-serve cancellation in-app, or email-only? (Email-only buys you a save attempt conversation.)
