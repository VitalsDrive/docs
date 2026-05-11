# VitalsDrive — Hardware Lifecycle PRD

**Version:** 1.0 | **Status:** Draft | **Updated:** March 2026

---

## 1. Overview

Hardware Lifecycle covers every physical touchpoint with the OBD2 device — from initial procurement and provisioning, through customer shipping and installation, to replacement and returns. This is the most operationally manual part of VitalsDrive and the one most likely to create customer friction if not systematized early. A device that isn't configured correctly will never send data, and that's the worst possible first impression.

---

## 2. Hardware Lifecycle Stages

```
[Procure] → [Receive & Inspect] → [Provision] → [Ship to Customer]
         → [Customer Install] → [Active Use] → [Replace / Return / Retire]
```

Each stage has associated state in the database and a set of tasks, some manual (you) and some automated (the system).

---

## 3. Data Model

### devices

Physical OBD2 device registry.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Device record identifier |
| imei | TEXT | UNIQUE, NOT NULL | 15-digit hardware identifier |
| iccid | TEXT | | SIM card identifier |
| model | TEXT | | e.g. "Teltonika FMC003" |
| firmware_ver | TEXT | | Device firmware version |
| status | TEXT | DEFAULT 'in_stock' | in_stock, provisioned, shipped, active, offline, faulty, returned, or retired |
| account_id | UUID | FK → accounts | Null until assigned to a customer |
| vehicle_id | UUID | FK → vehicles | Null until paired with a vehicle |
| provisioned_at | TIMESTAMPTZ | | When device was configured |
| shipped_at | TIMESTAMPTZ | | When shipped to customer |
| activated_at | TIMESTAMPTZ | | First telemetry received |
| returned_at | TIMESTAMPTZ | | When device was returned |
| notes | TEXT | | Free text for manual ops notes |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Record creation time |

### device_events

Audit trail for device state transitions.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Event identifier |
| device_id | UUID | FK → devices | Related device |
| event_type | TEXT | | provisioned, assigned, shipped, activated, went_offline, came_online, fault_detected, replacement_requested, returned, or retired |
| payload | JSONB | | Event-specific metadata |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Event time |

### shipments

Device shipping records.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Shipment identifier |
| device_id | UUID | FK → devices | Shipped device |
| account_id | UUID | FK → accounts | Destination customer |
| direction | TEXT | | outbound or return |
| tracking_number | TEXT | | Carrier tracking number |
| carrier | TEXT | | e.g. "Israel Post", "DHL" |
| status | TEXT | DEFAULT 'pending' | pending, shipped, delivered, or failed |
| shipped_at | TIMESTAMPTZ | | When shipped |
| delivered_at | TIMESTAMPTZ | | When delivered |
| address_snapshot | JSONB | | Delivery address snapshot at ship time |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | Record creation time |

---

## 4. Stage 1 — Procurement & Receiving

**Your process (manual):**

1. Order devices from supplier (Teltonika via Hypertech IL, or Alibaba manufacturer)
2. On receipt, inspect each unit — power on, confirm LED behavior matches spec
3. Register each device in the admin panel:
   - Enter IMEI (printed on device label)
   - Enter SIM ICCID (printed on SIM or packaging)
   - Set model/firmware version
   - Status → `in_stock`

**Minimum stock recommendation:**

- 0–10 customers: Keep 2–3 spare units on hand
- 10–50 customers: Keep 10% of active fleet as spares

---

## 5. Stage 2 — Provisioning

Provisioning means configuring the device to connect to _your_ Railway TCP server before it ships to a customer. This must happen before shipment — never ship an unconfigured device.

**Provisioning checklist (per device):**

- [ ] Insert SIM card
- [ ] Power device via USB or OBD2 test harness
- [ ] Send SMS configuration commands:
  ```
  Server IP/domain:   adminip[password] [railway-host] [port]
  APN (Monogoto):     APN[password] data.mono
  Heartbeat interval: TIMER[password] 30     ← 30-second update interval
  ```
- [ ] Confirm device connects — check admin panel for incoming telemetry from this IMEI
- [ ] Send SMS to reset/reboot device: `REBOOT[password]`
- [ ] Confirm reconnection after reboot
- [ ] Set device status → `provisioned`
- [ ] Log provisioning timestamp and config snapshot

**Why provision before shipping?**
If the customer tries to configure it themselves and makes a typo in the SMS command, the device silently fails and you have a support ticket. Pre-provisioning eliminates the most error-prone step entirely.

---

## 6. Stage 3 — Assignment & Shipping

When a customer adds a vehicle and requests a device:

1. **Assign device to account/vehicle** in admin panel
   - Select an `in_stock` or `provisioned` device
   - Link to the customer's account and vehicle record
   - Status → `shipped`

2. **Create shipment record** with tracking number and carrier

