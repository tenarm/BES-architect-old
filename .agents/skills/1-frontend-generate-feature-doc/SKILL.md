---
name: frontend-generate-feature-doc
description: "LIFECYCLE STEP 1: Generate the frontend feature specification document (frontend.md) for a BES feature."
---

# Skill: Frontend Generate Feature Doc
**Lifecycle Position: STEP 1 of 6 — Requirement**
**Feeds into:** `2-backend-generate-feature-doc`

This skill generates the canonical frontend feature specification. Its output (`frontend.md`) is the primary input for the backend planning skill and every downstream review skill.

---

## Lifecycle Context
```
[YOU ARE HERE]
STEP 1: frontend-generate-feature-doc  →  frontend.md
STEP 2: backend-generate-feature-doc   →  backend.md
STEP 3: feature-plan-reviewer          →  changes.md
STEP 4: master-implementation-architect →  master-implementation-roadmap.md
STEP 5: feature-implementation-orchestrator → CODE
STEP 6: post-implementation-documenter →  implementation-manual.md
```

---

## About the BES Architecture
- **Stack**: React (Nx Monorepo) frontend, FastAPI (PDM Monorepo) backend, PostgreSQL.
- **Shell UI**: `apps/shell` dynamically loads licensed module libraries via `ComponentRegistry.registerLazy()`.
- **Module Libraries**: Each module lives in `bes-frontend/libs/<module>/`, registered via `init<Module>Module()`.
- **Shared UI**: ALL components come from `@bes/shared-ui` (Table, Drawer, Skeleton, Timeline, ProcessPipeline, etc.).
- **Licensing**: `GET /api/v1/bootstrap` tells Shell which modules are `active` or `readonly`. UI must degrade gracefully for `READONLY_EXTENSIONS`.
- **Known Modules**: `core`, `finance`, `sales`, `inventory`, `hr`, `supply-chain`, `crm`, `settings`.

---

## Instructions for the Assistant

When the user asks to "frontend generate feature doc":

1. Ask the user for any specific details or source code to analyze first.
2. Generate the 14-section document below.
3. Save to `features-plan/<module-name>/<feature-name>/frontend.md`. Create the folder if needed.

---

## Required 14-Section Structure

## 1. Module
Parent BES module (e.g., `finance`, `sales`, `inventory`, `hr`, `supply-chain`, `settings`).

## 2. Name
Official name of the feature (e.g., "Chart of Accounts Manager").

## 3. Description
Business purpose and value within the BES system.

## 4. Depends On
- **Core Kernel:** How `BESBase`, Event Bus, RBAC, and Bootstrap support this feature.
- **Cross-Module:** Other BES modules this feature relies on.
- **UI Libraries:** Required `@bes/shared-ui` components and Nx libs.

## 5. Feature UI — The UI as Information
**Feature Name:** [Sub-feature name]
**UI Details:** Table columns, form inputs, tree views, empty/loading/error states.
**User Journey:** Step-by-step UX walkthrough.

## 6. Sample Data Structure (YAML + JSON)
- Include `subsidiary_id` (UUID) on all entities.
- Use string Decimals for currency (e.g., `"150.0000"`) — Money Rule (4 d.p.).
- Include `metadata_` JSONB field where Ghost Foreign Keys or custom fields apply.

## 7. Required APIs
All RESTful endpoints. **All responses MUST use the envelope:**
```json
{ "status": "success", "data": { ... }, "metadata": { "count": 100, "page": 1 }, "error": null }
```

## 8. Database Tables & Architecture
- **Foundation**: All tables inherit `BESBase` → `id` (UUID PK), `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `metadata_` (JSONB).
- **Hub-and-Spoke MDM**: Master data (`customers`, `vendors`, `products`, `uoms`) in `core` schema. Module tables FK to core. Extension tables (e.g., `sales_customer_details`) for module-specific fields.
- **Precision**: `Numeric(20,4)` for all financial columns.
- **File Storage**: Centralized Storage Service (no BLOB columns).

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits**: `UPPER_SNAKE_CASE` events (e.g., `FINANCE_ACCOUNT_CREATED`).
- **Listens To**: Events triggering SSE UI updates.

## 10. Business Rules & Validations
- Soft Deletes: `is_deleted = True`. Physical deletion is FORBIDDEN.
- Money Rule: 4 d.p. on backend and frontend (`decimal.js`/`big.js`).
- Domain-specific constraints (hierarchy depth, referential integrity, etc.).

## 11. Security, Audit, and RBAC
- **Permission Format**: `<module>:<resource>:<action>` (e.g., `finance:coa:write`).
- **Context-Aware RBAC**: `manual` (user) vs. `auto_trigger` (system via `elevate_context()`).
- **Licensing Mode**: Exact UI degradation for `READONLY_EXTENSIONS`.
- **Audit Trail**: Actions to log (`user_id`, `timestamp`, `previous_state`, `new_state`).

## 12. Process Transparency & Workflow Pipeline
- **Home Dashboard Pipeline**: Does this feature surface pending items (approvals, drafts)?
- **Right Panel (Drawer) UX**:
  - **Macro View (`ProcessPipeline`)**: Horizontal tracker at top of Drawer (e.g., Draft → Pending → Approved).
  - **Micro View (`Timeline`)**: Vertical activity feed in a "History" secondary tab.
- **Shell Navigation**: Sidebar placement and breadcrumb path.

## 13. Technical Implementation Roadmap (Day 1)
- **Phase 1**: `BESBase` models + `SQLModel.metadata.create_all()`.
- **Phase 2**: Service layer + REST endpoints (pagination + RBAC).
- **Phase 3**: Nx library + `ComponentRegistry.registerLazy()` + Shell registration.
- **Phase 4**: UI with `@bes/shared-ui` + SSE state updates.
- **Phase 5**: Pub/Sub event wiring.

## 14. Verification & QA Strategy
- Subsidiary isolation check.
- Money Rule (4-decimal) check.
- `READONLY_EXTENSIONS` UI degradation check.
- Bootstrap permission verification.
- 3–5 critical user scenarios.
- Event → SSE UI update integration test.

---

### Execution Rules
- **Output file**: `features-plan/<module-name>/<feature-name>/frontend.md`
- **Next step**: Run `2-backend-generate-feature-doc` to generate `backend.md`.
