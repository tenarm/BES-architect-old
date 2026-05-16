---
name: frontend-generate-feature-doc
description: This skill instructs the AI assistant on how to generate a comprehensive, structured frontend feature documentation markdown file for any upcoming feature in the BES project.
---

# Skill: Frontend Generate Feature Doc

This skill instructs the AI assistant on how to generate a comprehensive, structured frontend feature documentation markdown file for any upcoming feature in the BES project.

---

## Instructions for the Assistant

When the user asks to "frontend generate feature doc" or "create a frontend feature doc" for a specific feature, you MUST strictly follow this 14-section structure to ensure architectural consistency and to provide a complete implementation plan for the feature. 

Before generating, ask the user for any specific details they want included, or if they have source code you should analyze first. 

**IMPORTANT: File Location**
You MUST write the generated markdown file directly into the feature's subfolder within the module.
Format: `features-plan/<module-name>/<feature-name>/frontend.md`
Create the `<feature-name>` folder if it does not already exist.

Once ready, generate the markdown using the following structure:

### Required Structure

## 1. Module
State the parent module (e.g., Core, Finance, Inventory, HR).

## 2. Name
The official, formalized name of the feature.

## 3. Description
A detailed explanation of what the feature does, its core purpose, and its business value within the BES.

## 4. Depends on
List cross-module dependencies this feature relies on (e.g., Sales relies on Inventory; Supply Chain relies on Inventory).
- **Core Integration:** Briefly state how the Core module supports these dependencies. (e.g., "Core provides the unified `BESBase` PostgreSQL foundation, allowing Sales and Inventory to share the same physical database while maintaining logically isolated schemas like `sales_orders` and `inventory_items`").
- **Other Dependencies:** List UI libraries (e.g., `@bes/shared-ui`), required prior setups, or external APIs.

## 5. Feature name, details - the UI as information
**Feature Name:** [Sub-feature name]
**Details & UI Information:** 
Describe the UI elements in detail. Mention table columns, form inputs, tree views, interactive elements, empty states, and loading states. 

**User Journey & UX Flow:**
Provide a step-by-step walkthrough of how a user interacts with this feature. 
Example: "User navigates to Finance > AP -> Clicks 'New Invoice' -> Selects Vendor (triggering term auto-fill) -> Adds line items -> Submits for approval."

## 6. YAML or sample data structure
Provide a YAML schema representing the entity model, followed by a sample JSON payload representing the data state needed for this feature to render on the frontend.
- **IMPORTANT**: Always include `subsidiary_id` for multi-org scoping.
- **IMPORTANT**: Use string-based Decimals for currency fields (e.g., `"150.0000"`) to follow the "Money Rule" (4 decimal places).

## 7. Required APIs
List the RESTful endpoints (GET, POST, PUT, DELETE) required to drive the UI. Include payloads, descriptions, and critical validation rules.
- **IMPORTANT**: Enforce the Standardized API Response Structure. All endpoints MUST return the envelope: `{ "status": "success", "data": { ... }, "metadata": { ... }, "error": null }`.

## 8. Database Tables & Architecture
Specify the exact database schema (tables, column names, data types, primary/foreign keys). 
- **Foundation**: All tables MUST inherit from `BESBase` (providing `id`, `created_at`, `updated_at`, `created_by`, `is_deleted`, `metadata_`).
- **Master Data Management (Hub-and-Spoke)**: 
  - **Shared Hub:** Global entities (Master Data like `customers`, `vendors`, `products`, `uoms`) MUST live in the shared `core` schema and act as the single source of truth.
  - **Isolated Spokes:** Transactional tables (e.g., `sales_orders`, `finance_invoices`) MUST live in logically isolated schemas (e.g., `sales`, `finance`) but reference the core master data via Foreign Keys.
  - **Module Extensions:** If a module requires specific fields for a master entity (e.g., Sales needs a `credit_limit` for a customer), do NOT pollute the core table. Create an extension table in the module's schema (e.g., `sales_customer_details`) with a foreign key to the core entity.
