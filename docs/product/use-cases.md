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
- **Postconditions:** The client and vehicle assets are formally registered, an active service ticket is added to the workshop backlog and one or more tasks are now linked to it.

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

## 📝 UC-04: Log Billable Labor and Material Lines
- **Primary Actor:** Active Mechanic.
- **Preconditions:** The category in `ticket_requested_categories` has its `active_worker_id` matching the current logging Mechanic.
- **Main Success Scenario (Flow):**
    1.  The Mechanic opens the sidebar inventory component, filtered by the active task category.
    2.  The Mechanic uses drag-and-drop or text search to select a part or labor item.
    3.  The Mechanic submits the input form specifying quantities or duration.
    4.  The system appends a new row to the `job_line` table, automatically stamping the unallocated `created_at` timestamp and setting `is_billable = true`.
- **Postconditions:** Financial operational records are linked directly to the task context, tracking precise labor metrics for subsequent invoicing.

---

## 🛑 UC-05: Process Mid-Repair Scope Disagreement (Crossover Cutoff)
- **Primary Actor:** Service Advisor / Administrator.
- **Preconditions:** An active category requires a dynamic shift in operational scope due to hidden vehicle damage, and the client rejects or alters the future budget proposal.
- **Main Success Scenario (Flow):**
    1.  The Administrator triggers a mid-repair scope management loop from the administration terminal.
    2.  The system identifies the active category and seals all completed `job_line` items accumulated up to that millisecond, ensuring they remain immutable and billable.
    3.  The system transitions the old category record in `ticket_requested_categories` to `CANCELLED` (or `PARTIAL_STOP`).
    4.  In case of a partial stop being agreed, the Administrator appends a **brand new, distinct row** to `ticket_requested_categories` for the same technical category domain, logging the client's updated instructions.
    5.  The system pushes this new clean category to the general backlog in `PENDING` state.
- **Postconditions:** Historical business ambiguity is eliminated; past work is locked for billing, and the new scope begins a clean relational lineage.

---

## 🛡️ UC-06: Perform Quality Control Review and Log Rejection
- **Primary Actor:** Peer Reviewer (Mid Mechanic, Senior Mechanic, Chief, or Administrator).
- **Preconditions:** A task category has its status set to `QA_PENDING`. The reviewer holds an eligible rank relative to the item tier (e.g., Mid mechanics can review *Low* difficulty labor).
- **Main Success Scenario (Flow):**
    1.  The Reviewer opens the QA evaluation panel for the completed vehicle category.
    2.  The system runs the **Multi-Author Validation Engine**:
        - Scans `job_line` authors within the scope:
          1. **Strict Mode:** Blocks the Reviewer if they logged *any* line inside this task category.
          2. **Relaxed Mode:** Permits the Reviewer to inspect the task even if they participated in it, but strictly blocks them from approving the specific `job_line` rows they personally authored.
    3.  If a line fails visual or technical inspection, the Reviewer logs a **QA Rejection Note**.
    4.  The system triggers a new transaction:
        - Appends a specialized QA Rejection Note item as a `job_line`.
        - Forces `is_billable = false` on that rejection line item and the affected job lines.
        - Updates the state of the faulty execution rows in the `job_line` table to `REJECTED` and records the reviewer's ID.
        - Sets the parent task status in `ticket_requested_categories` to `IN_PROGRESS_REJECTED` so work can be restarted.
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
    6.  The system atomically sets `invoice.completed_at = CURRENT_TIMESTAMP` and `ticket.completed_at = CURRENT_TIMESTAMP`, closing the parent macro `ticket` and authorizing vehicle checkout.
- **Postconditions:** Internal data leaks are prevented from polluting client paperwork, and the vehicle visit completes its absolute 1-1 ledger pipeline (`ticket` → `invoice`).
