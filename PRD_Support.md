# VitalsDrive — Support & Ticketing PRD

**Version:** 1.0 | **Status:** Draft | **Updated:** March 2026

---

## 1. Overview

Support & Ticketing covers how customers report problems, how you track and resolve them, and how you maintain visibility into recurring issues. As a solo founder, the primary design constraint is that **support must not consume your engineering time**. The system needs to surface the right context automatically so you spend 2 minutes resolving a ticket, not 20 minutes reconstructing what the customer's device was doing.

---

## 2. Support Categories

Understanding the ticket types upfront shapes the entire system design. For VitalsDrive, almost every support issue will fall into one of five categories:

| Category             | Example                                | Likely Root Cause                                            |
| -------------------- | -------------------------------------- | ------------------------------------------------------------ |
| **Device Offline**   | "My van hasn't updated in 2 hours"     | SIM connectivity, device power loss, wrong server IP         |
| **Missing Data**     | "I can see GPS but no engine codes"    | Vehicle protocol mismatch (CAN vs K-Line), PID not supported |
| **Alert Not Firing** | "My battery died but I got no warning" | Threshold config, missed telemetry packet, alert logic bug   |
| **Dashboard/UI**     | "The map isn't loading"                | Frontend bug, browser issue                                  |
| **Billing**          | "I was charged twice"                  | Stripe sync issue, vehicle count mismatch                    |

This taxonomy matters because each category has a different resolution path. Device Offline tickets need telemetry diagnostics; Billing tickets need Stripe lookups. Your ticket system should route and surface context accordingly.

---

## 3. Data Model

### tickets

Support ticket records.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Ticket identifier |
| account_id | UUID | FK → accounts | Customer account |
| vehicle_id | UUID | FK → vehicles, nullable | Related vehicle (not all tickets are vehicle-specific) |
| submitted_by | UUID | FK → users | User who submitted |
| category | TEXT | | device_offline, missing_data, alert, dashboard, billing, or other |
| subject | TEXT | NOT NULL | Ticket subject line |
| status | TEXT | DEFAULT 'open' | open, in_progress, waiting_customer, resolved, or closed |
| priority | TEXT | DEFAULT 'normal' | low, normal, high, or critical |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Ticket creation time |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | Last update time |
| resolved_at | TIMESTAMPTZ | | When ticket was resolved |

### ticket_messages

Conversation thread within a ticket.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Message identifier |
| ticket_id | UUID | FK → tickets | Parent ticket |
| author_id | UUID | FK → users, nullable | Null = system message |
| author_role | TEXT | | customer, support, or system |
| body | TEXT | NOT NULL | Message content |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Message time |

### ticket_diagnostics

Auto-captured device/account state snapshot at ticket submission.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Diagnostic record identifier |
| ticket_id | UUID | FK → tickets | Related ticket |
| last_telemetry_at | TIMESTAMPTZ | | Last telemetry timestamp for selected vehicle |
| last_voltage | FLOAT | | Last known battery voltage |
| last_temp | FLOAT | | Last known coolant temperature |
| device_imei | TEXT | | Device IMEI at time of ticket |
| sim_iccid | TEXT | | SIM ICCID at time of ticket |
| account_status | TEXT | | Account status at time of ticket |
| vehicle_status | TEXT | | Vehicle status at time of ticket |
| snapshot_at | TIMESTAMPTZ | DEFAULT NOW() | When snapshot was captured |

---

## 4. Ticket Submission Flow (Customer Side)

### In-App Support Widget

Every page in the Angular dashboard has a persistent "?" help button in the bottom-right corner. Clicking it opens a slide-over panel with:

