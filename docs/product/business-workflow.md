# Business Workflow Overview

This file documents the workflow's overview in the workshop. Its goal is to show the different steps that define what a fully processed unit of work looks like, providing the business' macro view with its different possible scenarios.

## 📊 Interactive Model
The workflow structure is actively maintained and updated in Lucidchart to ensure accurate business flows, both before and during implementation.

👉 [Launch Interactive Workflow Diagram on Lucidchart](https://lucid.app/lucidchart/f388d9a9-3c39-4d37-ab1d-bd731008716f/edit?viewport_loc=-2149%2C105%2C5637%2C2367%2C0_0&invitationId=inv_f1b6dacb-1433-49cf-bbc2-da0b78180693)

## 🔑 Key Strategic Patterns Applied
- **Separation of Potential and Existing Clients:** A potential Client is one that has not yet accepted to have any work done in their vehicle, thus not generating a record in the `client` DB table. Conversely, Clients do have at least one initialized `order` linked to them, so if they were to return to the workshop in another occasion, their data would already be stored in the system (in compliance with applicable regulations). A potential Client's data may be exclusively stored via an application form (`appointment`), prior to their conversion to `Client`.
- **Sequential Entity Lifecycles:** An `invoice` is born from an `order`. An `order` is born from a `ticket`. All three are tightly coupled, as their existence is only possible when created in this mentioned order.
- **Edge-Cases handling:** Different routes are contemplated depending on `Client`'s feedback, quality of work or potential human mistakes at different points of the workflow, allowing to dynamically catch scenarios diverging from the normal "happy path".
- **Flexible Quality Assurance:** Each `Order Line` needs to be signed off by a `mechanic` before an `Order` is finalized. This can be any `mechanic` holding the same or higher rank compared to the underlying `item`'s inherent difficulty, provided that it's a different person than the one that logged the line.
