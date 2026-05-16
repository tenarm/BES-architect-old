---
name: backend-generate-feature-doc
description: This skill instructs the AI assistant to read an existing UI feature document and generate a detailed backend extension implementation plan.
---

# Skill: Backend Generate Feature Doc

This skill instructs the AI assistant to read a UI Feature Document (e.g., `features-plan/<module-name>/<feature-name>/frontend.md`) and generate a detailed, structured backend implementation plan that adheres strictly to the BES Factory Architecture Rules.

---

## Instructions for the Assistant

When the user asks you to "backend generate feature doc based on a feature doc," follow these instructions strictly.

### Prerequisites

1. Ask the user for the path to the UI Feature Document if they haven't provided it (e.g., `features-plan/finance/coa/frontend.md`).
2. Read the provided feature document carefully. Pay attention to the required APIs, data structures, and business logic.
3. Review the standard architectural rules defined in `.agent/rules/backend.md`, `.agent/rules/security.md`, and `.agent/rules/architecture.md`.

**IMPORTANT: File Location**
You MUST write the generated backend implementation plan directly into the feature's subfolder within the module.
Format: `features-plan/<module-name>/<feature-name>/backend.md`
Create the `<feature-name>` folder if it does not already exist.

### Output Format

Generate a comprehensive Backend Implementation Plan using the following structure. 

## 1. Module Overview
- **Extension Name:** The `snake_case` name for the backend extension.
- **Goal:** A brief summary of what the backend will accomplish based on the feature doc.

## 2. API Design & Payloads
List all required API endpoints. For each endpoint, detail:
- **Route:** The precise API path (e.g., `POST /api/v1/finance/coa`).
- **Input (Schemas):** The incoming payload schema (Pydantic `*Create` models). **NEVER** include `id`, `created_at`, `is_deleted`, or `subsidiary_id` in input schemas.
- **Desired Output:** The exact `StandardResponse` envelope (`success_response` or `paginated_response`), along with the `*Read` schema output fields.
- **Pagination:** Explicitly note if the endpoint requires `PaginationParams`.

## 3. Business Logic (Services Layer)
Describe the core logic that belongs in `services.py`:
- **Validation:** Specific checks (e.g., amounts must be positive, referencing valid core entities).
- **Transactions:** How the database operations will be handled safely within the `AsyncSession`.
- **The Money Rule:** Explicitly note the use of `Decimal` for all financial data calculations.

## 4. RBAC & Security Considerations
Detail the permissions required for the endpoints:
- **Role Permissions:** The exact permission strings (e.g., `"finance:coa:write"`) that will be enforced via `require_permission()`.
- **Subsidiary Scoping:** Confirm that the tables inherit from `BESBase` and that all queries will inherently scope to the `subsidiary_id` of the current context.

## 5. Audit Trail & Soft Deletion
State how tracking and deletion will be handled:
- Confirm that the `is_deleted` flag will be used instead of physical deletion.
- Note the automatic tracking of `created_at`, `updated_at`, and `created_by` via `BESBase`.

## 6. Events & Pub/Sub
- **Events Emitted:** Identify what actions will trigger an event emission (e.g., `COA_CREATED`) and the exact payload structure.
- **Event Subscriptions:** Identify what events this module must listen to from other modules.

## 7. Implementation Roadmap
Provide a step-by-step checklist to guide the actual coding:
1. Initialize the extension directory structure.
2. Define the `BESBase` models (`models.py`).
3. Define the Pydantic schemas (`schemas.py`).
4. Implement the service layer business logic (`services.py`).
5. Wire the HTTP endpoints (`router.py`) with pagination and RBAC.
6. Implement pub/sub event emitters/handlers (`events.py`).
7. Create the `ExtensionManifest` (`manifest.py`).
8. Add to Core permissions (`admin_permissions.json`).

### Execution Rules
- The final document should be saved in the corresponding feature directory: `features-plan/<module-name>/<feature-name>/backend.md`.
- Use professional formatting and standard markdown syntax.
