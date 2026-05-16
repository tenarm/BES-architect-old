---
name: backend-generate-feature-doc
description: This skill instructs the AI assistant to read an existing UI feature document and generate a detailed backend extension implementation plan.
---

# Skill: Backend Generate Feature Doc

This skill instructs the AI assistant to read a UI Feature Document (`features-plan/<module-name>/<feature-name>/frontend.md`) and produce a detailed, structured Backend Implementation Plan aligned with the BES Factory's Kernel-and-Plugin architecture.

---

## About the BES Backend Architecture

- **Stack**: FastAPI + SQLModel + PostgreSQL, managed via PDM workspace.
- **Core Kernel**: `bes-backend/core/` — provides Auth, DB pooling, Event Bus, RBAC, `BESBase`, responses, and pagination. Extensions MUST NOT modify Core.
- **Extensions**: `bes-backend/extensions/<module_name>/` — one PDM package per domain module.
- **Instances**: `bes-backend/instances/<client>/` — client-specific bootstrappers that activate extensions via `ACTIVE_EXTENSIONS` env var.
- **Startup**: On boot, the instance reads `ACTIVE_EXTENSIONS`, dynamically imports routers/models, and calls `SQLModel.metadata.create_all()` for self-healing table creation.
- **Licensing**: `READONLY_EXTENSIONS` loads a module in read-only mode (no write endpoints).

---

## Instructions for the Assistant

When the user asks to "backend generate feature doc" or "plan the backend extension," follow these steps:

1. Ask for the frontend doc path if not provided (e.g., `features-plan/finance/coa/frontend.md`).
2. Read the frontend doc carefully (APIs, data structures, events, business rules).
3. Read `.agent/rules/backend.md`, `.agent/rules/security.md`, and `.agent/rules/architecture.md`.

**IMPORTANT: File Location**
Write the backend plan into the same feature folder:
Format: `features-plan/<module-name>/<feature-name>/backend.md`
Create the `<feature-name>` folder if it does not already exist.

---

## Required Backend Plan Sections

## 1. Module Overview
- **Extension Name:** `snake_case` directory name (e.g., `finance`).
- **Backend Path:** `bes-backend/extensions/<extension_name>/`
- **Goal:** What this extension delivers based on the frontend doc.
- **Dependency Declaration:** `pyproject.toml` must list `core` as a dependency.

## 2. File Structure
List the exact files to create following the mandatory BES extension layout:
```
extensions/<module_name>/
├── <module_name>/
│   ├── __init__.py
│   ├── manifest.py      # ExtensionManifest — REQUIRED
│   ├── models.py        # SQLModel BESBase table definitions ONLY
│   ├── schemas.py       # Pydantic *Create/*Update/*Read schemas
│   ├── services.py      # All business logic (no HTTP logic here)
│   ├── router.py        # Thin FastAPI handlers (calls services)
│   └── events.py        # Event Bus emitters and subscribers
└── pyproject.toml
```

## 3. Data Models (`models.py`)
Define each SQLModel table class:
- **Always** inherit from `core.models.BESBase` (gets `id` UUID PK, `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `metadata_` JSONB).
- **Money Rule**: `Decimal` type with `sa_column=Column(Numeric(precision=20, scale=4))` for ALL financial fields.
- **Table naming**: `<module_name>_<entity_plural>` (e.g., `finance_accounts`).
- **Master Data FKs**: FK to `core` master tables (e.g., `customers.id`). Use `metadata_` Ghost FKs for cross-extension soft links.

## 4. Pydantic Schemas (`schemas.py`)
Define clean input/output schemas:
- `*Create`: Input schema. NEVER include `id`, `created_at`, `is_deleted`, `subsidiary_id`.
- `*Update`: Partial update schema. All fields Optional.
- `*Read`: Output schema. Includes computed fields from joins if needed.

## 5. Business Logic (`services.py`)
Describe all service functions:
- **Input**: `session: AsyncSession` + schema/data objects.
- **Validation**: Business rule checks (raise `HTTPException` for violations).
- **Transactions**: Multi-step operations wrapped in a single `AsyncSession` commit.
- **Money Rule**: All arithmetic uses `Decimal`, never `float`.
- **Soft Delete**: Use `is_deleted = True` — physical deletion is FORBIDDEN.
- **Permission Elevation**: Use `elevate_context()` for system-triggered writes to restricted resources (e.g., Sales posting to Finance GL).

## 6. API Routes (`router.py`)
List every endpoint:
- **Router prefix**: `/api/v1/<module_name>` (e.g., `/api/v1/finance`).
- **Thin layer**: Router calls service, returns response. No business logic.
- **Pagination**: ALL list endpoints use `core.pagination.PaginationParams` dependency.
- **RBAC**: ALL write/sensitive endpoints use `require_permission("<module>:<resource>:<action>")`.
- **Response Envelope**: Use `success_response(data=item)` or `paginated_response(data=items, total=count, ...)`.
- **Soft-delete filter**: All queries include `.where(Model.is_deleted == False)`.
- **READONLY mode**: Document which endpoints are disabled in `READONLY_EXTENSIONS` mode.

## 7. Events (`events.py`)
- **Emits**: Functions that emit events to the async Event Bus. Format: `UPPER_SNAKE_CASE` event types (e.g., `FINANCE_ACCOUNT_CREATED`). Include the full payload structure.
- **Subscribes**: Event handlers registered via `event_bus.subscribe(...)` in `register_event_handlers()`. Document events consumed from other modules.
- **Safety**: If a subscriber module is not loaded, events expire silently — no error.

## 8. Manifest (`manifest.py`)
Define the `ExtensionManifest` subclass:
- Must implement: `module_name`, `get_router()`, `get_models()`, `get_event_handlers()`.
- Must export a `manifest` instance at module level.

## 9. RBAC & Permissions
- **Permission strings**: `<module>:<resource>:<action>` (e.g., `finance:coa:read`, `finance:coa:write`).
- **Master JSON**: New permissions must be added to `core/core/admin_permissions.json`.
- **Context-Aware**: Distinguish manual (user) vs. auto_trigger (system via `elevate_context()`) actions.
- **Bootstrap**: Permissions flow to the UI via `GET /api/v1/bootstrap`.

## 10. Audit & Compliance
- `BESBase` auto-tracks `created_at`, `updated_at`, `created_by`.
- State any additional audit events to emit per business action (e.g., approval status changes).

## 11. Implementation Roadmap
Ordered checklist:
1. Create extension directory & `pyproject.toml`.
2. Define `BESBase` models (`models.py`).
3. Define Pydantic schemas (`schemas.py`).
4. Implement service layer (`services.py`).
5. Wire HTTP endpoints with pagination & RBAC (`router.py`).
6. Implement event emitters & handlers (`events.py`).
7. Create `ExtensionManifest` (`manifest.py`).
8. Register permissions in `admin_permissions.json`.
9. Add module to client's `onboarded/<client>.json` `licensed_modules`.
10. Verify: `GET /health` and `GET /api/v1/<module>/entities`.

---

### Execution Rules
- Save to: `features-plan/<module-name>/<feature-name>/backend.md`
- Use professional formatting and standard markdown syntax.
