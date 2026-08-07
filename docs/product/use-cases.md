# Use Cases

This document specifies the primary operational and transactional workflows driving the local monolithic MVP. 

> ⚠️ **Project Scope Note:** This version explicitly captures **Phase 1 (Local Monolithic MVP)** requirements. All cloud components, automated background timers, and offline replication logic are deferred to the Phase 2 specification sprint.

---

## 📥 UC-01: Ingest Vehicle Reception and Initialize Ticket
- **Primary Actor:** Service Advisor / Administrator.
- **Preconditions:** A `vehicle` has arrived at the workshop with a valid `appointment`.
- **Main Success Scenario (Flow):**
    1. The Service Advisor queries active appointments using the vehicle's license plate.
    2. The system displays matching appointment data parameters.
    3. The Service Advisor enters the vehicle's initial condition assessment details into the form.
    4. The Service Advisor clicks **Confirm Reception**.
    5. The system generates permanent `client` and `vehicle` profile records.
    6. The system generates a macro operational `ticket` container in an `OPEN` status.
    7. The system appends corresponding rows to the `ticket_requested_categories` table in a `BACKLOG` status.
- **Postconditions:** Active profiles are locked into production tables, and tasks are initialized on the floor backlog.

### 🧠 Alternative Flows
- **1a: Walk-In / Phone Registration**
    - *Condition:* The vehicle arrives without a pre-existing web appointment.
    - *Flow:* 
        1. At Step 1, the Advisor opens a blank, manual intake form.
        2. The Advisor enters the client identifier data manually.
        3. The flow rejoins UC-01 at Step 3.

- **4a: Aborted Reception / Form Reset**
    - *Condition:* The client rejects the intake terms, or the advisor cancels the registration.
    - *Flow:*
        1. At Step 4, the Service Advisor clicks **Cancel Intake**.
        2. The system purges the temporary form cache from memory without initializing records in the database.
        3. The flow terminates with no postconditions met.

---

## ⚙️ UC-02: Claim Workshop Task and Acquire Lock
- **Primary Actor:** Mechanic.
- **Preconditions:** A task row in `ticket_requested_categories` holds a status of `BACKLOG`.
- **Main Success Scenario (Flow):**
    1. The Mechanic browses the global workshop backlog via the UI terminal.
    2. The Mechanic selects an eligible category (e.g., *Engine Diagnostics*) and clicks **"Start Task"**.
    3. The system validates that the category is unassigned and matches the mechanic's rank capability.
    4. The system updates the task row: sets `assigned_mechanic_id` and `active_worker_id` to the current user, and sets `status = 'IN_PROGRESS'`.
- **Postconditions:** The chosen workshop category is exclusively locked by the executing technician, preventing other mechanics from logging lines against it.

### 🧠 Alternative Flows
- **3a: Capabilities Gate Violation**
    - *Condition:* The Mechanic attempts to bypass UI restrictions or claim a task that exceeds their rank capability.
    - *Flow:*
        1. At Step 3, the validation engine detects that the task's `difficulty` tier is higher than the technician's allowed rank.
        2. The system blocks the status modification, logging a `FAILED_VALIDATION` event to the audit table.
        3. The UI flashes a red warning banner: *"Access Denied: Rank capability insufficient for this technical category."*
        4. The task remains in the general backlog.

---

## 🔄 UC-03: Execute Manual Concurrency Task Takeover
- **Primary Actor:** Administrator / Chief Mechanic or Incoming Mechanic.
- **Preconditions:** A category task inside `ticket_requested_categories` is currently locked (`status = 'IN_PROGRESS'`) by an absent operator, and the system environment is running under its default configuration (`isPeerTakeoverAllowed = false`).
- **Main Success Scenario (Flow):**
    1. The Incoming Mechanic identifies the need to work on a locked task board item and requests managerial intervention.
    2. An Administrator or Chief Mechanic logs into the workshop terminal and accesses the specific active task control panel view.
    3. The Administrator or Chief Mechanic clicks the **"Force Takeover Override"** button (visible exclusively to administrative or chief management roles under the default configuration).
    4. The system executes an atomic database cutoff transaction:
        - Finds the open `labor_line` row belonging to the absent mechanic and stamps its `completed_at` column with `CURRENT_TIMESTAMP`.
        - Updates the `ticket_requested_categories.active_worker_id` value with the Incoming Mechanic's UUID, granting them the fresh active write-lock.
        - Records a high-priority `SUPERVISOR_OVERRIDE_SIGN_OFF` event inside the `system_event_log` table to guarantee unalterable forensic traceability.
    5. The workshop dashboard interface refreshes, confirming visually that the active editing lock has been successfully transferred to the Incoming Mechanic.
