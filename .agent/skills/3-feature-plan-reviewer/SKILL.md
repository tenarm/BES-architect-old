---
name: feature-plan-reviewer
description: This skill instructs the AI assistant to perform a comprehensive architectural and security review of a feature's frontend and backend plans.
---

# Skill: Feature Plan Reviewer

This skill instructs the AI assistant to act as a BES Solutions Architect and perform a thorough review of a single feature's plans (`frontend.md` + `backend.md`) in `features-plan/<module>/<feature>/`. It checks completeness, architectural adherence, security compliance, and cross-doc compatibility — then generates a `changes.md` specification.

---

## About the BES Architecture (Reviewer Context)

**Golden Rules to enforce:**
1. **BESBase**: Every table inherits it. Gets `id` (UUID), `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `metadata_` (JSONB).
2. **Money Rule**: `Numeric(20,4)` in DB, `Decimal` in Python, `decimal.js`/`big.js` in React. NEVER `float`.
3. **Soft Deletes**: `is_deleted = True`. Physical deletion is FORBIDDEN.
4. **Layering**: `models.py` → DB only. `schemas.py` → API I/O only. `services.py` → business logic. `router.py` → thin HTTP. `events.py` → bus.
5. **Response Envelope**: `{ "status", "data", "metadata", "error" }` always.
6. **Pagination**: All list endpoints use `PaginationParams`.
7. **RBAC**: All write/sensitive endpoints use `require_permission("<module>:<resource>:<action>")`.
8. **Hub-and-Spoke MDM**: Master data in `core` schema. Module tables reference via FK or `metadata_` Ghost FK.
9. **Event Bus**: Async emit/subscribe. Events are UPPER_SNAKE_CASE. Safe if subscriber not loaded.
10. **Licensing**: UI degrades gracefully for `READONLY_EXTENSIONS` modules.

---

## Instructions for the Assistant

When the user asks to "review the feature plan" for a specific feature:

1. Navigate to `features-plan/<module>/<feature>/`.
2. Read `frontend.md` and `backend.md`.
3. Enforce all Golden Rules below.
4. Generate the review report as a response, then write `changes.md`.

---

## Review Criteria

### A. Plan Completeness

**Frontend doc must contain all 14 sections:**
- Module, Name, Description, Depends On, UI Details, Sample Data, Required APIs, Database Tables, Events, Business Rules, Security/RBAC, Process Transparency, Implementation Roadmap, Verification & QA.

**Backend doc must contain all 10 sections:**
- Module Overview, File Structure, Data Models, Pydantic Schemas, Business Logic, API Routes, Events, Manifest, RBAC & Permissions, Audit & Compliance.

Flag any sections that are missing, vague, or incomplete.

### B. Architectural Adherence
- All tables inherit `BESBase`?
- `subsidiary_id` present on all tables?
- Table names follow `<module>_<entity_plural>` convention?
- Core master data (customers, vendors, products, uoms) referenced correctly via Hub-and-Spoke?
- Layering rules obeyed (no business logic in router)?

### C. Backend & Data Integrity
- Money Rule enforced (`Numeric(20,4)` in DB, `Decimal` in code)?
- All list endpoints have pagination?
- All responses use `StandardResponse` envelope?
- Soft-delete filter on all queries?
- `*Create` schemas used for all inputs (no ORM model as input)?

### D. Security & RBAC
- `require_permission()` specified for all write/sensitive endpoints?
- Permission string format correct: `<module>:<resource>:<action>`?
- Permissions registered in `admin_permissions.json`?
- `READONLY_EXTENSIONS` behavior documented in frontend?
- Context-aware RBAC noted where `elevate_context()` is needed?

### E. Event & Licensing Compatibility
- Event names unique and in UPPER_SNAKE_CASE?
- Emitted events in backend match events listed in frontend for SSE updates?
- Bootstrap endpoint usage documented where needed?

### F. Frontend ↔ Backend Compatibility
- API routes in `frontend.md` (Section 7) match routes planned in `backend.md` (Section 6)?
- Request/response payload shapes align between frontend sample data (Section 6) and backend schemas (Section 4)?
- Events emitted in backend (Section 7) match events frontend listens to (Section 9)?

---

## Output: Review Report (as response)

## 1. Executive Summary
Overall verdict: `APPROVED` / `NEEDS REVISION` / `CRITICAL GAPS`.

## 2. Completeness Scorecard
A table rating each section of frontend and backend docs (Complete / Partial / Missing).

## 3. Findings

### ✅ Architectural Strengths

### ❌ Critical Violations (blocks implementation)

### ⚠️ Compatibility Gaps (frontend ↔ backend misalignments)

### 💡 Recommendations

## 4. Final Verdict
- **Status**: [APPROVED / REJECT / PENDING]
- **Blocking Issues**: List any issues that must be fixed before implementation.

---

## Change Document Generation

After the review, ALWAYS write a `changes.md` file at `features-plan/<module>/<feature>/changes.md`.

This is the implementation specification for developers:

### 1. Impacted Backend Files
Table listing each file (`models.py`, `schemas.py`, etc.), change type (`[NEW]`/`[MODIFY]`), and implementation notes.

### 2. Impacted Frontend Files
Table listing Nx library path, page components, shell registration files, and any new `@bes/shared-ui` components needed.

### 3. Integration Checklist
- **Database**: Migration command (e.g., `SQLModel.metadata.create_all()` or Alembic).
- **Permissions**: Exact keys to add to `admin_permissions.json`.
- **Bootstrap**: Confirm permissions surfaced via `GET /api/v1/bootstrap`.
- **Events**: Payload examples for Pub/Sub smoke-testing.
- **Licensing**: Readonly mode verification steps.

### 4. Implementation Order
Prioritized task list following dependency order:
`Core Master Data → Backend Models → Schemas → Services → APIs (with RBAC) → Event Handlers → Manifest → Frontend Library → Shell Registration → UI Components → SSE Integration → QA`.
