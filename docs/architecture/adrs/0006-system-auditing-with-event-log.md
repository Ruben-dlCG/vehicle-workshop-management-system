# ADR 6: Immutable System Auditing and State Decoupling

## ℹ️ Context
An enterprise workshop application requires a permanent, unalterable security audit trail to track operational floor actions (such as `TASK_TAKEOVER`, `QA_REJECTED`, or `INVOICE_GENERATED`) to ensure operational accountability and prevent internal fraud. To satisfy this need, I previously created the `system_event_log` table in the database relational schema.

However, I soon found that mapping an audit engine in the way it was originally planned for the local monolithic MVP phase introduced three architectural challenges:
1. **The Deletion Trap:** If a Service Advisor or Administrator corrects a user error by deleting an accidental or messy draft row (e.g., an accidental invoice or a mistaken task category), a standard database relation utilizing `ON DELETE CASCADE` automatically wipes out the corresponding historical log rows. Conversely, using `ON DELETE RESTRICT` causes a business bottleneck by preventing users from cleaning up their everyday data entry mistakes.
2. **Context Preservation:** Raw relational data structures only point to current states. If an entity's operational values change over time (e.g., labor pricing rates are updated), an old audit log row pointing purely to that entity's foreign key loses its precise historical context.
3. **The Silent Change Problem (Floor UI):** When an advisor edits internal administrative details (e.g., updating vehicle instructions or notes), the mechanics operating on the floor remain blind to the alteration unless a specific tracking strategy guides their attention.

## 📍 Decision
I have chosen to implement an explicit, metadata-enriched audit ledger strategy featuring structured document serialization directly inside the `system_event_log` table, paired with lightweight state variables to drive frontend activity tracking:

1. **Rejection of Polymorphic Generic Keys:** I have explicitly rejected abstract generic foreign keys (paired text/UUID columns) to preserve an accurate representation of the entity diagram layout on our canvas. The log table shall maintain explicit, nullable columns for workshop entities susceptible to the abovementioned "deletion trap" challenge (e.g., `id_requested_category`, `id_job_line`, `id_invoice`).
2. **Defensive Deletion Boundaries (`SET NULL`):** All optional operational foreign keys inside the audit schema are strictly configured with `ON DELETE SET NULL`. If a parent operational asset is eliminated from the active workspace, the database engine executes the deletion smoothly without blocking the user, while leaving the historical log row completely untouched.
3. **Structured JSONB Historical Snapshots:** To completely eliminate data loss during a deletion, the system service layer must capture and serialize a lightweight JSON snapshot (`historical_snapshot`) of the target payload's critical business values (e.g., `{"invoice_number": "INV-102", "amount": 450.00}`) before committing the log entry. If a relation turns to `NULL`, this text bunker preserves the unalterable data parameters permanently.
4. **Contextual Update Delta Tracking:** For update events (e.g., `TICKET_AMENDED`), the `historical_snapshot` will store a precise structural delta tracking the properties *before* and *after* the mutation.
5. **Passive Task Activity Signaling (The Halo Indicator):** To prevent performance overhead, the system adds a `worker_last_seen_at` timestamp directly to the `ticket_requested_categories` table. When loading the interface, the server compares this timestamp against the latest audit entries. If a new update is found, the UI applies a **glowing Halo Effect** around the task card.
6. **Multi-Axis Environmental Verification:** To provide enterprise-grade forensic monitoring, every log event must capture the `client_ip_address` (network location mapping) and the `user_agent` (device/software signature) simultaneously. This decouples our tracking from simple database transactions and records environmental reality.


## ⚖️ Consequences & Justification
- **Absolute Traceability:** Combining the plain-text description field and the JSONB snapshot guarantees that an administrative auditor can reconstruct the exact narrative of an event, even if the primary financial rows are deleted from the active backlog, whether by negligence or malice.
- **Operational Fluidity:** Workshop employees retain the flexibility to erase data-entry mistakes in different stages of the vehicle lifecycle (e.g., vehicle intake, log of technical execution lines), without locking up the database engine or running into restrictive relational traps.
- **Asynchronous UI Synchronization:** Utilizing the time-stamped comparison engine enables a rich, interactive, unread activity status indicator without forcing the need to manage complex multi-threaded memory states or heavy database polling rules.
- **Phase 2 Analytics Readiness:** Flattening environmental metadata and snapshots into a single transactional row ensures this table structure scales natively into cold-storage, high-speed time-series logging engines during our cloud migration cycle.

## 🌐 Architectural Roadmap Adjustments (Phase 1 vs Phase 2)
- **Phase 1 (Local Monolith):** The Spring Boot service layer will handle the JSON serialization processing inline within transactional scopes. The frontend UI terminal exposes these logs statically via an "Activity Feed" panel and handles the "Halo Effect" calculations entirely over stateless HTTP request-response cycles.
- **Phase 2 (Cloud Distributed):** 
    - **Real-Time Push Notifications:** A stateful streaming layer will be introduced utilizing persistent **WebSockets / Server-Sent Events (SSE)** connections. This transforms the log append events into real-time, event-driven toast notification banners across localized mobile and tablet devices, instantly alerting technicians of task takeovers, QA rejections, and administrative note shifts as they happen.
    - **Infrastructure Scaling:** Environmental data processing (`client_ip_address` and `user_agent` extraction) will be moved upstream to the AWS API Gateway layer, and old historical event rows will be automatically offloaded to cold storage to minimize primary database resource footprints.
