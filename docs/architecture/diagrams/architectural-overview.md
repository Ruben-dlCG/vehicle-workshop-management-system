# Architectural Overview

This file blueprints the architectural foundation of the project. Its goal is to show the structural patterns that dictate how data flows from end to end, with a explicit focus on the different layers that shape the application schema, and the tools used to that effect. 

Note that, at this stage of development, two distinct phases have been planned. Consequently, the underlying architecture varies in some key aspects from phase 1 to phase 2.

## 📊 Interactive Model
The architectural structure is periodically maintained and subjected to be updated in Lucidchart, so as to ensure accurate representation of data models, processing boundaries and tool selection for each phase of the project.

👉 [Launch Interactive Architectural Diagram on Lucidchart](https://lucid.app/lucidchart/6a210dec-8a84-4b0c-881a-b9db306e6c1c/edit?viewport_loc=-322%2C-442%2C3380%2C1419%2CKJ1d39JUXv3b&invitationId=inv_65ed33ba-0e84-44a3-8c7d-8cff54ffcd8e)

## 🔑 Key Strategic Patterns Applied
- **Defined Project Phases:** Due to time limitations and current technical skill gaps, the lifecycle of this project has been split into two defined phases, each holding their own characteristics and supporting different functions. For additional details on this, please refer to the `README.md` file available [here](https://github.com/Ruben-dlCG/car-workshop-management-system/blob/main/README.md)
- **Layered Architecture:** The central piece of how this whole project is meant to be built lies upon the division of data in distinct layers of information. Each layer helps define the scope of the processed data at that specific point in the system. For example, a class named `InventoryService` is meant to be part of the Service Layer, as its only purpose is to handle core business logistics and coordinating data persistence.
- **Separation of Concerns:** Each layer and module is strictly responsible for one set of specific tasks, which helps on keeping a clean and highly maintainable system.

## 🔄 Operational Execution Examples (Unit of Work Lifecycle)

To fully illustrate the structural evolution between both project phases, the examples below detail how a single unit of work —checking and updating stock availability when a mechanic adds a brake pad to a vehicle— is executed across the two different blueprints.

### 🎨 Phase 1 Execution Lifecycle (Monolithic MVP)
1. **User Action:** A mechanic taps "Add Brake Pad" inside a shop-floor web view on a tablet connected to the local workshop Wi-Fi.
2. **Network Entry:** The browser issues a standard `HTTP POST` request containing form parameters directly to the local server engine.
3. **Perimeter Boundary:** Spring Security intercepts the request at the perimeter, confirming the authenticated user possesses `ROLE_MECHANIC` credentials.
4. **Presentation Routing:** A standard Spring `@Controller` catches the request parameters and delegates processing to the business layer.
5. **Business Orchestration:** The `InventoryService` coordinates the transaction by invoking the `PartRepository` to retrieve the active entity record from the database. Once loaded into memory, the service delegates execution to the entity's native `deductStock()` method, allowing the object to validate its own internal state boundaries and perform the deduction.
6. **Data Persistence:** Spring Data JPA translates the object mutation into a native `UPDATE` SQL query, committing the state change to the local **PostgreSQL database running inside Docker**.
7. **UI Compilation:** The Controller intercepts the database success signal, fetches the fresh inventory numbers, and feeds them into the **Thymeleaf Template Engine**.
8. **Server-Side Render:** Thymeleaf compiles a completely new, static HTML page on the server and ships it back across the local network. The tablet browser refreshes instantly to display the updated inventory count.

### 🚀 Phase 2 Execution Lifecycle (Distributed Cloud API)
1. **User Action:** A mechanic clicks "Add Brake Pad" inside a native mobile application while working deep inside a cellular blindspot or concrete service bay.
2. **Local Persistence:** The application does not freeze or crash due to network loss. The mobile app executes the business logic locally and logs the new item entry into the device's **embedded SQLite database** using a **UUID** data type.
3. **Network Reconnection:** Once the tablet re-enters an active Wi-Fi zone, a background service wakes up, pulls the compressed data payload out of SQLite, and initiates a secure connection over the internet.
4. **Cloud Entry:** The mobile client fires a compressed **JSON text payload** over `HTTPS` to the **Amazon API Gateway** perimeter.
5. **Security Verification:** The cloud perimeter inspects the request headers, validates a stateless **JSON Web Token (JWT)**, and securely routes the payload into an **Amazon ECS/Docker Container**.
6. **API Processing:** A specialized Spring `@RestController` receives the raw JSON text, maps it back into a Java object, and hands it directly to the dedicated **Offline Sync Engine**.
7. **Conflict Resolution:** The sync engine analyzes the incoming UUID, verifies timestamps to ensure no data conflicts have occurred on the central database, and updates the production-ready **Amazon RDS (PostgreSQL)** instance.
8. **Lightweight Response:** The backend returns a minimal, blazing-fast `200 OK` JSON response code back across the internet. The mobile app receives the token, marks its local SQLite sync cycle as complete, and updates its local screen without ever forcing a full page reload.

