# VitalsDrive — Billing & Subscriptions PRD

**Version:** 1.1 | **Status:** Draft | **Updated:** April 2026

**Changelog (v1.1):** Pricing updated to ILS (₪89 / ₪149 / ₪199 per vehicle/month). Trial now requires credit card for device deposit. Currency decision resolved: ILS at launch. COGS breakdown added. Device deposit handling added. Open questions updated.

---

## 1. Overview

Billing covers subscription management, invoicing, payment collection, and plan changes. Building this custom as a solo founder is the highest-risk decision in the entire product — billing bugs directly cost you money and destroy customer trust. The approach here minimizes custom billing logic by delegating all payment processing and invoice generation to Stripe, while keeping subscription state in your own Supabase database as the source of truth for feature access.

---

## 2. Core Principle: Stripe Handles Money, Supabase Handles Access

Never gate features based on Stripe data directly. The flow is:

```
Stripe event (webhook) → Update accounts.status in Supabase → Angular reads Supabase
```

This means if Stripe is slow or your webhook fails, your app still works — you just have a temporary sync gap to resolve. Feature access is always read from your own DB, never from a live Stripe API call on the hot path.

---

## 3. Pricing Model

Billing is **per vehicle, per month**. All prices are in ILS (Israeli Shekel). An account with 5 vehicles on Diagnostic+ pays ₪149 × 5 = ₪745/mo.

| Tier        | Price / Vehicle / Month | Gross Margin (est.) |
| ----------- | ---------------------- | ------------------- |
| Foundation  | ₪89                    | ~58%                |
| Diagnostic+ | ₪149                   | ~74%                |
| Predictive  | ₪199                   | ~82%                |

### Estimated COGS (per vehicle / month)

| Cost Category | Monthly Cost | Notes |
|---|---|---|
| SIM + data | ~$5.00 | Things Mobile, Zone 1+ (Israel), ~30 MB/month |
| Device depreciation | ~$2.60 | Teltonika FMC003 @ $63, ~24mo avg lifetime |
| Infrastructure share | ~$2.75 | Supabase Pro + Railway Pro, ~20 vehicles |
| **Total COGS** | **~$10.35** | |

### Billing Terms

- Annual billing option: 2 months free (10/12 effective monthly rate)
- Trial: 14 days free, Diagnostic+ tier. **Credit card required at signup** to cover the ₪150 device deposit.

---

## 4. Data Model

### subscriptions

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Subscription identifier |
| account_id | UUID | FK → accounts, UNIQUE | Customer account |
| stripe_customer_id | TEXT | UNIQUE | Stripe customer object |
| stripe_sub_id | TEXT | UNIQUE | Stripe subscription object |
| plan_tier | TEXT | NOT NULL | foundation, diagnostic_plus, or predictive |
| billing_cycle | TEXT | DEFAULT 'monthly' | monthly or annual |
| vehicle_count | INTEGER | NOT NULL, DEFAULT 1 | Active billable vehicles |
| status | TEXT | NOT NULL | trialing, active, past_due, or canceled |
| current_period_end | TIMESTAMPTZ | | When current billing period ends |
| cancel_at_period_end | BOOLEAN | DEFAULT false | Queued cancellation |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Subscription creation time |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | Last update time |

### invoices

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Invoice identifier |
| account_id | UUID | FK → accounts | Customer account |
| stripe_invoice_id | TEXT | UNIQUE | Stripe invoice reference |
| amount_ils | INTEGER | | Amount in agorot (×100), ILS |
| status | TEXT | | draft, open, paid, uncollectible, or void |
| period_start | TIMESTAMPTZ | | Billing period start |
| period_end | TIMESTAMPTZ | | Billing period end |
| paid_at | TIMESTAMPTZ | | When payment received |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Invoice creation time |

### deposits

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Deposit record identifier |
| account_id | UUID | FK → accounts, UNIQUE | Customer account |
| stripe_payment_intent_id | TEXT | UNIQUE | Stripe PaymentIntent for deposit |
| amount_ils | INTEGER | NOT NULL | Deposit amount in agorot (₪150 × 100) |
| status | TEXT | NOT NULL | charged, refunded, or forfeited |
| charged_at | TIMESTAMPTZ | | When deposit was collected |
| refunded_at | TIMESTAMPTZ | | When deposit was refunded |
| refund_reason | TEXT | | Why refunded (device return, cancellation) |

**Deposit state machine:**
```
charged → refunded (device returned within 30 days of cancellation)
charged → forfeited (device not returned, deposit kept)
```

---

## 5. Stripe Configuration

### Products & Prices

Set up in Stripe dashboard (not in code — prices rarely change):

| Product | Price |
|---|---|
| VitalsDrive Foundation | ILS 89/vehicle/month (recurring, per-seat by vehicle_count) |
| VitalsDrive Diagnostic+ | ILS 149/vehicle/month |
| VitalsDrive Predictive | ILS 199/vehicle/month |

Use Stripe's **per-seat pricing** model where quantity = number of active vehicles on the account.

### Currency

Billing is in **ILS (Israeli Shekel) from day one**. Israeli SMB customers and their accountants work in shekels and expect VAT invoices (חשבונית מס) in local currency. Stripe supports ILS natively.

### Deposit Handling

A ₪150 refundable device deposit is collected at signup via a separate Stripe PaymentIntent (not a subscription). This is stored in the `deposits` table and refunded on device return within 30 days of cancellation. See Section 4 for the full deposits state machine.

---

## 6. Key Billing Flows

### 6.1 Trial → Paid Conversion

