# Functional Requirements

This document contains the formal behavioral specifications for the workshop management pipeline. 

> ⚠️ **Project Scope Note:** This version explicitly captures **Phase 1 (Local Monolithic MVP)** requirements. Advanced distributed cloud mechanics, automated background daemons, and offline synchronization rules are deferred to the Phase 2 development sprint cycle.

## 🧭 1. Intake & Vehicle Reception Module (Ticketing)

- **RF-01a (Automatic Appointment Ingestion):** The system shall permanently store incoming web appointment requests in the `appointment` database table, restricting scheduling to pre-defined, fixed operational time slots (e.g., Slot 1, Slot 2) to eliminate overlap complexity.
*   **RF-01b (Manual Appointment Registration):** The system shall allow an authorized Administrator to manually log an `appointment` in the database for walk-in or phone-in requests, following the same fixed time-slot rules.
- **RF-01c (Decoupled Appointment Persistence):** All `appointment` records shall capture basic contact and vehicle identifiers (e.g., license plate) as plain text parameters, without initializing corresponding permanent records in the primary `client` or `vehicle` entity tables.
- **RF-02 (Formal Profile Conversion):** The system shall trigger a formal client profile conversion sequence, generating permanent records in the primary `client` and `vehicle` tables, only upon the initial creation and execution of the client's first operational `ticket`.
- **RF-03 (Sequential Pipeline Integrity):** The system shall enforce a strict 1-to-0..1 relationship between a `ticket` and an `order`, allowing a vehicle to remain in a pending queue before a technician initializes and spawns the subsequent repair `order`.
- **RF-04 (Multi-Task Decoupled Ingestion):** The system shall allow a single `ticket` to hold multiple requested workshop categories simultaneously inside the `ticket_requested_categories` intermediate table, preserving the financial boundary of **1 Ticket → 1 Order → 1 Invoice**.

## ⚙️ 2. Shop Floor Execution & Quality Control (Workshop Concurrency & QA)

- **RF-05 (Granular Task Assignment & Locks):** The system shall expose pending repair categories on the backlog strictly based on mechanic rank capability. When a qualified mechanic claims a category, the system shall update `active_worker_id` to apply a system-level lock on that specific task scope.
- **RF-06 (Fluid Task Hand-off):** The system shall allow a qualified peer to execute a manual **"Force Takeover"** override on a locked task. This action shall atomically close the previous worker's active `order line` (`completed_at = CURRENT_TIMESTAMP`) and grant the new mechanic exclusive write-access to log subsequent lines via the UI forms.
- **RF-07 (Unified Timestamp Logging):** The system shall automatically generate a mandatory `created_at` timestamp whenever a mechanic appends a new `order_line` to an active task scope. Active labor duration shall be calculated dynamically in memory using `Duration.between` against the nullable `completed_at` column.
- **RF-08 (Strict Multi-Author Review Boundary):** The system shall enforce a multi-tier peer-review gate (ADR 4). The validation engine shall block a qualified mechanic from approving work if their ID matches the line's final execution author (`order_line.id_mechanic`) OR if they have logged any other execution line (`order_line`) within that same task scope.
- **RF-09 (Asymmetric Validation Toggles):** The system shall support workshop-level configurations (`isSameRankReviewAllowed`) to relax quality controls, allowing Mid mechanics to validate Normal items and Senior mechanics to validate High items, provided strict work isolation is maintained.
- **RF-10 (Transaction-Based Quality Rejections):** The system shall support internal quality control rejections by logging a specialized `"QA Rejection Note"` item as an `order_line`. The insertion of this line shall automatically set the `is_billable` flag to `false` and roll the parent task's status back to `IN_PROGRESS`/`REJECTED` for immediate rectification of the line items only.
- **RF-11 (Immutable State-Machine Audit Ledger):** The system shall atomically log all key workshop lifecycle events (such as `TASK_TAKEOVER`, `ORDER_PROMOTED`, or `INVOICE_DISPUTED`) into a dedicated `system_event_log` table, capturing the human author, details, and conditional, context-driven foreign keys for comprehensive audit tracking.

## 🔄 3. Mid-Repair Scope Management (Scope Creep Loop)

- **RF-12 (Mid-Repair Pause Control):** The system shall support a mid-repair scope adjustment loop when hidden vehicle damage requires customer authorization for additional work. Completed labor and parts logged up to the decision milestone shall remain immutable and billable.
- **Phased Scope Approvals:**
    - *Full Dynamic Acceptance:* The system shall allow an Administrator to append the newly discovered repair categories to the active ticket, letting qualified mechanics claim them from the backlog to log subsequent work lines.
    - *Partial Scope Acceptance (Scope Cutoff):* To eliminate historical ambiguity, if a client's decision results in rejection or alteration of a mid-repair task scope, the Administrator shall transition the current active category to a closed/cancelled state, sealing all accumulated execution lines as final. A new, distinct category row shall be appended to trace the newly agreed operational scope.
    - *Full Client Reversal (Halt Work):* The system shall instantly lock the active order, cancel all unperformed categories, and compile an immediate partial ledger summarizing only the labor and parts consumed up to that milestone.

## 💳 4. Financial Settlement & Lifecycle Finalization (Billing)

- **RF-13 (Client Invoice Ledger Protection):** The system shall calculate the initial gross invoice total using a domain filter stream that explicitly strips out internal rejection lines by evaluating the `is_billable = true` relational constraint, preventing internal quality data from leaking onto customer paperwork.
- **RF-14 (Customer Dispute Logging):** The system shall provide a nullable `disputed_notes` text field directly inside the `invoice` schema to permanently record financial friction or commercial feedback against the ledger without forcing a premature warranty transaction.
- **RF-15 (Atomic Settlement Enforcement):** The system shall enforce strict transactional accounting limits, blocking the vehicle's macro `ticket` from changing state to closed and releasing the keys until the associated `invoice` status transitions out of the pending payment pool (`PENDING_PAYMENT`, `PAID`, `PAID_DISPUTED`).
