---
name: feature-implementation-orchestrator
description: "LIFECYCLE STEP 5: Read the master-implementation-roadmap.md and execute the end-to-end implementation of a feature using the full context pyramid."
---

# Skill: Feature Implementation Orchestrator
**Lifecycle Position: STEP 5 of 6 — Build**
**Reads from:**
- `features-plan/master-implementation-roadmap.md` (from Step 4c) — determines WHICH feature to build and in what order.
- `features-plan/project-readiness-report.md` (from Step 4b) — enforces global Golden Rules.
- `features-plan/<module>/module-cross-features-changes.md` (from Step 4a) — shared services & dependencies.
- `features-plan/<module>/<feature>/changes.md` (from Step 3) — the implementation spec.
- `features-plan/<module>/<feature>/frontend.md` + `backend.md` (from Steps 1 & 2) — domain nuances.

**Triggers:** `util-create-extension`, `util-create-ui-module`, `util-add-api-endpoint` (sub-routines)
**Feeds into:** `post-implementation-documenter` (Step 6)

This is the execution engine. It takes the context pyramid and builds the actual code in the correct architectural order.

---

## Lifecycle Context
```
STEP 1: frontend-generate-feature-doc  →  frontend.md
STEP 2: backend-generate-feature-doc   →  backend.md
STEP 3: feature-plan-reviewer          →  changes.md
STEP 4a: module-architect-reviewer     →  module-cross-features-changes.md
STEP 4b: project-readiness-auditor     →  project-readiness-report.md
STEP 4c: master-implementation-architect →  master-implementation-roadmap.md
[YOU ARE HERE]
STEP 5: feature-implementation-orchestrator → CODE
STEP 6: post-implementation-documenter →  implementation-manual.md
```

---

## Instructions for the Assistant

When the user asks to "implement the feature" for a specific step in the roadmap:

### 1. Full Context Preparation
Read the Context Pyramid in this order:
1. `features-plan/master-implementation-roadmap.md` — find the current Step N, its status, source docs, and skill to use.
2. `features-plan/project-readiness-report.md` — confirm Golden Rules and global architectural constraints.
3. `features-plan/<module>/module-cross-features-changes.md` — identify shared services, event catalog, and whether a shared base was planned first.
4. `features-plan/<module>/<feature>/changes.md` — the implementation spec (backend files, frontend files, integration checklist, implementation order).
5. `features-plan/<module>/<feature>/frontend.md` + `backend.md` — recover any nuanced business rules or UI details not captured in `changes.md`.

**If contradictions exist**: Architecture (project-readiness-report) > Execution (changes.md) > Details (frontend/backend md).

### 2. Environment Check
- Does `bes-backend/extensions/<module>/` exist? If NO → use `util-create-extension`.
- Does `bes-frontend/libs/<module>/` exist? If NO → use `util-create-ui-module`.
- Is this adding to an existing extension? → use `util-add-api-endpoint` for new routes.

### 3. Backend Implementation (Section 4 of `changes.md`)
In strict order — DO NOT skip steps:

1. **Models** (`models.py`):
   - Inherit `BESBase`. Confirm `subsidiary_id`, `metadata_`, `is_deleted`.
   - Apply Money Rule: `Column(Numeric(precision=20, scale=4))` for all financial fields.

2. **Schemas** (`schemas.py`):
   - `*Create`: No `id`, `created_at`, `is_deleted`, `subsidiary_id`.
   - `*Update`: All fields Optional.
   - `*Read`: Full output including computed fields.

3. **Services** (`services.py`):
   - Business logic only. No HTTP. No response wrapping.
   - Transaction safety: single `AsyncSession` commit per operation.
   - Soft delete: `is_deleted = True`. NEVER delete from DB.
   - Money Rule: `Decimal` for all arithmetic.

4. **Router** (`router.py`):
   - Thin layer. Call service, return response. No logic.
   - ALL list endpoints: add `PaginationParams` dependency.
   - ALL write/sensitive endpoints: add `require_permission("<module>:<resource>:<action>")`.
   - ALL responses: use `success_response()` or `paginated_response()`.

5. **Events** (`events.py`):
   - Emitters: `UPPER_SNAKE_CASE` event type. Full payload.
   - Subscribers: registered via `event_bus.subscribe()`.

6. **Manifest** (`manifest.py`):
   - Register router, models, event handlers.

7. **Permissions** (`admin_permissions.json`):
   - Add `<module>:<resource>:<action>` keys.
   - Verify in `GET /api/v1/bootstrap` response.

### 4. Frontend Implementation (Section 4 of `changes.md`)

#### Step C: Frontend Implementation
1. **Component Inventory & Promotion Logic**: Identify all required UI components from `frontend.md`.
   - **Strict Promotion Rule**: ONLY promote a component to `@bes/shared-ui` if it is a **generic primitive** (e.g., custom DatePicker, StatusBadge, DataGrid) that will be reused by at least two other modules.
   - **Utility Trigger**: If a missing component meets the "Strict Promotion Rule", use `util-create-shared-component` to expand the shared library first.
   - **Module-Specific UI**: All feature-specific layouts, dashboards, and domain-heavy components MUST be built locally in `bes-frontend/libs/<module>/src/lib/<feature>/` or `src/lib/components/`.
2. **Component Development**: Build the UI using the expanded `@bes/shared-ui` toolkit.
   - **Visual Excellence**: Every component built MUST be premium, interactive, and high-fidelity (no placeholders).
   - **Transactional forms**: Use Drawer with `ProcessPipeline` at top (macro view) + "History" tab with `Timeline` (micro view).
   - Handle READONLY mode: hide/disable action buttons based on `bootstrap.permissions`.
3. **Registry**: Register the components in the module's `index.ts` using `ComponentRegistry.registerLazy()`.
4. **Shell Integration**: Register the module in the Shell's `main.tsx` and `app-config.tsx`.
   - Add nav item in the sidebar config.

### 5. Verification
Execute the Integration Checklist from `changes.md`:
- `GET /health` — module is loaded.
- `GET /api/v1/<module>/<resource>` — API returns data.
- Verify `subsidiary_id` isolation (data from one tenant not visible to another).
- Verify READONLY mode UI degradation.
- Verify permissions appear in Bootstrap response.
- Trigger a key user action and verify the SSE event fires and updates the UI.

### 6. Handover Summary
Provide a summary report:
- All files created/modified (with paths).
- Any deviations from `changes.md` and the reason.
- Any manual steps needed (e.g., database migrations, env var updates).
- Confirm: ready for Step 6 (`post-implementation-documenter`).

---

### Utility Sub-Routines (use when needed)
- **`util-create-extension`**: Scaffold a new backend extension module.
- **`util-create-ui-module`**: Scaffold a new Nx frontend library.
- **`util-add-api-endpoint`**: Add a new API route to an existing extension.

### Execution Rules
- **After completion**: Run `post-implementation-documenter` (Step 6) to close out the feature.
