# Vehicle Lifecycle & Workshop Relational Schema

This file documents the data layer constraints and entity relationships driving the logistics workflow.

## 📊 Interactive Model
The entity-relationship structure is actively maintained and updated in Lucidchart to map data flows before implementation.

👉 [Launch Interactive EER Diagram on Lucidchart](https://lucid.app/lucidchart/6a210dec-8a84-4b0c-881a-b9db306e6c1c/edit?viewport_loc=-550%2C-1110%2C3357%2C1542%2C0_0&invitationId=inv_65ed33ba-0e84-44a3-8c7d-8cff54ffcd8e)

## 🔑 Key Strategic Patterns Applied
- **UUID Implementations:** Taking into consideration Phase 2 of this project, primary keys utilize globally unique 36-character identifiers to guarantee future data sync scalability without synchronization overlap collisions.
- **Joined Inheritance:** The `person` entity acts as an abstract core shared across `customer` and `mechanic` subclasses, enforcing clean data boundaries at the schema level. Similarly, the `item` entity serves the same purpose for the `part` and `service` entities.
