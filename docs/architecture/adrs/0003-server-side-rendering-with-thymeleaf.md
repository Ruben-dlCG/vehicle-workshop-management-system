# ADR 3: Server-Side Rendering via Thymeleaf for User Interfaces

## ℹ️ Context
The application must deliver functional, responsive user interfaces tailored for workshop laptops and mechanics' tablets. I have not yet undertaken my formal mobile development coursework, creating a major frontend tooling gap for the summer phase.

## 📍 Decision
I have chosen to utilize the Model-View-Controller (MVC) pattern natively supported by Spring Boot, using Thymeleaf to handle server-side HTML/CSS rendering.

## ⚖️ Consequences & Justification
- **Scope Control:** Utilizing Thymeleaf completely eliminates the overhead of learning and managing complex frontend build pipelines (npm, JavaScript SPA frameworks) or mobile compilers during the summer.
- **Rapid Prototyping:** I can build fully functional forms and tables rapidly by combining simple HTML with responsive CSS grid frameworks (Bootstrap/Tailwind via CDN).
- **Migration Strategy:** This choice serves as a temporary, stable bridge. In Phase 2, this specific presentation layer will be replaced by a native mobile application talking to a REST API.