- **Postconditions:** The historical labor timeline of the original mechanic is safely sealed in the database, and the new operator acquires exclusive write-access under high-tier authorization.

### 🧠 Alternative Flows
- **1a: Peer-to-Peer Autonomy (Relaxed Mode Toggle)**
    - *Condition:* The workshop environment has been explicitly configured in a relaxed modality (`isPeerTakeoverAllowed = true`), specifically designed for smaller garage operations.
    - *Flow:*
        1. At Step 1, the system detects that the peer autonomy configuration flag is active.
        2. The frontend engine renders and enables the **Force Takeover** button directly on the screen of the Incoming Mechanic, allowing them to acquire the write-lock regardless of the macro task difficulty.
        3. The Incoming Mechanic clicks the takeover button autonomously without requiring a manager's presence or credentials.
        4. The system executes the atomic database cutoff transaction described in Step 4 of the main flow, sealing the previous worker's line and updating `active_worker_id` to the Incoming Mechanic. It registers the audit footprint as a standard `TASK_TAKEOVER` event.
        5. The workflow jumps directly to Step 5, granting write access to the mechanic immediately.

---

## 🧑‍🔧 UC-04a: Log Billable Labor Service Lines and Verify Ranks
- **Primary Actor:** Active Mechanic.
- **Preconditions:** The category in `ticket_requested_categories` has its `active_worker_id` matching the current logging Mechanic. The selected `item` is a `labor` activity. The `labor` difficulty tier is less than or equal to the logging mechanic's certified rank capability
- **Main Success Scenario (Flow):**
    1. The Mechanic opens the sidebar execution component on their terminal view, filtered automatically by the active task category.
    2. The Mechanic types an alphanumeric code (SKU) or a standard text string to find and select a catalog labor activity.
    3. The Mechanic populates the form specifying an optional commentary note, and clicks **"Append Labor Line"**.
    4. The system appends a new row to the `labor_line` table with `line_status = 'PENDING'`.
    5. The frontend dashboard refreshes the active task scope view to render the newly created labor row on the grid.
- **Postconditions:** The Mechanic's labor metrics are bound to the active task ledger, and the generated row is initialized to track active floor execution times.

## 📦 UC-04b: Log Billable Material Lines and Deduct Inventory
- **Primary Actor:** Active Mechanic.
- **Preconditions:** The category in `ticket_requested_categories` has its `active_worker_id` matching the current logging Mechanic. The Mechanic is working inside a prepopulated `labor_line`. The selected `item` is a physical `part`.
- **Main Success Scenario (Flow):**
    1. The Mechanic opens the sidebar inventory component, filtered automatically by the active task category.
    2. The Mechanic enters an alphanumeric code (SKU) or a standard text string to find and select a physical catalog part.
    3. The Mechanic specifies the `requested_quantity`, inputs an optional note, and clicks **"Append Material Line"**.
    4. The system verifies that the warehouse `current_stock` is greater than or equal to the `requested_quantity`.
    5. The system appends a new row to the `material_line` table.
    6. The system deducts the material count directly from the warehouse inventory (`current_stock = current_stock - requested_quantity`).
    7. The frontend dashboard refreshes the active task scope view to render the newly created material row on the grid.
- **Postconditions:** Physical inventory balances are atomically decremented, and the operational material row is secured inside the active labor scope ledger.

### 🧠 Alternative Flows (UC-04b)
- **4a: Warehouse Inventory Stockout**
    - *Condition:* The requested material quantity of a physical `part` exceeds the available `current_stock` value.
    - *Flow:*
        1. At Step 4, the database engine blocks the transaction.
        2. The system logs a `FAILED_VALIDATION` security event to the audit ledger.
        3. The UI interrupts the flow and renders a warning toast: *"Transaction Denied: Insufficient stock available in warehouse."*
        4. The task dashboard drops the input parameters, leaving inventory untouched, and returns the mechanic to Step 3.

---

