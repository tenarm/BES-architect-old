---
name: feature-plan-reviewer
description: "LIFECYCLE STEP 3: Review frontend.md + backend.md for a feature, enforce the Golden Rules, and generate changes.md (the consolidated implementation spec)."
---

# Skill: Feature Plan Reviewer
**Lifecycle Position: STEP 3 of 6 — Audit**
**Reads from:** `frontend.md` + `backend.md` (outputs of Steps 1 & 2)
**Feeds into:** `master-implementation-architect` (Step 4) and `feature-implementation-orchestrator` (Step 5)

This skill is the quality gate. It enforces all 10 BES Golden Rules, checks cross-doc compatibility, validates completeness, and generates the consolidated `changes.md` — the definitive implementation spec for developers.

---

## Lifecycle Context
```
STEP 1: frontend-generate-feature-doc  →  frontend.md
STEP 2: backend-generate-feature-doc   →  backend.md
[YOU ARE HERE]
STEP 3: feature-plan-reviewer          →  changes.md  ← QUALITY GATE
STEP 4: master-implementation-architect →  master-implementation-roadmap.md
STEP 5: feature-implementation-orchestrator → CODE
STEP 6: post-implementation-documenter →  implementation-manual.md
```

---

## The 10 BES Golden Rules (Enforce ALL)
1. **BESBase**: Every table inherits it → `id` (UUID), `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `metadata_` (JSONB).
2. **Money Rule**: `Numeric(20,4)` in DB, `Decimal` in Python, `decimal.js`/`big.js` in React. NEVER `float`.
3. **Soft Deletes**: `is_deleted = True`. Physical deletion FORBIDDEN.
4. **Layering**: `models.py` → DB only. `schemas.py` → API I/O. `services.py` → business logic. `router.py` → thin HTTP. `events.py` → bus.
5. **Response Envelope**: `{ "status", "data", "metadata", "error" }` always.
6. **Pagination**: `PaginationParams` on all list endpoints.
7. **RBAC**: `require_permission("<module>:<resource>:<action>")` on all write/sensitive endpoints.
8. **Hub-and-Spoke MDM**: Master data in `core` schema. Module tables FK to core.
9. **Event Bus**: Async emit/subscribe. UPPER_SNAKE_CASE. No direct cross-module imports.
10. **Licensing**: UI degrades gracefully for `READONLY_EXTENSIONS`. Backend disables write endpoints.

---

## Instructions for the Assistant

When the user asks to "review the feature plan" for `features-plan/<module>/<feature>/`:

1. Read `frontend.md` and `backend.md`.
2. Run all review criteria below.
3. Output the review report as a response.
4. Write `changes.md` to `features-plan/<module>/<feature>/changes.md`.

---

## Review Criteria

### A. Plan Completeness
**Frontend doc must have all 14 sections**: Module, Name, Description, Depends On, UI Details, Sample Data, Required APIs, Database Tables, Events, Business Rules, Security/RBAC, Process Transparency, Implementation Roadmap, Verification & QA.

**Backend doc must have all 10 sections**: Module Overview, File Structure, Data Models, Pydantic Schemas, Business Logic, API Routes, Events, Manifest, RBAC & Permissions, Implementation Roadmap.

### B. Architectural Adherence
- All tables inherit `BESBase`?
- `subsidiary_id` on every table?
- Table names follow `<module>_<entity_plural>`?
- Hub-and-Spoke MDM correctly applied?
- Strict layer separation observed?

### C. Backend & Data Integrity
- Money Rule enforced (`Numeric(20,4)` + `Decimal`)?
- All list endpoints have pagination?
- All responses use `StandardResponse` envelope?
- Soft-delete filter on all queries?
- `*Create` schemas used for all inputs (not ORM models)?

### D. Security & RBAC
- `require_permission()` on all write/sensitive endpoints?
- Permission string format `<module>:<resource>:<action>` correct?
- Permissions registered in `admin_permissions.json`?
- `READONLY_EXTENSIONS` behavior documented in both docs?

### E. Frontend ↔ Backend Compatibility
- Routes in `frontend.md` Section 7 match routes in `backend.md` Section 6?
- Payload schemas in `frontend.md` Section 6 align with `backend.md` Section 4?
- Events emitted in `backend.md` match events subscribed in `frontend.md`?

---

## Output 1: Review Report (as response)

### Executive Summary: `APPROVED` / `NEEDS REVISION` / `CRITICAL GAPS`

### Completeness Scorecard
Table rating each section (Complete / Partial / Missing).

### Findings
- ✅ Architectural Strengths
- ❌ Critical Violations (blocks implementation)
- ⚠️ Compatibility Gaps
- 💡 Recommendations

### Final Verdict: `APPROVED` / `REJECT` / `PENDING`

---

## Output 2: changes.md (ALWAYS write this file)
**Path**: `features-plan/<module>/<feature>/changes.md`

This is the implementation spec for the `feature-implementation-orchestrator`.

### 1. Impacted Backend Files
Table: file path | change type (`[NEW]`/`[MODIFY]`) | implementation notes.

### 2. Impacted Frontend Files
Table: Nx path | change type | `@bes/shared-ui` components needed | shell registration files.

### 3. Integration Checklist
- Database: migration command.
- Permissions: exact keys for `admin_permissions.json`.
- Bootstrap: confirm permissions in `GET /api/v1/bootstrap`.
- Events: payload examples for smoke-testing.
- Licensing: readonly mode verification steps.

### 4. Implementation Order
`Core Master Data → Backend Models → Schemas → Services → APIs (RBAC) → Events → Manifest → Frontend Library → Shell Registration → UI Components → SSE Integration → QA`

---

### Execution Rules
- **Output files**: `changes.md` in `features-plan/<module>/<feature>/`
- **Module-level context**: Optionally read `module-cross-features-changes.md` to check for shared services.
- **Next step**: Run `4-module-architect-reviewer` (if whole-module audit needed) OR go straight to `master-implementation-architect` (Step 4).
