---
name: post-implementation-documenter
description: "LIFECYCLE STEP 6: Audit the finished implementation code and generate implementation-manual.md — the definitive post-build technical reference for a feature."
---

# Skill: Post-Implementation Documenter
**Lifecycle Position: STEP 6 of 6 — Finalize**
**Reads from:**
- `bes-backend/extensions/<module>/` — the actual built Python code.
- `bes-frontend/libs/<module>/` — the actual built React components.
- `features-plan/<module>/<feature>/changes.md` — original spec, for deviation analysis.
- `features-plan/<module>/<feature>/frontend.md` + `backend.md` — original intent.

**Output:** `features-plan/<module>/<feature>/implementation-manual.md`

This is the final step of the lifecycle. It performs a "Code-to-Doc" audit of the finished implementation and generates the canonical technical reference for the feature — what was actually built, how it works, and how to maintain it.

---

## Lifecycle Context
```
STEP 1: frontend-generate-feature-doc  →  frontend.md
STEP 2: backend-generate-feature-doc   →  backend.md
STEP 3: feature-plan-reviewer          →  changes.md
STEP 4a: module-architect-reviewer     →  module-cross-features-changes.md
STEP 4b: project-readiness-auditor     →  project-readiness-report.md
STEP 4c: master-implementation-architect →  master-implementation-roadmap.md
STEP 5: feature-implementation-orchestrator → CODE
[YOU ARE HERE]
STEP 6: post-implementation-documenter →  implementation-manual.md ← LIFECYCLE COMPLETE
```

---

## Instructions for the Assistant

When the user asks to "document the implementation" for a feature:

### 1. Source Code Audit
- Scan `bes-backend/extensions/<module>/<module>/` — read `models.py`, `schemas.py`, `services.py`, `router.py`, `events.py`, `manifest.py`.
- Scan `bes-frontend/libs/<module>/src/lib/<feature>/` — read all component files and registry entries.
- Compare what was built against `changes.md` — document any deviations and the reason.

### 2. Compliance Verification
Verify the implementation adheres to all Golden Rules:
- [ ] `BESBase` inherited on all tables (includes `subsidiary_id`, `metadata_`, `is_deleted`).
- [ ] Money Rule: `Numeric(20,4)` in all model columns, `Decimal` in all service arithmetic.
- [ ] Soft Delete: no `session.delete()` calls — only `is_deleted = True`.
- [ ] All list endpoints have `PaginationParams`.
- [ ] All write/sensitive endpoints have `require_permission()`.
- [ ] All responses use `success_response()` or `paginated_response()`.
- [ ] `subsidiary_id` filter present in all service-level queries.
- [ ] UI degrades correctly for `READONLY_EXTENSIONS`.
- [ ] `ProcessPipeline` and `Timeline` components used for all transactional drawers.
- [ ] All components registered lazily via `ComponentRegistry.registerLazy()`.

### 3. Generate Documentation

Write `features-plan/<module>/<feature>/implementation-manual.md`.

---

## implementation-manual.md Structure

## 1. Implementation Summary
- **Feature**: Name and module.
- **Lifecycle Status**: COMPLETE ✅
- **Deviations from spec**: Any changes made during implementation vs. `changes.md`.

## 2. Backend — As-Built Specification

### Database Schema
Table listing all SQLModel tables, columns, types, and relationships as actually implemented.

### API Reference
For each endpoint:
- **Route**: `METHOD /api/v1/<module>/<resource>`
- **Permission**: `require_permission("<key>")`
- **Request Body**: JSON schema.
- **Response**: JSON example using `StandardResponse` envelope.

### Business Logic Summary
Key service functions — input, business rules applied, output.

### Event Catalog
All events emitted and subscribed:
- **Emitted**: Event name, trigger action, payload schema.
- **Subscribed**: Event name, source module, handler action.

## 3. Frontend — As-Built Specification

### Component Tree
List of components, their file paths, and purpose.

### `@bes/shared-ui` Usage
Which shared-ui components are used and how.

### Process Transparency Integration
- `ProcessPipeline` status stages and trigger actions.
- `Timeline` events captured in the History tab.

### Shell Registration
- Route path.
- `ComponentRegistry` key.
- Sidebar navigation entry.

## 4. Security & Audit Compliance
- List of permission keys registered in `admin_permissions.json`.
- List of user actions written to the audit trail.
- `READONLY_EXTENSIONS` behavior (what is hidden/disabled).
- `subsidiary_id` isolation verification results.

## 5. Integration Points
- External modules this feature depends on.
- Events this feature emits that other modules consume.
- Events this feature consumes from other modules.

## 6. Maintenance & Troubleshooting
- **Common Issues**: Known failure scenarios and fixes.
- **Key Log Locations**: Where to find relevant logs for debugging.
- **Extension Config**: How to toggle this feature via `ACTIVE_EXTENSIONS`.

---

### Execution Rules
- **Output file**: `features-plan/<module>/<feature>/implementation-manual.md`
- **Lifecycle complete**: Once this file is written, the feature is fully documented and closed.
- Update `master-implementation-roadmap.md` to mark the step as `[x] DONE`.
