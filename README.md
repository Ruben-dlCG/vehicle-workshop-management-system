# vehicle-workshop-management-system
A robust internal management system engineered to streamline car workshop logistics—from customer intake to final invoicing. Focus on modular design and separation of concerns. This is a self-driven, architectural exploration project designed to meet production-grade standards and real-world business requirements.

## 🛠️ Tools
Java 21, Spring ecosystem, Thymeleaf, JUnit & MockMvc (applying ISTQB principles for testing), PostgreSQL, Docker & AWS (for deployment and scalability).

## 🧭 Current Focus (Summer 2026, MVP)
  - **Layered Architecture**: Enforcing strict separation of concerns by organizing the backend into distinct Controller, Service, and Data layers.
  - **Monolithic MVC Pattern**: Delivering user interfaces for laptops and tablets using Spring Boot and Thymeleaf templates for rapid, reliable local development.
  - **Security & Roles**: Implemented Role-Based Access Control (RBAC) via Spring Security to separate supervisor functions from mechanic tasks.
  - **ISTQB-Grounded Testing**: Validating core billing and workflow logic using JUnit 5 and MockMvc integration tests.

## 🚀 Future Roadmap (Academic year 2026-2027, Evolution)
  - **API Transformation**: Refactoring Thymeleaf controllers into a decoupled REST API serving JSON payloads.
  - **Offline-Ready Data Sync**: Leveraging UUID-based entities designed this summer to support a local, offline-first mobile application (DAM).
  - **Cloud Deployment**: Migrating the local Docker container environment to AWS (RDS and ECS).