# Vehicle Lifecycle & Workshop Relational Schema

This file documents the data layer constraints and entity relationships driving the logistics workflow.

## 📊 Interactive Model
The entity-relationship structure is actively maintained and updated in Lucidchart to map data flows before implementation.

👉 [Launch Interactive EER Diagram on Lucidchart](https://lucid.app/lucidchart/6a210dec-8a84-4b0c-881a-b9db306e6c1c/edit?viewport_loc=-550%2C-1110%2C3357%2C1542%2C0_0&invitationId=inv_65ed33ba-0e84-44a3-8c7d-8cff54ffcd8e)

## 🔑 Key Strategic Patterns Applied
- **UUID Implementations:** Taking into consideration Phase 2 of this project, primary keys utilize globally unique 36-character identifiers to guarantee future data sync scalability without synchronization overlap collisions.
- **Joined Inheritance:** The `person` entity acts as an abstract core shared across `customer` and `mechanic` subclasses, enforcing clean data boundaries at the schema level. Similarly, the `item` entity serves the same purpose for the `part`, `service` and `review_rejection_note` (a special non-billable type) entities.
- **Decoupled Asynchronous Workflows:** To align with real-world shop operations and preserve the strict financial boundary of **1 Ticket → 1 Order → 1 Invoice**, the direct link between a ticket and a single mechanic or category has been removed. Instead, a relational associative table, `ticket_requested_categories`, maps multiple diagnostic scopes to a single vehicle visit.
- **Concurrency & Multitenant Execution:** To allow multiple specialized mechanics to collaborate on the same vehicle without concurrency overrides, the `ticket_requested_categories` table tracks discrete task lifecycles using `assigned_mechanic_id` (the task lead responsible for diagnostics) and `active_worker_id` (the technician currently holding the active floor lock for that task).
- **Isolated Telemetry and Audit Rails:** Operational entries inside `order_line` maintain an explicit foreign key pointing back to their parent task in `ticket_requested_categories`. This isolates tracking metrics per technical domain. Additionally, a binary `is_billable` flag allows the integration of zero-cost internal quality control logs (`review_rejection_note`) within the order workflow while safely protecting final client invoices from line pollution.
- **Customer Dispute Tracking:** The `invoice` schema includes a nullable `disputed_notes` text field to permanently log customer feedback or commercial friction against the financial ledger without forcing a premature warranty or physical rework transaction.
- **History Events Log:** To improve tracking and monitor work progress, a specialized system event log table was added to the schema. This table records all key state transitions across different workshop entities (such as orders or invoices) along with the mechanic responsible for the action.

*(For detailed execution mechanics and operational state-machine rules, see ADR 5 [here](https://github.com/Ruben-dlCG/vehicle-workshop-management-system/blob/main/docs/architecture/adrs/0005-operational-lifecycle-task-delegation.md))*
