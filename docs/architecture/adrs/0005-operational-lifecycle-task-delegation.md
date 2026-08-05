# ADR 5: Ticket Operational Lifecycle & Task Delegation

> ⚠️ **STATUS: SUPERSEDED**  
> This decision record has been officially invalidated and superseded by **[ADR 07: Ticket Operational Lifecycle and Entity Refactoring](./0007-lifecycle-entity-refactoring.md)** due to the complete removal of the intermediate Order domain model. Retained strictly for chronological historical context.

---

## ℹ️ Context
The system must support complex vehicle repairs where multiple faults require intervention from different specialists simultaneously. To prevent invoice fragmentation and preserve traceability, a vehicle visit must map to a strict single financial pipeline (**1 Ticket → 1 Order → 1 Invoice**). 

However, allowing multiple technicians to collaborate on a single asset introduces critical concurrency challenges: preventing mechanics from overwriting each other's active work, optimizing floor throughput during shift changes, and adapting to different workshop operational sizes (small local garages vs. large enterprise dealership service departments).

Furthermore, tracking floor actions like overrides or task hand-offs presents an architectural dilemma. Utilizing empty or dummy `order_line` records to log user interactions mixes business domains, corrupts invoicing calculations, and falsely flags mechanics for quality assurance blocks under ADR 4 rules when no actual work was executed.

## 📍 Decision
I have chosen to implement a decoupled, relationally normalized task board strategy driven by a floor-level configuration flag (`isStrictFloorLockingEnabled`):

1. **Relationally Normalized Task Allocation:** The database schema has been updated to create an intermediate table called `ticket_requested_categories` which tracks individual repair scopes, explicitly rejecting native SQL arrays or JSONB columns inside the `ticket` table to preserve First Normal Form (1NF) atomicity and protect business outcomes.
2. **Granular Order Line Scoping:** Every record in the `order_line` table must maintain a foreign key reference pointing directly back to its specific authorizing parent row in `ticket_requested_categories`. This isolates material and labor lines by their specific technical domain (e.g., Engine vs. Brakes).
3. **Role Definitions within Tasks:** Each requested category records an `assigned_mechanic_id` (the technician responsible for the general diagnostic scope and fixing the issue) and an `active_worker_id` (the specific technician actively logging execution tasks at that exact moment).
4. **Execution Freedom over Assignment:** While a task tracks a primary Lead, any `mechanic` holding an equal or higher rank capability compared to the `item`'s difficulty tier is authorized to contribute `order_line` records to the active order, eliminating bottlenecked bays during time off.
5. **Concurrency Controls & Floor Modes:** To prevent technicians from stepping on each other's work, an active worker applies a system-level lock to that category. Overriding this lock is regulated by two modes (implemented manually via user request-response triggers in Phase 1, with automated systems added in Phase 2):
    - **Relaxed Mode (Default):** Standard peer-to-peer overrides via a "Force Takeover" button are allowed for qualified ranks, relying on real-world verbal synchronization inside tight-knit teams.
    - **Strict Mode (`isStrictFloorLockingEnabled` enabled):** The active lock is unbreakable by standard peers. The lock can only be broken by an automated background inactivity timeout or via an authorized system **Administrator**.
6. **Task-Isolated Timeout Design:** Under Strict Mode, inactivity is evaluated independently per task scope. A lock is considered active if the newest `created_at` timestamp of an associated `order_line` OR a manual UI frontend "Heartbeat Ping" timestamp has occurred within the set active window. If both criteria fail, only that specific task lock is released back to the general backlog.
7. **Decoupled System Audit Ledger:** To provide complete visibility into workshop management actions without corrupting financial entities, the system must log all lifecycle click-events (e.g., `TASK_TAKEOVER`, `FORCE_UNLOCK`) directly into a dedicated **`system_event_log`** table. This guarantees immediate frontend confirmation for the operator and permanent tracking for administration without generating ghost financial rows.

## ⚖️ Consequences & Justification
- **Data Integrity and Accounting Compliance:** Enforcing the 1-to-1-to-1 lifecycle constraint ensures clean ledger compilation during invoicing, completely isolating "rework/warranty" data into independent parent-linked pipelines.
- **Dynamic Capacity Optimization:** Decoupling the car from a single mechanic allows specialized teams (e.g., tire techs and master diagnostic techs) to work synchronously on different zones of the vehicle safely.
- **Independent Failure Isolation:** Scoping order lines to specific requested categories (tasks) ensures that one mechanic's negligence or departure does not impact or freeze the digital workspace of an active colleague working on the same vehicle.
- **Strict Separation of Concerns:** Introducing the `system_event_log` protects the `order_line` model. Financial queries remain fast and untainted, while administrators gain an uncompromised, tamper-proof history of workshop floor discipline, setting up an ideal dataset for any future logistics machine learning algorithms.

## 🌐 Architectural Roadmap Adjustments (Phase 1 vs Phase 2)
To maintain consistent pace during initial deployment without over-engineering the background processing layer, the floor concurrency mechanisms are divided into a phased release strategy:

- **Phase 1 (Local Monolith):** The timeout automation engine is intentionally deferred to reduce backend threading overhead. Concurrency and abandonment tracking will rely strictly on user-driven atomic request-response mechanics. If a mechanic leaves a task locked, a qualified peer can utilize the **"Force Takeover"** manual override button. This executes a clean database cutoff of the stale line and transfers the active lock immediately.

- **Phase 2 (Cloud Distributed):** 
    - **Automated Daemon Ingestion:** The background async timeout engine (`@Scheduled` cron daemon) will be introduced inside our AWS ECS containers. It will poll active tasks every 60 seconds, evaluate isolation metrics via the latest associated `order_line.created_at` timestamp, and automatically eject stale workers without manual floor intervention.
    - **Edge Device Heartbeats:** To protect technicians from anxiety during long, non-material diagnostic tasks, the client applications (Mobile/Web) will execute local frontend timers. If no screen input occurs within an specified time window, a low-friction "Still Working?" prompt will fire locally, dispatching a lightweight heartbeat packet to the central AWS Gateway to reset the lock window without altering financial line entities.
