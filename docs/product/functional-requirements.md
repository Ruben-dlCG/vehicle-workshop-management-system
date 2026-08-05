# Functional Requirements

This document contains the formal behavioral specifications for the workshop management pipeline. 

> ⚠️ **Project Scope Note:** This version explicitly captures **Phase 1 (Local Monolithic MVP)** requirements. Advanced distributed cloud mechanics, automated background daemons, and offline synchronization rules are deferred to the Phase 2 development sprint cycle.

---

## 🧭 1. Intake & Vehicle Reception Module (Ticketing)

- **RF-01a (Automatic Appointment Ingestion):** The system shall permanently store incoming web appointment requests in the `appointment` database table, restricting scheduling to pre-defined, fixed operational time slots (e.g., Slot 1, Slot 2) to eliminate overlap complexity.
*   **RF-01b (Manual Appointment Registration):** The system shall allow an authorized Administrator to manually log an `appointment` in the database for walk-in or phone-in requests, following the same fixed time-slot rules.
- **RF-01c (Decoupled Appointment Persistence):** All `appointment` records shall capture basic contact and vehicle identifiers (e.g., license plate) as plain text parameters, without initializing corresponding permanent records in the primary `client` or `vehicle` entity tables.
- **RF-02 (Formal Profile Conversion):** The system shall trigger a formal client profile conversion sequence, generating permanent records in the primary `client` and `vehicle` tables, only upon the initial creation and execution of the client's first operational `ticket`.
- **RF-03 (Task Board Backlog Initialization):** The system shall allow a vehicle visit to remain in an inactive front-office queue (`ticket.status = 'OPEN'`) before a shop floor technician interacts with the task board backlog to initialize and claim a specific `ticket_requested_categories` repair scope.
- **RF-04 (Multi-Task Decoupled Ingestion):** The system shall allow a single `ticket` to hold multiple requested workshop categories simultaneously inside the `ticket_requested_categories` intermediate table, preserving the financial boundary of **1 Ticket → 1 Invoice**.

---

## ⚙️ 2. Shop Floor Execution & Quality Control (Workshop Concurrency & QA)

