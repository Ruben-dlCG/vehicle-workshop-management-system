## 🏁 Definition of Done (DoD) Checklist

A user story or feature branch is only considered **"Done"** and ready to merge when it satisfies both the automated CI pipeline and the following operational quality constraints:

### ⚙️ Technical & Automation Boundary
- [ ] **Compilation:** Code compiles locally and passes the GitHub Actions remote build runner.
- [ ] **Test Coverage:** All associated JUnit 5 and MockMvc integration tests pass with zero failures.

### 🏛️ Architecture & Clean Code Alignment
- [ ] **Rich Domain Model Encapsulation:** Business logic and data calculations reside inside Rich Entities; Services function purely as transactional coordinators.
- [ ] **Database Integrity:** Critical multi-step data mutations are explicitly wrapped in Spring `@Transactional` boundaries.

### 📋 Documentation & Traceability
- [ ] **Schema Synchronization:** Any modifications to database tables or relationships are manually mirrored inside the master Lucidchart EER tab.
- [ ] **ADR Logging:** High-impact architectural deviations or trade-off decisions are logged inside a dedicated Architecture Decision Record (see relevant folder [here](https://github.com/Ruben-dlCG/vehicle-workshop-management-system/tree/main/docs/architecture/adrs)).
- [ ] **UI Polish:** The interface has been manually verified using browser responsive layouts to guarantee operational usability for both office desktop views and mechanic shop-floor tablet screens.