- **Scoping**: Include `subsidiary_id` (UUID/String) on every table for organizational isolation.
- **Precision**: Enforce `Numeric(20,4)` for all financial/money columns per the "Money Rule".
- **File Storage**: If the feature requires file attachments, explicitly state that it MUST use the Centralized Storage Service (no custom blob columns).
- **Schema**: Explicitly state which module's database schema these tables belong to (e.g., Finance Module, Core Module).

## 9. Events & Real-Time Updates (Pub/Sub)
Align with the BES real-time SSE architecture. List:
- **Emits:** Events published when actions happen in this feature.
- **Listens To:** Events this feature reacts to (e.g., to update UI state dynamically).

## 10. Business Rules & Validations
Document strict constraints, domain logic, hierarchy depth limits, deletion protections, and inheritance rules.
- **Soft Deletes**: Physical deletion is FORBIDDEN. All entities must use the `is_deleted` flag for logical deletion.
- **Decimal Precision**: Enforce 4 decimal places for all financial calculations on both backend and frontend.

## 11. Security, Audit, and RBAC
- **Roles:** Define user roles (e.g., Admin, Viewer) and what they can do.
- **Read-Only Licensing Mode:** Define how the UI gracefully degrades (e.g., hiding action buttons, disabling forms) if the active tenant has only a `READONLY_EXTENSIONS` license for this module.
- **Audit Trail:** State what actions must be logged in the central BES audit system (tracking `user_id`, `timestamp`, `previous_state`, `new_state`).

## 12. Process Transparency & Workflow Pipeline
To eliminate the "blackbox" nature of background processes and show architectural traceability as a Solution Architect, document how this feature integrates into the global workflow:
- **Pending Pipeline (Home Dashboard):** Does this feature generate actionable items (e.g., pending approvals, drafts waiting for review) that need to be surfaced on the user's Home pending pipeline? If so, what are the triggers?
- **Process UI Integration:** Specify *when* and *where* to use the transparency components. If the feature uses a Right Panel (Drawer) for forms, apply this UX pattern:
  - **Macro View (`ProcessPipeline`):** Use the horizontal tracker anchored at the **top of the Drawer** (above the form) to show the high-level status (Draft → Pending → Approved). This also serves as the action center for inline approvals.
  - **Micro View (`Timeline`):** Use the vertical activity feed placed inside a **secondary "History/Activity" tab** within the Drawer body. This prevents the dense 5Ws audit data (Who, What, When, Why) from cluttering the editable form details.
- **Traceability:** Draft which specific parts of this feature require process chain visualization.
- **Shell UI & Navigation:** Specify where this feature should be placed in the global sidebar menu and any breadcrumb paths.

## 13. Technical Implementation Roadmap
Provide a phased "Day 1" implementation plan:
- **Phase 1: Backend Foundation**: Define the `BESBase` models and migrations.
- **Phase 2: Core Logic & APIs**: Develop the service layer and REST endpoints.
- **Phase 3: Frontend Infrastructure**: Create the NX library (if needed) and register the feature in the shell.
- **Phase 4: UI Development**: Build the screens using `@bes/shared-ui`.
- **Phase 5: Event Integration**: Implement Pub/Sub logic for real-time updates.

## 14. Verification & QA Strategy
Define the success criteria and testing scenarios:
- **Scoping Check**: How will we verify `subsidiary_id` data isolation?
- **Precision Check**: How will we verify the "Money Rule" (4-decimal rounding)?
- **Functional Scenarios**: List 3-5 critical user paths to test.
- **Integration Test**: Verify the specific Pub/Sub event flow.

### Execution Rules
- The final document should be saved in the corresponding feature directory: `features-plan/<module-name>/<feature-name>/frontend.md`.
- Use professional formatting and standard markdown syntax.