- **RF-05 (Granular Task Assignment & Locks):** The system shall expose pending repair categories on the backlog strictly based on mechanic rank capability. When a qualified mechanic claims a category, the system shall update `active_worker_id` to apply a system-level lock on that specific task scope.
- **RF-06 (Fluid Task Hand-off):** The system shall allow a qualified peer to execute a manual **"Force Takeover"** override on a locked task. This action shall atomically close the previous worker's active `job_line` (`completed_at = CURRENT_TIMESTAMP`) and grant the new mechanic exclusive write-access to log subsequent lines via the UI forms.
- **RF-07 (Unified Timestamp Logging):** The system shall automatically generate a mandatory `created_at` timestamp whenever a mechanic appends a new `job_line` to an active task scope. Active labor duration shall be calculated dynamically in memory using `Duration.between` against the nullable `completed_at` column.
- **RF-08 (Strict Multi-Author Review Boundary):** The system shall enforce a multi-tier peer-review gate (ADR #4). Based on configuration settings, the validation engine shall block a qualified mechanic from approving an execution line if they logged *any* `job_line` within that entire task category scope (Strict Mode) OR if they match that specific line's execution author (Relaxed Mode).
- **RF-09 (Asymmetric Validation Toggles):** The system shall support workshop-level configurations (`isSameRankReviewAllowed`) to relax quality controls, allowing Mid mechanics to validate Normal items and Senior mechanics to validate High items, provided strict work isolation is maintained.
- **RF-10 (Transaction-Based Quality Rejections):** The system shall support internal quality control rejections by updating the faulty row in `job_line` setting `line_status = 'REJECTED'`, `id_reviewer = :reviewerId`, and `is_billable = false`. The system shall concurrently insert a specialized `"QA Rejection Note"` row as a `job_line` and roll the parent task's status back to `IN_PROGRESS_REJECTED` for immediate rectification.
- **RF-11 (Immutable State-Machine Audit Ledger):** The system shall atomically log all key workshop lifecycle events (such as `TASK_TAKEOVER`, `TICKET_AMENDED`, or `INVOICE_DISPUTED`) into a dedicated `system_event_log` table (ADR #6), capturing the human author, details, environmental metadata, and a JSONB historical snapshot string with defensive `ON DELETE SET NULL` database constraints applied to temporary operational keys.

---

## 🔄 3. Mid-Repair Scope Management (Scope Creep Loop)

- **RF-12 (Mid-Repair Pause Control):** The system shall support a mid-repair scope adjustment loop when hidden vehicle damage requires customer authorization for additional work. Completed labor and parts logged up to the decision milestone shall remain immutable and billable.
- **Phased Scope Approvals:**
    - *Full Dynamic Acceptance:* The system shall allow an Administrator to append the newly discovered repair categories to the active ticket, letting qualified mechanics claim them from the backlog to log subsequent work lines.
    - *Partial Scope Acceptance (Scope Cutoff):* To eliminate historical ambiguity, if a client's decision results in rejection or alteration of a mid-repair task scope, the Administrator shall transition the current active category to a closed/cancelled state, sealing all accumulated execution lines as final. A new, distinct category row shall be appended to trace the newly agreed operational scope.
    - *Full Client Reversal (Halt Work):* The system shall instantly lock the active task views, cancel all unperformed categories, and compile an immediate partial ledger summarizing only the labor and parts consumed up to that milestone.
- **RF-13 (Administrative Text & Category Amendments):** The system shall restrict runtime text modifications strictly to the internal administrative `advisor_notes` field on active `ticket` entities. All customer request expansions or scope adjustments must explicitly spawn distinct rows within the `ticket_requested_categories` matrix. Any note amendment must log a delta snapshot mapping the 'before' and 'after' data properties to the system log.
- **RF-14 (Contextual Visual Activity Signaling):** The system shall track a mechanic's engagement history with their claimed task board rows via an active timestamp parameter (`worker_last_seen_at`). If administrative modifications or new category additions introduce records newer than the technician's last-seen threshold, frontend application components shall apply an active, pulsing **Halo Effect** boundary around the parent task view container to signal unread operational updates.

---

## 📦 4. Warehouse Control Module (Inventory)

- **RF-15 (Atomic Material Deductions):** When a mechanic logs a physical material catalog item (`item_type = 'PART'`) onto an active task scope, the system shall verify warehouse stock limits. Upon submission, the engine must execute an atomic subtraction transaction (`current_stock = current_stock - requested_quantity`) inside the `part` table before saving the corresponding `job_line` record.
- **RF-16 (Automated Stock Depletion Alarms):** The system shall evaluate warehouse metrics immediately following any physical inventory deduction. If a material's `current_stock` drops to or below its recorded `min_stock` threshold, the engine shall atomically log a `LOW_STOCK_ALARM` event inside the `system_event_log` table, forcing a high-priority replenishment warning to render across administrative dashboards without interrupting active workshop floor execution.

---

## 💳 5. Financial Settlement & Lifecycle Finalization (Billing)

- **RF-17 (Client Invoice Ledger Protection):** The system shall calculate the initial gross invoice total using a domain filter stream that explicitly strips out internal rejection lines by evaluating the `is_billable = true` relational constraint, preventing internal quality data from leaking onto customer paperwork.
- **RF-18 (Customer Dispute Logging):** The system shall provide a nullable `disputed_notes` text field directly inside the `invoice` schema to permanently record financial friction or commercial feedback against the ledger without forcing a premature warranty transaction.
- **RF-19 (Atomic Settlement Enforcement):** The system shall enforce strict transactional accounting limits, blocking the vehicle's macro `ticket` from changing state to closed and releasing the keys until the associated `invoice` status transitions out of the pending payment pool (`PENDING_PAYMENT`, `PAID`, `PAID_DISPUTED`).