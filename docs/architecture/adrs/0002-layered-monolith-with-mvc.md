# ADR 2: Layered Monolith with MVC as architectural patterns

## ℹ️ Context
Before writing the application code, a decision needs to be made regarding how to organize and structure said code.

## 📍 Decision
Structure the application code as a layered monolith, in combination with the MVC architectural pattern.

## ⚖️ Consequences & Justification
- **Simplicity:** Choosing a monolith instead of a microservices architecture is sufficient for the application goals and its functionalities, reducing deployment overhead simultaneously.
- **Architectural Complementarity:** This design combines the Layered Pattern and the MVC Pattern, which sit at different abstraction levels. The Layered structure organizes the internal responsibilities of the backend, while MVC isolates the user's interaction pipeline.
- **Separation of Concerns:** By ensuring that division boundaries are not strictly tied to physical folders, the business logic (Services) remains decoupled from both the presentation routers (Controllers) and the data presentation formats (Views). This allows entities (Model) to be safely shared across layers without creating structural coupling.
- **Refactoring Runway:** This clear separation guarantees that the underlying Java business services and database data layers can remain 100% untouched when the Thymeleaf View engine is replaced by a REST API framework in Phase 2.
