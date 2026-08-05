# Use Cases

This document specifies the primary operational and transactional workflows driving the local monolithic MVP. 

> ⚠️ **Project Scope Note:** This version explicitly captures **Phase 1 (Local Monolithic MVP)** requirements. All cloud components, automated background timers, and offline replication logic are deferred to the Phase 2 specification sprint.

---

## 📥 UC-01: Ingest Vehicle Reception and Initialize Ticket
- **Primary Actor:** Service Advisor / Administrator.
- **Preconditions:** A vehicle has arrived at the workshop, with or without a pre-existing appointment.
- **Main Success Scenario (Flow):**
    1. The Service Advisor queries active appointments using the vehicle's license plate or opens a blank manual walk-in form.
    2. The system displays matching details or provides empty fields for manual entry.
    3. The Service Advisor records the vehicle's initial condition assessment and potential client service requests into the form.
    4. The Service Advisor confirms the reception (or cancels the form if intake is aborted), triggering the profile conversion sequence.
    5. The system creates and saves permanent `client` and `vehicle` profiles.
    6. The system generates a macro operational `ticket` linked to the new client profile and sets its state to `OPEN`.
    7. The system creates a record in the `ticket_requested_categories` table containing at least 1 valid `job_category` (e.g., brakes) linked to the newly generated `ticket`, and sets its state to `BACKLOG`.
- **Postconditions:** The client and vehicle assets are formally registered, an active service ticket is added to the workshop backlog and one or more tasks are now linked to it in a `BACKLOG` state.

---

## ⚙️ UC-02: Claim Workshop Task and Acquire Lock
- **Primary Actor:** Mechanic.
- **Preconditions:** A `ticket` exists with at least one record in `ticket_requested_categories` holding a status of `BACKLOG`.
- **Main Success Scenario (Flow):**
    1.  The Mechanic browses the global workshop backlog via the UI terminal.
    2.  The system filters visible categories strictly based on the Mechanic's current rank capability (e.g., *Low*, *Normal*, *High* difficulties).
    3.  The Mechanic selects an eligible category (e.g., *Engine Diagnostics*) and clicks **"Start Task"**.
    4.  The system validates that the category is unassigned and updates the row in `ticket_requested_categories`:
        *   Sets `assigned_mechanic_id` to the current Mechanic (Task Lead).
        *   Sets `active_worker_id` to the current Mechanic (Active Lock).
        *   Transitions task status to `IN_PROGRESS`.
- **Postconditions:** The chosen workshop category is exclusively locked by the executing technician, preventing other mechanics from logging lines against it.

---

## 🔄 UC-03: Execute Manual Concurrency Task Takeover
- **Primary Actor:** Releasing Mechanic (Mechanic B).
- **Preconditions:** A category in `ticket_requested_categories` is currently locked (`status = 'IN_PROGRESS'`) by an absent colleague (Mechanic A).
- **Main Success Scenario (Flow):**
    1.  Mechanic B opens the task board interface and selects the locked category.
    2.  The system displays a visual warning stating the task is currently active under Mechanic A's credentials.
    3.  Mechanic B clicks the **"Force Takeover"** override button.
    4.  The system triggers an atomic database cutoff transaction:
        - Finds Mechanic A's open `job_line` and stamps `completed_at = CURRENT_TIMESTAMP`.
        - Updates `ticket_requested_categories.active_worker_id` to Mechanic B's ID.
    5.  The frontend displays a green success toast notification, unlocking write-access for Mechanic B.
- **Postconditions:** The historical labor of Mechanic A is safely sealed, and Mechanic B acquires the active floor lock without creating empty placeholder rows.

---

## 📦 UC-04: Log Billable Labor and Material Lines
- **Primary Actor:** Active Mechanic.
- **Preconditions:** The category in `ticket_requested_categories` has its `active_worker_id` matching the current logging Mechanic.
- **Main Success Scenario (Flow):**
    1. The Mechanic opens the sidebar inventory component, filtered automatically by the active task category.
    2. The Mechanic enters an alphanumeric code (SKU) or a text string into the catalog search bar to select a matching part or labor item.
    3. The Mechanic populates the execution form, specifying the required quantity (limited to 1 for labor items) and inputting an optional note, and clicks **"Append Line"**.
    4. The system validates the input transaction against warehouse limits:
        - If the item is a physical **PART**, the system verifies that `current_stock` is greater than or equal to the requested quantity.
    5. The system commits an atomic record transaction:
        - Appends a new row to the `job_line` table, automatically stamping the `created_at` timestamp and setting line to `status = 'PENDING'` and `is_billable = true`.
        - If the item is a **PART**, the system deducts the material count directly from the warehouse inventory (`current_stock = current_stock - requested_quantity`).
    6. The system evaluates real-time inventory threshold limits:
        - If the new `current_stock` sits at or below the safety `min_stock` limit, the system logs a high-priority `LOW_STOCK_ALARM` event to the central audit ledger.
    7. The frontend dashboard refreshes the active task scope view to render the newly created execution line on the mechanic's grid.
- **Postconditions:** Financial operational records are linked directly to the task context, physical inventory balances are atomically decremented, and real-time replenishment alarms are triggered if safety thresholds are breached.

---

