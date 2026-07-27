# ADR 1: Selection of PostgreSQL as primary RDBMS

## ℹ️ Context
The application manages high-volume transactional data (tickets, order lines, inventory). While I have foundational experience with MySQL and Oracle from academic coursework, PostgreSQL was selected for this project.

## 📍 Decision
Implement PostgreSQL 15+ containerized via Docker as the primary relational database management system.

## ⚖️ Consequences & Justification
- **Industry Alignment:** PostgreSQL is the modern enterprise standard for robust, transactional backend services, filling a key gap in my technical profile.
- **Cloud Readiness:** PostgreSQL integrates natively and optimally with Amazon RDS, streamlining the Phase 2 cloud migration strategy.
- **ORM abstraction:** Utilizing Spring Data JPA (Hibernate) minimizes syntax friction, allowing me to focus on relational design while gaining exposure to PostgreSQL ecosystems.
