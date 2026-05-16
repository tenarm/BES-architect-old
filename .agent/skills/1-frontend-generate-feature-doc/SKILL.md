---
name: frontend-generate-feature-doc
description: This skill instructs the AI assistant on how to generate a comprehensive, structured frontend feature documentation markdown file for any upcoming feature in the BES project.
---

# Skill: Frontend Generate Feature Doc

This skill instructs the AI assistant to generate frontend feature documentation aligned with the BES Factory's Kernel-and-Plugin (Core + Extension) architecture, React Nx Shell UI, and all guiding architectural principles.

---

## About the BES Architecture

- **Stack**: React (Nx Monorepo) frontend, FastAPI (PDM Monorepo) backend, PostgreSQL.
- **Shell UI**: A single `apps/shell` that dynamically loads licensed module libraries via the `ComponentRegistry`.
- **Module Libraries**: Each module lives in `bes-frontend/libs/<module>/` and is registered via `init<Module>Module()`.
- **Shared UI**: All components (Button, Table, Skeleton, Drawer, Timeline, ProcessPipeline) come from `@bes/shared-ui`.
- **Licensing**: The `/api/v1/bootstrap` endpoint tells the Shell which modules are `active` or `readonly`. UI must degrade gracefully for `READONLY_EXTENSIONS`.

---

## Instructions for the Assistant

When the user asks to "frontend generate feature doc" or "create a frontend feature doc," you MUST strictly follow this 14-section structure.

Before generating, ask the user for any specific details they want included, or if they have source code to analyze first.

**IMPORTANT: File Location**
Write the generated markdown file into the feature's subfolder:
Format: `features-plan/<module-name>/<feature-name>/frontend.md`
Create the `<feature-name>` folder if it does not already exist.

---

## Required 14-Section Structure

## 1. Module
State the parent BES module (e.g., `finance`, `sales`, `inventory`, `hr`, `supply-chain`, `settings`).

## 2. Name
The official, formalized name of the feature (e.g., "Chart of Accounts Manager").

## 3. Description
A detailed explanation of the feature's business purpose and value within the BES system.

## 4. Depends On
List cross-module dependencies (e.g., Sales relies on `core` Customers; Supply Chain relies on Inventory Items).
- **Core Kernel:** State how the Core supports this (e.g., "`BESBase` provides UUID PKs, audit trail, `subsidiary_id`, and `metadata_` JSONB expansion").
- **Other Dependencies:** List required `@bes/shared-ui` components, other module libraries, or external APIs.

## 5. Feature UI — The UI as Information
**Feature Name:** [Sub-feature name]
**Details & UI Information:**
Describe all UI elements: table columns, form inputs, tree views, interactive states (loading, empty, error).

**User Journey & UX Flow:**
Step-by-step walkthrough (e.g., "User navigates Finance > COA → Clicks 'New Account' → Fills form in Drawer → Submits → Table refreshes via SSE event").

## 6. Sample Data Structure (YAML + JSON)
Provide a YAML schema then a sample JSON API response payload.
- **ALWAYS** include `subsidiary_id` (UUID) for multi-org scoping.
- **ALWAYS** use string-based Decimals for currency (e.g., `"150.0000"`) per the Money Rule (4 d.p.).
- Include the `metadata_` JSONB field where Ghost Foreign Keys or custom fields are needed.

## 7. Required APIs
List all RESTful endpoints (GET, POST, PUT, DELETE) needed by the UI.
- **Response Envelope:** ALL endpoints MUST return:
  ```json
  { "status": "success", "data": { ... }, "metadata": { "count": 100, "page": 1 }, "error": null }
  ```
- State if the endpoint is consumed by bootstrap (`GET /api/v1/bootstrap`) or SSE stream.