## 🛑 UC-05: Process Mid-Repair Scope Disagreement (Crossover Cutoff)
- **Primary Actor:** Service Advisor / Administrator.
- **Preconditions:** An active category requires a dynamic shift in operational scope due to hidden vehicle damage, and the Administrator records a client alteration or rejection of the proposed budget.
- **Main Success Scenario (Flow):**
    1.  The Administrator triggers a mid-repair scope management loop from the administration terminal.
    2.  The system identifies the active category and seals all completed `job_line` items accumulated up to that millisecond, ensuring they remain immutable and billable.
    3.  The system transitions the old category record in `ticket_requested_categories` to `CANCELLED` (or `PARTIAL_STOP`).
    4.  In case of a partial stop being agreed, the Administrator appends a **brand new, distinct row** to `ticket_requested_categories` for the same technical category domain, logging the client's updated instructions.
    5.  The system pushes this new clean category to the general backlog in `BACKLOG` state.
- **Postconditions:** Historical business ambiguity is eliminated; past work is locked for billing, and the new scope begins a clean relational lineage.

---

## 🛡️ UC-06: Perform Quality Control Review and Log Rejection
- **Primary Actor:** Peer Reviewer (Mid Mechanic, Senior Mechanic, Chief, or Administrator).
- **Preconditions:** A task category has its status set to `QA_PENDING`. The reviewer holds an eligible rank relative to the item tier (e.g., Mid mechanics can review *Low* difficulty labor).
- **Main Success Scenario (Flow):**
    1.  The Reviewer opens the QA evaluation panel for the completed vehicle category (task).
    2.  The system runs the **Multi-Author Validation Engine**:
        - Scans `job_line` authors within the scope:
          1. **Strict Mode:** Blocks the Reviewer if they logged *any* line inside this task category.
          2. **Relaxed Mode:** Permits the Reviewer to inspect the task even if they participated in it, but strictly blocks them from approving the specific `job_line` rows they personally authored.
    3.  If a line fails visual or technical inspection, the Reviewer flags the specific line as rejected and submits the form.
    4.  The system triggers a new transaction:
        - Updates the target row in `job_line` setting `status = 'REJECTED'`, `id_reviewer = :reviewerId`, `reviewed_at = CURRENT_TIMESTAMP`, and `is_billable = false`.
        - Appends a specialized `review_rejection_note` item as a new `job_line` row with `is_billable = false`.
        - Sets the parent task status in `ticket_requested_categories` to `IN_PROGRESS_REJECTED` and clears `active_worker_id`.
- **Postconditions:** Already approved lines remain frozen and safe, while the parent task is re-opened on the floor for targeted rectification of the affected components.

---

## 💳 UC-07: Compile and Settle Financial Client Invoice
- **Primary Actor:** Administrator / Financial Cashier.
- **Preconditions:** All categories associated with the macro `ticket` hold a status of `COMPLETED` (or `CANCELLED` / `PARTIAL_STOP`) and have passed QA review where applicable.
- **Main Success Scenario (Flow):**
    1.  The Administrator requests invoice generation for the completed macro `ticket`.
    2.  The system initializes a domain filter stream that scans all child `job_line` items across the ticket pipeline.
    3.  The system strips out all rows holding `is_billable = false` (internal QA rejection notes and unapproved lines), calculating the true gross total.
    4.  The system generates the `invoice` record in a `PENDING_PAYMENT` state.
    5.  Upon verifying payment or recording commercial friction, the Administrator updates the state to `PAID` or `PAID_DISPUTED` (logging text details in `disputed_notes`).
    6.  The system atomically sets `invoice.completed_at = CURRENT_TIMESTAMP` and `ticket.completed_at = CURRENT_TIMESTAMP`, closing the parent macro `ticket` (`status = 'CLOSED'`) and authorizing vehicle checkout.
- **Postconditions:** Internal data leaks are prevented from polluting client paperwork, and the vehicle visit completes its absolute 1-1 ledger pipeline (`ticket` → `invoice`).

---

## 📝 UC-08: Amend Administrative Ticket Notes
- **Primary Actor:** Service Advisor / Administrator.
- **Preconditions:** A macro `ticket` exists in an active state (`OPEN`, `IN_PROGRESS`, `READY_FOR_BILLING`).
- **Main Success Scenario (Flow):**
    1. The Service Advisor opens the active ticket management panel on their terminal.
    2. The Service Advisor edits the internal text string within the `advisor_notes` form field.
    3. The Service Advisor clicks **"Save Changes"**.
    4. The system updates the `ticket` record in the database with the new text.
    5. The system triggers an atomic audit logging transaction:
        - Appends a new event row to the `system_event_log` table with an `event_type` of `TICKET_AMENDED`.
        - Captures the old text and new text inside the `historical_snapshot` JSONB delta column.
- **Postconditions:** The administrative ticket record is updated with new operational information, and an immutable delta history log is saved to the audit ledger.

---

## 📡 UC-09: Synchronize Task Board and Clear Activity Signals
- **Primary Actor:** Mechanic.
- **Preconditions:** The Mechanic is viewing the central workshop dashboard or an active task screen (`status = 'IN_PROGRESS'`).
- **Main Success Scenario (Flow):**
    1. The Mechanic opens or manually refreshes their active task workspace view.
    2. The system queries the `system_event_log` to find the latest update timestamp for that specific ticket.
    3. The system compares the log timestamp against the task's recorded `worker_last_seen_at` timestamp.
    4. If a newer `TICKET_AMENDED` or structural addition event is detected, the Thymeleaf engine appends an alerting CSS rule to render a pulsing **Halo Effect** around the task card.
    5. The Mechanic clicks into the blinking task card to review the "Activity Feed" panel.
    6. The system executes a background database transaction:
        - Updates `ticket_requested_categories.worker_last_seen_at = CURRENT_TIMESTAMP`.
    7. The frontend clears the pulsing Halo Effect, signaling that the user is fully synchronized with the administrative data.
- **Postconditions:** The technician is visually synchronized with the latest front-office instructions, and the task's last-seen threshold is updated to clear active warnings.