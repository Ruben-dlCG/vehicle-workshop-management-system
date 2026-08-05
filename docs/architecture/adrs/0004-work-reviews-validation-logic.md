# ADR 4: Work Reviews Validation Logic

## ℹ️ Context
To achieve excellent working standards and protect business integrity, the application must enforce peer review in completed work, before sending each Vehicle back to the Client. Nevertheless, this could be done in multiple ways, each of them providing a set of trade-offs. Some questions that came to mind were: 
-   Should self-review be contemplated as an option? 
-   Should Juniors/Apprentices be allowed to review and approve the simplest jobs? 
-   What should determine if a Mechanic is qualified enough to review another Mechanic's work? 
-   Should validation logic be applied based on the tenure/rank of the Mechanic(s) who worked on the Order, or rather be based on each Item's difficulty?

## 📍 Decision
I have chosen to apply a flexible, configuration-driven validation strategy:
1. When an Order is flagged for finalization (usually done by the Mechanic that included the last Order Line for the Vehicle), a checkbox will render next to each recorded Order Line.
2. Because each Order Line contains both a reference to the Mechanic and its underlying Item (including subclasses), the system will fetch the Mechanic's ID to compare with the reviewer's ID. If they match, the review is denied, unless both IDs belong to the only existent Chief in that specific location, provided that the Senior co-sign option is unchecked in the administrator settings.
3. Once the peer review process is initiated, the reviewer will only see the Order Lines available to their rank, based on the underlying Item's difficulty level. The system supports two modes toggled via administrator settings:
    - **Strict Mode (Default):** Low difficulty items require a Mid or higher. Normal difficulty items require a Senior or higher. High difficulty items require a Chief. Peers of the exact same rank cannot review matching difficulty items (e.g., a Mid cannot review a Normal/Mid item).
    - **Relaxed Mode (`isSameRankReviewAllowed` enabled):** Intended for shops with a reduced headcount. The rules are relaxed so Mid mechanics can review Normal items, and Senior mechanics can review High items, provided they did not perform the original work.
4. Regardless of the above, Juniors are never allowed to formally review any other Mechanic's work nor their own work.
5. In the case of Chief Mechanics handling High-difficulty items, they will be allowed self-review only if they are the only Chief at the workshop location and the "Senior co-sign" option is not checked. If it is checked, the system mandates a dual-signature from an available Senior mechanic.

## ⚖️ Consequences & Justification
- **De-risking Software Constraints:** I'm the developer, not the shop owner. In micro-workshops of 5 to 6 people, the owner knows their team's trustworthiness better than any hardcoded rule. Providing a toggle shifts operational risk tolerance to the administrator.
- **Avoidance of bottlenecks:** Grounding the review process based on Item difficulty instead of purely Mechanic rank means that if a higher-ranked Mechanic performs a lower-difficulty job (due to sick leave, scheduling, etc.), the review processes smoothly based on the task itself.
- **Exlusion of Juniors:** A Junior mechanic, by definition, is still learning and lacks the diagnostic experience to spot subtle mistakes or safety issues.
- **Flexibility for Chiefs:** This application caters to workshops of any size, which often hold different realities for the Chief role. In many small businesses, the Chief is usually the owner too, and they are often burdened with heavy multi-tasking due to their shop's reduced headcount. To keep operations running smoothly, these businesses can benefit from solo-chief self-review. Conversely, growing mid-size configurations can toggle on the Senior co-sign feature to leverage experienced Senior Mechanics for quality control assistance, especially during periods of high demand.

## 🌐 Architectural Roadmap Adjustments (Phase 1 vs Phase 2)
To ensure this validation logic scales cleanly across the planned deployment phases, the technical implementation must observe the following constraints:
- **Phase 1 (Local Monolith):** The validation rules and configuration flags (`isSameRankReviewAllowed`, `isSeniorCoSignEnabled`) will reside strictly within the Spring Boot Business/Service layer. Database interactions will go directly to local PostgreSQL via Spring Data JPA, ensuring instant data consistency.
- **Phase 2 (Cloud Distributed):**
    - **Dual-Layer Validation:** The validation logic must be written symmetrically on both the frontend client applications (to ensure instant UI feedback) and inside the AWS ECS REST Controllers. The server must explicitly re-validate all payloads to prevent client-side bypasses.
    - **Offline Sync Conflict Strategy:** Because the mobile app will utilize a local SQLite database for offline operations, workshop configuration settings linked to the device may become stale. The AWS Offline Sync Engine must mandate that all incoming offline reviews are re-validated against the current cloud state of the workshop settings inside Amazon RDS at the exact time of synchronization. If a conflict occurs due to a change in strict/relaxed mode permissions while offline, the cloud server's ruleset prevails.

---

### 🔄 Amendment - August 2026 (Refinement during Use Case Redaction)

- **Status:** Approved

- **Context:** During the detailed specification of core shop floor use cases (specifically UC-04 and UC-06), I realized that executing work reviews at a macro "Order" level introduces severe database locking bottlenecks and fails to capture fluid floor concurrency. Additionally, item categorization rules required flexibility to prevent administrative deadlocks in workshops with smaller headcounts.

- **Refined Decisions:**
  1. **Task-Level Finalization Matrix:** Work reviews are officially executed down at the atomic execution tier (`job_line`) and evaluated contextually at the task scope tier (`ticket_requested_categories`). The legacy "Order" container is eliminated. A macro `Ticket` is only promoted to `READY_FOR_BILLING` when 100% of its child requested categories pass their respective review gates.
  2. **The Objectivity Gate Toggle (`isTaskParticipationReviewAllowed`):** I have expanded the capability engine with an explicit configuration flag to assess technician team bias:
     - *Strict Mode (Default):* The validation engine blocks a reviewer if they logged *any* row in `job_line` inside that entire task category scope to prevent collaborative blind spots.
     - *Relaxed Mode:* Participation rules drop back to individual line-level isolation, allowing a mechanic to inspect a task scope they contributed to, provided they strictly bypass rows they personally authored.