## 8. Database Tables & Architecture
Specify the exact table schema (columns, types, PKs, FKs).
- **Foundation**: All tables MUST inherit `BESBase` → gets `id` (UUID PK), `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `metadata_` (JSONB).
- **Hub-and-Spoke MDM**:
  - **Core Hub:** Global master data (`customers`, `vendors`, `products`, `uoms`) lives in the `core` schema. Cross-module extension data uses `metadata_` Ghost Foreign Keys or extension tables (e.g., `sales_customer_details`).
  - **Module Spoke:** Transactional tables (e.g., `finance_invoices`) live in the module schema but FK to core.
- **Scoping**: `subsidiary_id` on every table for intra-tenant isolation.
- **Precision**: `Numeric(20,4)` for all financial columns.
- **File Storage**: File attachments MUST use the centralized Storage Service (no BLOB columns in tables).
- **UOM**: Reference the `core.uoms` registry for all unit-of-measure values.

## 9. Events & Real-Time Updates (Pub/Sub)
Align with the BES async Event Bus (Day 1: asyncio in-memory; Day 2: persistent outbox/Redis).
- **Emits:** Events published on user actions (e.g., `FINANCE_ACCOUNT_CREATED`). Use `UPPER_SNAKE_CASE`.
- **Listens To:** Events this feature reacts to for real-time UI updates (e.g., SSE stream trigger).
- **Safety Note:** If the subscribing module is not licensed, events expire silently.

## 10. Business Rules & Validations
- **Soft Deletes**: Physical deletion is FORBIDDEN. Use `is_deleted = True` exclusively.
- **Money Rule**: 4 decimal places enforced on backend (`Numeric(20,4)`) and frontend (`decimal.js`/`big.js`).
- Hierarchy depth limits, referential integrity, and other domain-specific constraints.

## 11. Security, Audit, and RBAC
- **Permission Format**: `<module>:<resource>:<action>` (e.g., `finance:coa:write`).
- **Context-Aware RBAC**: Distinguish between `manual` (user-initiated) and `auto_trigger` (system-initiated via Permission Elevation / `elevate_context()`).
- **Licensing Mode**: Define exact UI degradation for `READONLY_EXTENSIONS` (e.g., hide action buttons, disable forms, show read-only badge).
- **Bootstrap**: State which permissions flow through `GET /api/v1/bootstrap` to the Shell.
- **Audit Trail**: List actions to log: `user_id`, `timestamp`, `previous_state`, `new_state`.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline (Home Dashboard):** Does this feature surface actionable items (pending approvals, drafts) on the Home page pipeline?
- **Right Panel (Drawer) UX Pattern:**
  - **Macro View (`ProcessPipeline`):** Horizontal status tracker at the **top of the Drawer** (e.g., Draft → Pending Approval → Posted). Also serves as inline approval action center.
  - **Micro View (`Timeline`):** Vertical activity feed in a **secondary "History" tab** within the Drawer. Shows 5W audit data (Who, What, When, Where, Why) without cluttering the form.
- **Shell Navigation:** Where does this feature appear in the sidebar? What is the breadcrumb path?

## 13. Technical Implementation Roadmap (Day 1)
- **Phase 1: Backend Foundation** — `BESBase` models, `SQLModel.metadata.create_all()` auto-migration.
- **Phase 2: Core Logic & APIs** — Service layer, REST endpoints with pagination and RBAC.
- **Phase 3: Frontend Infrastructure** — Nx library generation, `ComponentRegistry.registerLazy()`, Shell registration in `main.tsx` and `app-config.tsx`.
- **Phase 4: UI Development** — Screens built with `@bes/shared-ui`, Drawer pattern, SSE state updates.
- **Phase 5: Event Integration** — Pub/Sub logic, real-time UI refresh.

## 14. Verification & QA Strategy
- **Subsidiary Isolation:** Verify data is filtered by `subsidiary_id` across tenants.
- **Money Rule Check:** Verify 4-decimal rounding on all financial inputs.
- **Licensing Check:** Verify UI degrades correctly under `READONLY_EXTENSIONS`.
- **Bootstrap Validation:** Verify permissions appear correctly in the bootstrap response.
- **Functional Scenarios:** List 3–5 critical user paths.
- **Event Integration Test:** Verify Pub/Sub event triggers the expected SSE UI update.

---

### Execution Rules
- Save to: `features-plan/<module-name>/<feature-name>/frontend.md`
- Use professional formatting and standard markdown syntax.