1. **Category selector** — dropdown matching the 5 categories above
2. **Vehicle selector** — if the issue is vehicle-specific (pre-populated if they're on a vehicle page)
3. **Subject + description** — free text
4. **Submit** button

On submission:

- Ticket row created in `tickets`
- System **automatically captures a diagnostic snapshot** into `ticket_diagnostics`:
  - Last telemetry timestamp for the selected vehicle
  - Last known voltage, coolant temp
  - Current account status
  - Device IMEI and SIM details
- Customer receives an email confirmation with ticket ID
- You receive an email/push notification

> **This diagnostic snapshot is the most valuable part of the system.** When you open a "device offline" ticket, you already know when it last sent data, what the last readings were, and whether it's an account/billing issue — before you've read the customer's message.

### Email-to-Ticket (Fallback)

Some customers will just email you directly. For MVP, handle these manually by creating tickets on their behalf from the admin panel. A proper email ingestion parser (e.g. parsing a support@ mailbox) is a Phase 2 feature.

---

## 5. Ticket Management (Your Admin View)

A `/admin/tickets` route in Angular.

### Ticket List View

| Column        | Detail                                |
| ------------- | ------------------------------------- |
| ID            | Short ticket reference (e.g. VD-0042) |
| Account       | Company name + plan tier              |
| Vehicle       | Vehicle display name (if applicable)  |
| Category      | Color-coded pill                      |
| Subject       | First 60 chars                        |
| Status        | Color-coded status badge              |
| Last Activity | Relative time (e.g. "2h ago")         |
| Priority      | Flag if high/critical                 |

Default sort: open tickets, newest first. Filter by status, category, account.

### Ticket Detail View

```
┌─────────────────────────────────────────────────────┐
│ VD-0042 · Device Offline · HIGH                      │
│ Moshe Delivery Ltd · Van #3 · Opened 3h ago          │
├─────────────────────────────────────────────────────┤
│ DIAGNOSTIC SNAPSHOT (auto-captured)                  │
│ Last telemetry:  3h 14m ago                         │
│ Last voltage:    12.1V                               │
│ Last temp:       88°C                                │
│ Account status:  active                              │
│ Device IMEI:     356307041234                        │
├─────────────────────────────────────────────────────┤
│ MESSAGES                                             │
│ [Customer] "The van stopped showing on the map..."   │
│ [You] "Hi Moshe, I can see the device last..."       │
├─────────────────────────────────────────────────────┤
│ [Reply box]                           [Resolve] [↑]  │
└─────────────────────────────────────────────────────┘
```

The diagnostic snapshot is shown inline at the top of every ticket — not buried in a separate tab.

---

## 6. Automated Proactive Alerts (Support Deflection)

The best support ticket is one that never gets submitted. Set up automated outreach for the most common issues before the customer notices:

| Trigger                                                      | Automated Action                                                                                    |
| ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| Device sends no telemetry for >2 hours during business hours | Email to account owner: "We noticed Van #3 hasn't reported in 2 hours. Here's how to troubleshoot." |
| Battery voltage drops below 11.8V (critical threshold)       | In-app alert + email: "Van #3 battery is critically low."                                           |
| DTC code appears for first time                              | In-app notification + email with plain-English explanation                                          |
| Account trial ends in 3 days                                 | Reminder email                                                                                      |
| Invoice payment fails                                        | Immediate email with link to update payment method                                                  |

These are implemented as Supabase Edge Functions (cron-style) or as triggers on the `telemetry_logs` table — no separate infrastructure needed.

---

## 7. SLA Targets (Solo Founder Reality)

Don't commit to SLAs you can't keep. Be honest with customers upfront.

| Priority     | Definition                                 | Target Response | Target Resolution |
| ------------ | ------------------------------------------ | --------------- | ----------------- |
| **Critical** | Fleet-wide data loss, all vehicles offline | 2 hours         | 8 hours           |
| **High**     | Single vehicle offline >4 hours            | 4 hours         | 24 hours          |
| **Normal**   | Missing data, UI bugs                      | 1 business day  | 3 business days   |
| **Low**      | Feature questions, general feedback        | 2 business days | Best effort       |

> As a solo founder, "business hours" is whatever your actual availability is. Set the expectation in your onboarding email. Israeli customers generally accept same-day response if you set that expectation clearly.

---

## 8. Knowledge Base (Self-Service Deflection)

Before a customer submits a ticket, show them relevant articles from a simple knowledge base. This is a static set of markdown files rendered in Angular — no CMS needed for MVP.

Priority articles to write before launch:

1. How to configure your device server IP via SMS
2. Why is my vehicle showing offline?
3. What do OBD error codes mean? (Link to OBD code lookup)
4. How to add a new vehicle
5. How to update your payment method
6. Supported vehicle list (CAN bus vs K-Line compatibility)

---

## 9. MVP Sequencing

| Priority | Feature                                              | Rationale                                    |
| -------- | ---------------------------------------------------- | -------------------------------------------- |
| P0       | Ticket submission form (in-app)                      | Customers need a way to reach you            |
| P0       | Diagnostic snapshot on submission                    | Saves you 80% of the back-and-forth          |
| P0       | Email notification to you on new ticket              | You need to know immediately                 |
| P0       | Admin ticket list + detail view                      | You need to see and respond to tickets       |
| P1       | Automated device offline detection + proactive email | Deflects the most common ticket type         |
| P1       | Customer email confirmation on submit                | Basic professionalism                        |
| P2       | Knowledge base articles (static)                     | Self-service deflection                      |
| P2       | Ticket status visible to customer in-app             | Reduces "any update?" follow-ups             |
| P3       | Email-to-ticket ingestion                            | Only needed once email volume is significant |

---

## 10. Open Questions

1. Will you handle support in Hebrew, English, or both? (Affects knowledge base and email templates)
2. Do you want a public status page (e.g. status.vitalsdrive.com) so customers can check if there's a known outage before submitting a ticket?
3. At what point does ticket volume justify looking at a dedicated tool (linear, Intercom, etc.) even though the plan is to build custom?