## ➕ UC-05a: Process Mid-Repair Customer Scope Expansion (Add-On Request)
- **Primary Actor:** Service Advisor / Administrator.
- **Preconditions:** The macro `ticket` sits in an active `OPEN` status, and the Client contacts the front office mid-repair to request a brand-new, unrelated service operation to be performed on the vehicle.
- **Main Success Scenario (Flow):**
    1. The Administrator opens the active ticket scope management interface panel.
    2. The Administrator clicks the **"Add New Category Scope"** button.
    3. The Administrator selects the newly requested target technical category from the master catalog directory form.
    4. The Administrator inputs the new client's instructions into the `category_advisor_notes` form field and clicks **"Confirm Scope Expansion"**.
    5. The system executes a standalone database transaction:
        - Inserts a brand-new, distinct row into the `ticket_requested_categories` table linked directly to the parent operational `ticket`.
        - Persists the text string inside the `category_advisor_notes` column of that specific new record.
        - Automatically initializes the new task status to `BACKLOG` and sets `active_worker_id = NULL`.
    6. The system appends a `TICKET_EXPANDED` event row to the `system_event_log` table.
    7. The centralized workshop board updates dynamically, rendering the new task card on the general floor backlog.
- **Postconditions:** A clean, unassigned task card is broadcasted directly to the shop floor without disturbing, canceling, or altering any of the ongoing, active repairs on the vehicle.

## 🤝 UC-05b: Process Mid-Repair Scope Alteration (Partial Fix and Task Reset)
- **Primary Actor:** Service Advisor / Administrator.
- **Preconditions:** An active category task requires a structural shift in scope due to newly discovered damage, and the client rejects the current budget but authorizes an updated repair plan.
- **Main Success Scenario (Flow):**
    1. The Administrator opens the active ticket scope management interface.
    2. The Administrator selects the affected active task category and clicks **"Execute Scope Cutoff"**.
    3. The system seals all open `labor_line` rows accumulated inside that specific category and stamps them with a `completed_at` timestamp.
    4. The system transitions the status of that specific task category row to `CANCELLED`.
    5. The system clears all active `active_worker_id` lock columns to release the mechanics.
    6. The Administrator opens the task expansion form and selects the target domain (which can be a fresh instance of the same technical category or an entirely new category block).
    7. The Administrator inputs the newly negotiated client instructions into the `category_advisor_notes` form field and clicks **"Append New Task"**.
    8. The system executes a database transaction:
        - Inserts a brand-new, distinct row into the `ticket_requested_categories` table.
        - Persists the text string inside the `category_advisor_notes` column of that specific new record.
        - Pushes the task directly to the general floor backlog in a `BACKLOG` state.
- **Postconditions:** Accumulated operational costs are locked safely for future billing, the original task row is permanently closed to preserve history, and a clean task row is initialized on the floor backlog.

## 🛑 UC-05c: Process Full Client Reversal (Halt Work and Partial Compilation)
- **Primary Actor:** Service Advisor / Administrator.
- **Preconditions:** Hidden vehicle damage results in a full budget rejection by the client, who demands an immediate cessation of all workshop activities and vehicle return.
- **Main Success Scenario (Flow):**
    1. The Administrator opens the active ticket scope management interface.
    2. The Administrator clicks the high-priority **"Halt All Work"** button.
    3. The system seals all open `labor_line` rows accumulated inside all active categories and stamps them with a `completed_at` timestamp.
    4. The system transitions the status of all unperformed or active task category rows to `CANCELLED`.
    5. The system clears all active `active_worker_id` lock columns to release the mechanics.
    6. The system automatically promotes the macro parent `ticket.status` directly to `READY_FOR_BILLING`.
- **Postconditions:** All shop floor execution blocks are instantly frozen, unperformed tasks are safely purged, and the parent ticket is immediately cleared for partial invoice compilation (UC-07).

---

## 🛡️ UC-06: Perform Quality Control Review and Sign Off
- **Primary Actor:** Peer Reviewer (Mid Mechanic, Senior Mechanic, or Chief).
- **Preconditions:** A task category in `ticket_requested_categories` has its status set to `QA_PENDING`. The Reviewer holds an eligible rank relative to the category's `scope_difficulty` tier.
- **Main Success Scenario (Flow):**
    1. The Reviewer opens the QA evaluation panel for the completed task category on their terminal screen.
    2. The system executes the **Multi-Author Validation Engine**, checking user profiles against active concurrency isolation modes (Strict vs. Relaxed configuration toggle).
    3. The Reviewer visually and technically inspects the vehicle's craftsmanship on the shop floor, cross-referencing it with the logged `labor_line` operations and their nested `material_line` rows.
    4. The Reviewer verifies that all technical work is flawless, confirms the material parts, and clicks **"Approve Task Scope"**.
    5. The system executes an atomic confirmation transaction:
        - Updates all child rows in `labor_line` linked to that category to `line_status = 'APPROVED'`, stamps `id_reviewer = :reviewerId`, and sets `reviewed_at = CURRENT_TIMESTAMP`.
        - Transitions the parent task row in `ticket_requested_categories` to `status = 'COMPLETED'` and clears the `active_worker_id` write access lock.
    6. The central workshop board updates dynamically, clearing the vehicle task card from the active floor queue.
