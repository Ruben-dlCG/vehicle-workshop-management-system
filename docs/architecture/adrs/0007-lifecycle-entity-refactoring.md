# ADR 7: Ticket Operational Lifecycle and Entity Refactoring

## ℹ️ Context
Early iterations of this architecture (recorded in the legacy ADR 5) utilized a parallel twin entity layout: a front-office `ticket` representing customer intent, and a subsequent back-office `order` representing mechanical repair authorization. However, as business requirements matured to support fluid shop floor concurrency mechanics (such as Task Takeovers and decentralized line-by-line peer signatures), managing a hollow intermediate transactional entity introduced unnecessary table bloat and data-sync risks.

Additionally, catalog metrics required refinement: capturing difficulty tiers at the root `item` catalog level forced dummy data onto physical materials (`part`), which do not possess technical execution difficulty.

## 📍 Decision
I have chosen to completely eliminate the legacy `order` database table, converting its structural responsibilities into a streamlined, highly decoupled relational hierarchy that supersedes the conventions of ADR 05:
```
[Ticket] (Admin/Billing) ───> [Ticket_Requested_Categories] (Task Board) ───> [Job_Line] (Execution)
```
1. **State Machine Separation:** Front-office administrative lifecycles are regulated by `ticket.status` (`OPEN`, `READY_FOR_BILLING`, `CLOSED`). Shop floor technical execution lifecycles are decoupled entirely into the `ticket_requested_categories` table via custom task board states (`BACKLOG`, `IN_PROGRESS`, `IN_PROGRESS_REJECTED`, `QA_PENDING`, `COMPLETED`, `CANCELLED`).
2. **Financial Boundary Flattening:** The pipeline constraints are simplified to a strict **1 Ticket ──> 1 Invoice** architecture. The `ticket` serves as the singular core root ledger container, linking atomic `job_line` entries instead of the old `order_line`.
3. **Isolating Difficulty to Services:** The `difficulty` enumeration tier (`LOW`, `NORMAL`, `HIGH`) is moved from the base `item` schema and isolated exclusively inside the `service` labor table. Physical warehouse items (`part`) remain strictly focused on inventory metrics (`current_stock`, `min_stock`), freeing them from difficulty constraints.

## ⚖️ Consequences & Justification
- **Lean Database Footprint:** Deleting a vestigial table strips away redundant joins, simplifies the JPA mapping layer, and potentially lowers transaction failures during Phase 2 offline synchronization.
- **Granular Failure Isolation:** Failed reviews roll back the specific category status to `IN_PROGRESS_REJECTED` and clear the active worker lock, returning the item to the backlog for immediate rectification without disturbing or freezing adjacent tasks on the vehicle visit.
- **Domain Modeling Integrity:** Structuring difficulty under the `service` table keeps catalog data normalized, preventing developers from having to map dummy fields onto physical inventory stocks.