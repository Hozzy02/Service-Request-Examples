# Service Request Intake Gate Architecture

## 1. Core Architectural Pillars
* **Asset Inventory Hierarchy:** Map logical **Business Applications** to environment instances (e.g., `CRM - Production`, `CRM - Staging`, `Analytics Portal - Prod`, `Sandbox Environment`).
* **Role Separation:** Keep clear operational boundaries between the operations team and the architecture and engineering team through distinct roles, team queues, and permission scopes.
* **Risk-Prioritized Compliance:** Anchor asset criticality, change risk scoring, and audit controls to a recognized control framework.

---

## 2. The Intake Gate: Review Verification at Request Time

### Architectural Overview
The **Intake Gate** is an automated evaluation workflow triggered whenever a request involves software procurement, new integrations, architecture changes, or onboarding a new application or vendor.

Rather than relying on manual triage, the gate queries the **asset inventory** at ticket creation to verify clearance status across two tracks:
1. **Architecture Review (AR):** Evaluates data flows, authentication, network egress, and alignment with the control baseline.
2. **Vendor Review (VR):** Evaluates third-party risk, attestations, data handling, and vendor posture.

### Operating Mode: Phase 1 — Soft Gate ("Log & Alert")
* **Behavior:** Tickets are **not** blocked from entering operational queues.
* **Purpose:** Establish baseline metrics on how many requests bypass intake, identify unvetted vendors, and tune false positives before moving to Phase 2 or Phase 3.
* **Enforcement Action:** Tags the ticket, flags the inventory record, logs an exception, and spawns child review tickets without halting the parent workflow.

---

## 3. Inventory Data Model
The application or vendor record requires these custom fields:

| Field Name | Field Type | Allowed Values / Format | Purpose |
| :--- | :--- | :--- | :--- |
| `ar_status` | Single Select | `Not Started`, `In Progress`, `Approved`, `Conditional`, `Expired`, `Rejected` | Tracks Architecture Review state. |
| `ar_last_reviewed` | Date | `YYYY-MM-DD` | Used to calculate 12-month expiration. |
| `ar_ticket_ref` | Ticket Link | Ticket ID | Links to the authoritative AR record. |
| `vr_status` | Single Select | `Not Started`, `In Progress`, `Approved`, `Conditional`, `Expired`, `Rejected` | Tracks Vendor Review state. |
| `vr_last_reviewed` | Date | `YYYY-MM-DD` | Tracks annual vendor re-review. |
| `vr_ticket_ref` | Ticket Link | Ticket ID | Links to the authoritative VR record. |
| `data_classification` | Single Select | `Public`, `Internal`, `Confidential`, `Restricted` | Determines whether conditional approvals need escalation. |
| `gate_exempt` | Boolean | `True` / `False` | Explicit override for pre-approved utilities. |

A scheduled rule sets `ar_status` or `vr_status` to `Expired` when the matching last-reviewed date is more than 365 days old.

---

## 4. Intake Decision Matrix

When a qualifying request is submitted (e.g., *New Software Request*, *New API Integration*, *Vendor Onboarding*, or a *Change Request* linked to an application record), the automation evaluates the linked record in this order:

### Step 0: Exemption Check
* **Condition:** `gate_exempt = True`.
* **Action:** Tag `gate:exempt`, log the exemption, and route to fulfillment.

### Step 1: Inventory Lookup
* **Condition:** The request references an application or vendor not in the inventory.
* **Action (Soft Gate):**
  1. Create a draft record with `ar_status = Not Started` and `vr_status = Not Started`.
  2. Tag `gate:unregistered-asset`.
  3. Alert the review queue and spawn both child review tickets.

### Step 2: Vendor Review Check
* **Condition:** `vr_status` is `Not Started`, `Expired`, or `Rejected`.
* **Action (Soft Gate):**
  1. Tag `gate:missing-vr` (or `gate:expired-vr`).
  2. If a VR ticket is already open, link it to the request. Otherwise spawn a new VR child ticket and set `vr_status = In Progress`.
  3. Post an internal note: *"Soft Gate Notice: Vendor Review is incomplete. Workflow allowed to proceed under Phase 1 audit mode."*

### Step 3: Architecture Review Check
* **Condition:** `ar_status` is `Not Started`, `Expired`, or `Rejected`, or an architecture-impacting integration is flagged on an existing system.
* **Action (Soft Gate):**
  1. Tag `gate:missing-ar`.
  2. If an AR ticket is already open, link it. Otherwise spawn a new AR child ticket and set `ar_status = In Progress`.
  3. Log a soft-gate event for monthly reporting.

### Step 4: Clearance Pass
* **Condition:** `vr_status = Approved` AND `ar_status = Approved`, both within the review window.
* **Action:** Tag `gate:cleared`, append a verification timestamp, and route to fulfillment.

### Step 5: Fallthrough
* **Condition:** Any state not covered above, such as `Conditional` on either review or a mix of approved and in-progress.
* **Action:** Tag `gate:needs-attention`, notify the review queue, and apply the current phase's rule.

---

## 5. Phased Rollout

| Phase | Mode | Gate Behavior | Exit Criteria |
| :--- | :--- | :--- | :--- |
| **Phase 1 (Current)** | **Log & Alert (Soft Gate)** | Evaluates inventory, tags tickets, spawns missing review tickets, logs metrics; never blocks. | 30–60 days of baseline data; inventory covers the top 80% of active apps and vendors. |
| **Phase 2** | **Conditional Hold** | Blocks only when `data_classification = Restricted` or a `Rejected` status is present; soft-gates the rest. Overrides need a named approver and are logged. | Zero false-positive blocks on standard requests over 30 days. |
| **Phase 3** | **Hard Gate** | Blocks fulfillment until both `ar_status` and `vr_status` equal `Approved`. | Executive and change board sign-off. |

---

## 6. Implementation Checklist
1. **Create Custom Fields:** Add the `ar_*`, `vr_*`, `data_classification`, and `gate_exempt` fields to the application or vendor record type.
2. **Build Child Ticket Templates:** Create templates for Architecture Review and Vendor Review, with automation that writes `Approved` and the review date back to the parent record on closure.
3. **Configure the Intake Rule:** Add a ticket-creation or status-change rule on intake forms that runs Steps 0–5.
4. **Schedule the Expiry Rule:** Set statuses to `Expired` after 365 days.
5. **Build a Soft-Gate Dashboard:** Report `gate:missing-ar`, `gate:missing-vr`, and `gate:unregistered-asset` counts by department to prepare leadership for Phase 2.