- **Postconditions:** Technical labor operations and their structural material parts are permanently verified, signed by an authorized validator, and pushed to the administrative ledger for front-office reconciliation.

### 🧠 Alternative Flows
- **3a: Material Allocation Correction (Parts Overcharge)**
    - *Condition:* The Reviewer verifies the physical labor is perfect, but identifies an uninstalled, erroneous, or extra physical part row inside a material line box.
    - *Flow:*
        1. At step 3, the Reviewer clicks the discrepancy toggle switch directly next to the faulty row in the `material_line` grid canvas.
        2. The system updates that specific record to `is_billable = false` and appends an automated log footprint.
        3. The user interface updates dynamically, highlighting the corrected part row in gray.
        4. The workflow returns straight back to Step 4 to proceed with the approval loop.

- **3b: Line-Level Technical Quality Rejection (BOM Operational Rollback)**
    - *Condition:* The Reviewer identifies an active structural defect, safety failure, or poor craftsmanship on a labor operation.
    - *Flow:*
        1. At step 3, the Reviewer flags the specific faulty `labor_line` row as rejected, inputs a mandatory technical rationale explanation into the form, and clicks **"Submit Technical Rejection"**.
        2. The system executes an atomic rollback transaction:
            - Updates the targeted row in `labor_line` to `line_status = 'REJECTED'`, stamps `id_reviewer = :reviewerId`, and sets `is_billable = false`.
            - Automatically cascades `is_billable = false` down to all child `material_line` records nested under that specific labor service parent.
            - Inserts a brand-new row into the `labor_line` table, setting `id_item` to point to a template row inside the `review_rejection_note` catalog, copying the reviewer's technical rationale string into the `mechanic_notes` column, and initializing it with `is_billable = false`.
            - Resets the parent task row in `ticket_requested_categories` to `status = 'IN_PROGRESS_REJECTED'` and clears `active_worker_id`.
        3. The system inserts a priority `TASK_QA_REJECTED` row into the `system_event_log` table.
        4. The task card updates instantly on the global floor workspace, reappearing on the backlog for targeted mechanic rectification.

---

## 🧾 UC-07: Compile, Review, and Settle Financial Client Invoice
- **Primary Actor:** Administrator (or Service Advisor / Chief Mechanic).
- **Preconditions:** All categories associated with the macro `ticket` hold a status of `COMPLETED` or `CANCELLED`, and the parent `ticket` sits in a `READY_FOR_BILLING` status.
- **Main Success Scenario (Flow):**
    1. The Administrator pulls up the active billing terminal and clicks **"Initialize Invoice Draft"**.
    2. The system executes a domain filter stream that gathers all child `labor_line` rows, calculates an initial dynamic gross total, and generates an `invoice` record holding a status of `DRAFT_REVIEW`.
    3. The system renders the **"Invoice Line Gate Check"**, displaying an interactive grid of every logged labor in the context of their parent task, and the spare parts nested inside their linked labor rows, all side-by-side with their execution author, timestamp duration (where applicable), and an active toggle switch button.
    4. The Administrator reviews the line metadata parameters to ensure operational fairness and accuracy.
    5. The Administrator clicks **"Lock and Finalize Ledger"**.
    6. The system permanently locks all associated `labor_line` and `material_line` rows from further mutation, filters out any rows where `is_billable = false`, seals the official binding financial totals (`total_base`, `total_tax`, `total_amount`), and promotes the invoice status to `PENDING_PAYMENT`.
    7. The Administrator processes the successful full client payment transaction.
    8. The system updates the `invoice` state to `PAID`.
    9. The system atomically sets `invoice.completed_at` and `ticket.completed_at` to `CURRENT_TIMESTAMP`, and permanently promotes `ticket.status = 'CLOSED'`.
- **Postconditions:** Internal data lines are validated via forced human review, financial numbers are permanently frozen before payment capture, and the vehicle visit completes its absolute 1-1 lifecycle pipeline (`ticket` → `invoice`).

### 🧠 Alternative Flows
- **4a: Flag Non-Billable Line Discrepancies**
    1. At Step 4, the Administrator identifies an erroneous, excessive, or unfair labor/part line item (e.g., a duplicated oil filter).
    2. The Administrator clicks the toggle switch button next to the faulty row.
    3. The system updates that specific row in the corresponding `material_line` or `labor_line` table, setting `is_billable = false`.
    4. The system recalculates the dynamic draft gross total on the fly and refreshes the canvas screen.
    5. The workflow returns straight back to Step 4 for final ledger locking.