3. **Send customer shipping confirmation email** including:
   - Tracking link
   - Installation guide (plug into OBD2 port under dashboard, driver's side)
   - What to expect: "Within 5 minutes of plugging in, you should see your vehicle appear on the map"
   - Support contact if it doesn't work

**Israeli shipping options:**

- Israel Post registered mail: ~₪15–25, 2–3 business days, tracking available
- DHL Express Israel: ~₪60–80, next day, better for high-value devices
- Recommendation: Israel Post for standard, DHL for replacements where downtime matters

---

## 7. Stage 4 — Customer Installation

Installation is a customer self-service action — OBD2 is plug-and-play, no tools required. Your job is to make activation confirmation automatic.

**Activation flow:**

1. Customer plugs device into OBD2 port
2. Device powers on from vehicle's OBD2 port (always-on 12V)
3. Device connects to cellular network (Monogoto/Telnyx SIM)
4. Device sends Login Packet to Railway TCP server
5. Parser identifies IMEI → looks up device record → confirms it's assigned to a vehicle
6. First telemetry packet written to `telemetry_logs`
7. **Automatic:** device status → `active`, `activated_at` timestamp recorded
8. **Automatic:** customer receives email "✅ Van #3 is now live on VitalsDrive"
9. **Automatic:** you receive admin notification of successful activation

If no activation within 48 hours of shipment delivery:

- Automated email to customer with troubleshooting steps
- Admin flag on the device record for your attention

---

## 8. Stage 5 — Active Use & Monitoring

Once active, the system continuously monitors device health:

| Condition                           | Detection Method                         | Action                                                  |
| ----------------------------------- | ---------------------------------------- | ------------------------------------------------------- |
| Device offline >2h (business hours) | Cron job checks last telemetry timestamp | Flag in admin, proactive customer email                 |
| Device offline >24h                 | Same                                     | Admin alert, create support ticket automatically        |
| Voltage consistently <11.5V         | Telemetry analysis                       | Alert customer (device may lose power)                  |
| No GPS fix (lat/lng = 0,0)          | Telemetry check                          | Flag in admin — device may be in a dead zone or blocked |

---

## 9. Stage 6 — Replacement & Returns

### Replacement Request

Customer submits a support ticket categorized as "Device Offline" that escalates to hardware fault, or requests replacement directly.

**Your process:**

1. Confirm fault via diagnostic snapshot in ticket (was device ever active? last seen?)
2. If confirmed hardware fault:
   - Create new shipment with a replacement device (provisioned unit from stock)
   - Ship via DHL Express (priority)
   - Include return label for the faulty unit
   - Update original device status → `faulty`
   - Update replacement device → assigned to same vehicle
3. Customer plugs in replacement → activates automatically (same IMEI lookup flow)

### Return Policy (Suggested)

| Scenario                        | Policy                                                 |
| ------------------------------- | ------------------------------------------------------ |
| DOA (never activated)           | Full replacement at no cost, within 30 days            |
| Hardware fault within 12 months | Replacement at no cost (warranty)                      |
| Customer cancels contract       | Device must be returned or ₪150 non-return fee charged |
| Lost device                     | ₪150 replacement fee                                   |

### Return Processing

1. Device received back at your address
2. Inspect — can it be refurbished and re-provisioned?
3. If yes: clean, test, re-provision → status: `in_stock`
4. If no: status: `retired`, dispose or return to supplier

---

## 10. Admin Hardware Panel

A `/admin/hardware` view in Angular:

| View                   | Content                                                                |
| ---------------------- | ---------------------------------------------------------------------- |
| **Inventory**          | All devices by status — in_stock, provisioned, shipped, active, faulty |
| **Active Devices**     | All live devices with last-seen timestamp, assigned account/vehicle    |
| **Offline Devices**    | Devices with no telemetry >2h, sorted by last seen (oldest first)      |
| **Pending Activation** | Shipped but not yet activated (>48h since delivery = flag)             |
| **Faulty / Returned**  | Historical log of replaced hardware                                    |

---

## 11. MVP Sequencing

| Priority | Feature                                     | Rationale                                   |
| -------- | ------------------------------------------- | ------------------------------------------- |
| P0       | Device registry (IMEI, status, assignment)  | Can't track hardware without this           |
| P0       | Provisioning checklist + status tracking    | Most critical pre-ship step                 |
| P0       | Auto-activation on first telemetry          | Core "it works!" moment                     |
| P0       | Admin hardware inventory view               | You need to know what you have              |
| P1       | Shipment record + tracking number logging   | Basic ops traceability                      |
| P1       | Offline device detection + admin flag       | Catch problems before customers report them |
| P1       | Activation confirmation email to customer   | Professional, reduces support tickets       |
| P2       | Automated 48h no-activation follow-up email | Deflects "my device doesn't work" tickets   |
| P2       | Return processing workflow                  | Only needed once you have returns           |
| P3       | Refurbishment tracking                      | Only relevant at scale                      |

---

## 12. Open Questions

1. Will you ship devices yourself from home, or use a small fulfillment partner (e.g. a local logistics company in IL)?
2. What's the minimum order quantity from your supplier, and how does that affect your initial inventory investment?
3. Will devices be sold (customer owns it) or rented (you own it, required return on cancel)? This affects your balance sheet and return policy significantly.
4. How will you handle customers in remote areas of Israel with weak cellular coverage (Negev, etc.)?
