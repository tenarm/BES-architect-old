---
name: backend-generate-feature-doc
description: "LIFECYCLE STEP 2: Read frontend.md and generate the backend extension implementation plan (backend.md) for a BES feature."
---

# Skill: Backend Generate Feature Doc
**Lifecycle Position: STEP 2 of 6 — Design**
**Reads from:** `features-plan/<module>/<feature>/frontend.md` (output of Step 1)
**Feeds into:** `3-feature-plan-reviewer`

This skill reads the frontend feature specification and generates a matching backend implementation plan. Both documents together are the input to the Feature Plan Reviewer.

---

## Lifecycle Context
```
STEP 1: frontend-generate-feature-doc  →  frontend.md
[YOU ARE HERE]
STEP 2: backend-generate-feature-doc   →  backend.md
STEP 3: feature-plan-reviewer          →  changes.md
STEP 4: master-implementation-architect →  master-implementation-roadmap.md
STEP 5: feature-implementation-orchestrator → CODE
STEP 6: post-implementation-documenter →  implementation-manual.md
```

---

## About the BES Backend Architecture
- **Stack**: FastAPI + SQLModel + PostgreSQL, PDM workspace.
- **Core Kernel** (`bes-backend/core/`): Auth (JWT), DB Pooling, RBAC (`require_permission`), Event Bus, `BESBase`, `StandardResponse`, `PaginationParams`, File Storage, UOM Registry. Extensions MUST NOT modify Core.
- **Extensions** (`bes-backend/extensions/<module>/`): One PDM package per domain module.
- **Instances** (`bes-backend/instances/<client>/`): Client bootstrappers activating extensions via `ACTIVE_EXTENSIONS` env var.
- **Startup**: `SQLModel.metadata.create_all()` — self-healing table creation on boot.
- **Licensing**: `READONLY_EXTENSIONS` loads a module in read-only mode.

---

## Instructions for the Assistant

When the user asks to "backend generate feature doc":

1. Ask for `features-plan/<module>/<feature>/frontend.md` path if not provided.
2. Read `frontend.md`, `.agent/rules/backend.md`, `.agent/rules/security.md`, `.agent/rules/architecture.md`.
3. Generate the backend plan below.
4. Save to `features-plan/<module-name>/<feature-name>/backend.md`. Create folder if needed.

---

## Required Backend Plan Sections

## 1. Module Overview
- **Extension Name**: `snake_case` (e.g., `finance`).
- **Backend Path**: `bes-backend/extensions/<extension_name>/`.
- **Goal**: What this extension delivers.
- **pyproject.toml dependency**: Must list `core` as dependency.

## 2. File Structure
Exact files to create:
```
extensions/<module_name>/
├── <module_name>/
│   ├── __init__.py
│   ├── manifest.py    # ExtensionManifest — REQUIRED
│   ├── models.py      # SQLModel BESBase tables ONLY
│   ├── schemas.py     # Pydantic *Create/*Update/*Read
│   ├── services.py    # All business logic
│   ├── router.py      # Thin FastAPI handlers
│   └── events.py      # Event Bus emitters & subscribers
└── pyproject.toml
```

## 3. Data Models (`models.py`)
- Inherit `core.models.BESBase` — gets `id` (UUID PK), `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `metadata_` (JSONB).
- **Money Rule**: `sa_column=Column(Numeric(precision=20, scale=4))` for ALL financial fields.
- Table naming: `<module>_<entity_plural>` (e.g., `finance_accounts`).
- Master Data FKs to `core` tables. Use `metadata_` Ghost FKs for cross-extension soft links.

## 4. Pydantic Schemas (`schemas.py`)
- `*Create`: Input schema. NEVER include `id`, `created_at`, `is_deleted`, `subsidiary_id`.
- `*Update`: Partial update (all fields Optional).
- `*Read`: Output schema with all computed/joined fields.

## 5. Business Logic (`services.py`)
- Input: `session: AsyncSession` + schema/data objects.
- Validation: raise `HTTPException` for rule violations.
- Transactions: Multi-step ops in a single `AsyncSession` commit.
- Money Rule: All arithmetic uses `Decimal`, never `float`.
- Soft Delete: `is_deleted = True`. Physical deletion FORBIDDEN.
- Permission Elevation: Use `elevate_context()` for system-triggered cross-resource writes.

## 6. API Routes (`router.py`)
- Router prefix: `/api/v1/<module_name>`.
- Thin layer only — no business logic.
- ALL list endpoints use `PaginationParams` dependency.
- ALL write/sensitive endpoints use `require_permission("<module>:<resource>:<action>")`.
- ALL responses use `success_response(data=...)` or `paginated_response(data=..., total=...)`.
- Soft-delete filter: `.where(Model.is_deleted == False)` on all queries.
- Document which endpoints are disabled in `READONLY_EXTENSIONS` mode.

## 7. Events (`events.py`)
- Emits: `UPPER_SNAKE_CASE` event types with full payload schema.
- Subscribes: Handlers registered via `event_bus.subscribe(...)` in `register_event_handlers()`.
- Events from unloaded modules expire silently.

## 8. Manifest (`manifest.py`)
- Subclass `ExtensionManifest`.
- Implement: `module_name`, `get_router()`, `get_models()`, `get_event_handlers()`.
- Export a `manifest` instance at module level.

## 9. RBAC & Permissions
- Permission strings: `<module>:<resource>:<action>`.
- Add to `core/core/admin_permissions.json`.
- Distinguish `manual` vs. `auto_trigger` (system context via `elevate_context()`).
- Permissions surfaced via `GET /api/v1/bootstrap`.

## 10. Implementation Roadmap
Ordered checklist:
1. Create extension directory & `pyproject.toml`.
2. Define `BESBase` models.
3. Define Pydantic schemas.
4. Implement service layer.
5. Wire endpoints with pagination & RBAC.
6. Implement event emitters & handlers.
7. Register `ExtensionManifest`.
8. Add permissions to `admin_permissions.json`.

---

### Execution Rules
- **Output file**: `features-plan/<module-name>/<feature-name>/backend.md`
- **Next step**: Run `3-feature-plan-reviewer` with both `frontend.md` and `backend.md`.