- **6a: Invoicing Lockdown Due to Zero-Amount Totals**
    1. At Step 6, the domain filter stream yields a gross financial total of zero because all underlying child execution lines were set to non-billable.
    2. The system strictly blocks the ledger from finalization and prevents status promotion to `PENDING_PAYMENT`.
    3. The terminal displays a restriction warning: *"Invoicing Halted: Cannot compile a client invoice with zero billable elements. Perform manual budget reconciliation."*
    4. The `invoice` remains locked in a draft review state, forcing manual intervention back at step 4.

- **7a: Customer Dispute Settlement**
    1. At Step 7, the customer expresses commercial friction or disputes a valid line charge during checkout but demands immediate vehicle release.
    2. The Administrator opens the dispute parameter text form block on the screen.
    3. The Administrator inputs the customer's raw feedback into the `disputed_notes` schema field.
    4. The system transitions the invoice status to `PAID_DISPUTED`.
    5. The workflow rejoins back at Step 9, closing the ticket while preserving financial tracking data for subsequent management review.

---

## 📝 UC-08: Amend Administrative Ticket Notes
- **Primary Actor:** Service Advisor / Administrator.
- **Preconditions:** A macro `ticket` exists in an active operational state (`OPEN`).
- **Main Success Scenario (Flow):**
    1. The Service Advisor opens the active ticket management panel on their administrative terminal interface.
    2. The Service Advisor edits or appends text strings within the general `advisor_notes` form field and clicks **"Save Notes"**.
    3. The system executes an update query targeting the `ticket` record, overwriting the `advisor_notes` column in the database.
    4. The system automatically inserts a high-priority row into the `system_event_log` table (`event_type = 'TICKET_AMENDED'`), serializing the old text string and the newly inputted text string into a dynamic JSON delta blob secured inside the `historical_snapshot` JSONB column.
    5. The system executes a background event broadcast that compares timestamps, implicitly triggering unread activity flags for all active technicians currently assigned to tasks under that parent ticket context.
- **Postconditions:** General administrative logs are updated, an unalterable delta history log is saved to the audit ledger, and real-time activity signals are armed to notify the workshop floor via the visual Halo pipeline (UC-09).

---

## 📡 UC-09: Synchronize Task Board and Clear Activity Signals
- **Primary Actor:** Mechanic.
- **Preconditions:** The Mechanic is viewing their active personal workbench dashboard or an individual task execution screen where the task status holds a value of `IN_PROGRESS` or `IN_PROGRESS_REJECTED`.
- **Main Success Scenario (Flow):**
    1. The Mechanic opens or manually refreshes their active task workspace view context.
    2. The system queries the `system_event_log` table to retrieve the latest update timestamp (`TICKET_AMENDED` or scope alteration log) associated with that specific parent ticket.
    3. The system compares the latest log timestamp parameters against the task row's recorded `worker_last_seen_at` timestamp and detects that front-office modifications have occurred in the past.
    4. The Thymeleaf frontend template engine evaluates the timeline mismatch and injects an alerting CSS utility class:
        - **On the Dashboard Grid:** Renders a localized, pulsing **glowing Halo Effect** outline around the specific task card box.
        - **On the Individual Task Screen:** Renders a subtle, blinking high-visibility warning notification badge at the top header area of the workspace console.
    5. The Mechanic clicks the pulsing card or interacts with the header alert badge to expand the updated "Activity Feed" panel and review the new administrative data text.
    6. The system executes a background database transaction:
        - Updates the target row and sets `ticket_requested_categories.worker_last_seen_at = CURRENT_TIMESTAMP`.
    7. The frontend interface dynamically strips away the alerting CSS class utility, smoothly turning the pulsing visual indicators off.
- **Postconditions:** The Mechanic is visually synchronized with the latest front-office administrative context without interrupting ongoing floor operations, and the task's tracking ledger threshold is updated to clear active signals.

### 🧠 Alternative Flows
- **3a: Workspace Already Synchronized (Clean Default Render)**
    - *Condition:* The system detects that the latest front-office log timestamp completely aligns with or falls behind the task's recorded `worker_last_seen_at` value.
    - *Flow:*
        1. At step 3, the system skips the injection of alerting CSS utility classes entirely.
        2. The frontend terminal view renders its default, clean user interface state with zero pulsing indicators or warning banners.
        3. The Mechanic proceeds directly with standard technical logging operations (UC-04a/b), terminating the workflow with no database mutations performed.