```
Day 0:   Account created → trial starts, ₪150 deposit charged, Stripe customer created
Day 11:  In-app banner: "3 days left on your trial. Confirm your plan."
Day 14:  Trial ends
         If confirmed → Stripe subscription created → status: active
         If not       → account status: suspended (read-only mode)
Day 21:  If still suspended → send final warning email
Day 28:  If still suspended → status: churned, telemetry ingestion stopped
```

### 6.2 Adding a Vehicle (Seat Increase)

When a customer activates a new vehicle (device paired successfully):

1. `vehicles` table gains a new `active` row
2. Backend recalculates `vehicle_count` for the account
3. Stripe subscription quantity is updated via API with the new count
4. Stripe prorates the charge automatically for mid-cycle additions

### 6.3 Removing a Vehicle

Customer retires a vehicle (stolen, sold, etc.):

- Vehicle status set to `retired`
- `vehicle_count` decremented
- Stripe subscription quantity updated
- Proration applied as a credit toward next invoice

### 6.4 Plan Upgrade/Downgrade

- Upgrade (e.g. Foundation → Diagnostic+): Takes effect immediately, prorated
- Downgrade: Takes effect at end of current billing period (`cancel_at_period_end` pattern on old price, new price queued)

### 6.5 Payment Failure

Stripe handles retries automatically (Smart Retries). Webhook events to handle:

| Stripe Event | Action |
|---|---|
| `invoice.payment_failed` | Set account status → `past_due`, send payment failure email |
| `invoice.payment_succeeded` | Set account status → `active` |
| `customer.subscription.deleted` | Set account status → `suspended` |

---

## 7. Webhook Handler Requirements

The webhook endpoint must:

- Accept POST requests from Stripe at `/webhooks/stripe`
- Validate the `stripe-signature` header using `STRIPE_WEBHOOK_SECRET`
- Handle the following events:
  - `invoice.payment_succeeded` → set account status to `active`
  - `invoice.payment_failed` → set account status to `past_due`, send payment failure email
  - `customer.subscription.updated` → sync subscription state
  - `customer.subscription.deleted` → set account status to `suspended`

> **Critical:** Always return `200` to Stripe immediately, then process async. Never do slow DB operations before responding or Stripe will retry the webhook.

---

## 8. Customer-Facing Billing UI (Angular)

A `/billing` route visible to account Owners only.

| Section             | Content                                                                                   |
| ------------------- | ----------------------------------------------------------------------------------------- |
| **Current Plan**    | Tier name, price per vehicle (ILS), vehicle count, monthly total                          |
| **Next Invoice**    | Amount due (ILS + VAT), date, breakdown per vehicle                                       |
| **Invoice History** | List of past invoices with download link (Stripe-hosted PDF)                              |
| **Payment Method**  | Card on file (last 4 digits, expiry) with "Update Card" button → Stripe Customer Portal   |
| **Plan Change**     | Upgrade/downgrade selector with pricing preview in ILS                                    |
| **Cancel**          | "Cancel at end of period" option — shows confirmation dialog, does NOT cancel immediately |

> Use **Stripe Customer Portal** for payment method management and invoice downloads. It's a hosted Stripe UI you redirect to — zero custom code for those flows.

---

## 9. Israeli Tax Considerations

Israel requires VAT (מע"מ) at 18% on B2B SaaS sales. This is not optional.

- Configure Stripe Tax to collect and remit Israeli VAT automatically
- Stripe will add VAT to invoices for Israeli customers based on their billing address
- You'll need to register for VAT with the Israeli Tax Authority (רשות המסים) once revenue exceeds the threshold (~₪120,000/year)
- **MVP shortcut:** For the first few pilot customers, handle VAT manually. Automate once you have 5+ paying accounts.

---

## 10. Infrastructure Cost Reference

Monthly platform costs (fixed, not per-vehicle):

| Service | Plan | Monthly Cost (USD) |
|---|---|---|
| Supabase | Pro (required — no project pausing) | $25 |
| Railway | Pro (TCP server 24/7 + webhook service) | ~$20–35 |
| Things Mobile Portal | Flat per account | $3 |
| **Total Infrastructure** | | **~$48–63/mo** |

SIM cost per vehicle (Things Mobile, Israel = Zone 1+ at $0.10/MB):
- ~30 MB/month per vehicle (30s update interval, ~350 bytes/packet) = $3.00 data
- $2.00 monthly SIM fee
- Note: Hot Mobile (IL) charges in 10kB increments — consider packet aggregation to reduce effective MB usage
- **Total SIM: ~$5/vehicle/month**

---

## 11. MVP Sequencing

| Priority | Feature                                                     | Rationale                                           |
| -------- | ----------------------------------------------------------- | --------------------------------------------------- |
| P0       | Stripe customer + subscription creation on trial conversion | Can't take money without this                       |
| P0       | Webhook handler for payment success/failure                 | Must keep account status in sync                    |
| P0       | Basic billing page (current plan, next invoice)             | Customers need to see what they're paying           |
| P1       | Vehicle count sync to Stripe quantity                       | Needed for accurate billing                         |
| P1       | Stripe Customer Portal integration                          | Handles card updates and invoice downloads for free |
| P1       | Plan upgrade flow                                           | Revenue expansion                                   |
| P2       | Annual billing option                                       | Nice to have, reduces churn                         |
| P2       | VAT automation via Stripe Tax                               | Manual until you have volume                        |

---

## 11. Open Questions

1. Bill in USD or ILS at launch?
2. Credit card required at trial start, or only at conversion? (No-card trials have higher signup rate but lower conversion rate.)
3. What's the grace period between payment failure and service suspension — 7 days? 14 days?
4. Will you offer a hardware deposit or rental model that needs to be reflected in billing?
