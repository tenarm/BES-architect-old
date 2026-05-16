---
name: feature-implementation-orchestrator
description: This skill instructs the AI assistant to read a consolidated changes.md file and execute the end-to-end implementation of a feature.
---

# Skill: Feature Implementation Orchestrator

This skill instructs the AI assistant to take a `changes.md` (Consolidated Technical Specification) and execute the actual implementation across the backend and frontend, following the prescribed architectural order.

---

## Instructions for the Assistant

When the user asks you to "implement the feature" for a specific path:

### 1. Full-Context Preparation & Validation
- Navigate to the feature folder: `features-plan/<module>/<feature>/`.
- **Read the Full Context Pyramid**:
    1. **Project Level**: Read `features-plan/project-readiness-report.md` to understand global architectural mandates (The Golden Rules).
    2. **Module Level**: Read `features-plan/<module>/module-cross-features-changes.md` to identify shared services, logic, or dependencies that must be respected.
    3. **Technical Specs**: Read `features-plan/<module>/<feature>/changes.md` for the consolidated roadmap.
    4. **Feature Details**: Review the original `frontend.md` and `backend.md` to ensure no subtle business rules or UI nuances were lost during consolidation.
- **Cross-Reference**: If you find any contradictions between the docs, prioritize the `project-readiness-report.md` (Architecture) followed by `changes.md` (Execution).
- **Environment Check**:
    - Verify that the target module extension exists in `bes-backend/extensions/`. If not, you must use the `create-extension` skill.
    - Verify that the target Nx library exists in `bes-frontend/libs/`. If not, you must use the `create-ui-module` skill.

### 2. Execution Phase (Following the Roadmap)

Follow the **"Implementation Order"** defined in Section 4 of `changes.md`. You MUST NOT skip steps.

#### Step A: Backend Foundation
1. **Models**: Update `models.py`. Ensure `BESBase` inheritance and "The Money Rule" (Decimal precision).
2. **Schemas**: Update `schemas.py` with `*Create`, `*Update`, and `*Read` models.
3. **Services**: Implement the business logic in `services.py`. Ensure transaction safety and `subsidiary_id` awareness.
4. **Router**: Implement the FastAPI routes in `router.py`. Apply `require_permission` and `StandardResponse` envelopes.
5. **Events**: Implement emitters and subscribers in `events.py`.
6. **Manifest**: Ensure the feature is registered in `manifest.py`.

#### Step B: Security & Configuration
1. **Permissions**: Add the new permission keys to `core/core/admin_permissions.json`.
2. **Bootstrap**: Verify that the backend returns these permissions in the `/bootstrap` endpoint.

#### Step C: Frontend Implementation
1. **Component Development**: Build the UI components in the module's Nx library using `@bes/shared-ui`.
2. **Registry**: Register the components in the module's `index.ts` using `ComponentRegistry.registerLazy()`.
3. **Shell Integration**: Register the module in the Shell's `main.tsx` and `app-config.tsx`.

### 3. Verification & Handover
- Perform the "Integration Checklist" from `changes.md`.
- Run `GET /health` to ensure the module is loaded.
- Run `GET /api/v1/<module>/<resource>` to verify the new API.
- Provide a summary of all files created/modified and any manual steps the user needs to take (e.g., database migrations).
