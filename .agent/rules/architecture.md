# Architecture Rules — BES Factory

These rules define the core architectural constraints for the BES Factory project. Every code change MUST comply with these rules.

---

## 1. Core-as-a-Package Pattern

- The `bes-backend/core/` package is the **kernel**. It contains auth, database, RBAC, event bus, base models, and shared utilities.
- **NEVER** add domain-specific business logic (finance, sales, HR, etc.) to the core package.
- Extensions depend on Core via `pyproject.toml` dependency declaration: `dependencies = ["core"]`.
- Core MUST NOT import from any extension. Dependency flows one way: `Extension → Core`.

## 2. Extension Module Structure

Every backend extension module MUST follow this file structure:

```
extensions/<module_name>/
├── <module_name>/
│   ├── __init__.py
│   ├── manifest.py      # ExtensionManifest implementation (REQUIRED)
│   ├── models.py         # SQLModel table definitions ONLY
│   ├── schemas.py         # Pydantic request/response schemas
│   ├── services.py        # Business logic layer
│   ├── router.py          # FastAPI HTTP handlers (thin, calls services)
│   └── events.py          # Event subscribers and emitters
└── pyproject.toml
```

- **models.py**: ONLY database table classes inheriting from `BESBase`. No API schemas here.
- **schemas.py**: Pydantic `*Create` and `*Read` classes for API input/output validation.
- **services.py**: All business logic, validation, and cross-table operations.
- **router.py**: Thin HTTP layer — receives request, calls service, returns response.
- **events.py**: Event bus subscribers (incoming events) and emitter functions (outgoing events).
- **manifest.py**: MUST export a `manifest` instance of `ExtensionManifest`.

## 3. Multi-Tenancy

- Every client gets a **dedicated database** (DB-per-tenant isolation).
- Client configuration is stored in `onboarded/<client_id>.json`.
- The `subsidiary_id` column on `BESBase` provides row-level organizational scoping WITHIN a client's database.
- Use `subsidiary_id_context` ContextVar for automatic query filtering.

## 4. Repository Layout

```
BES/
├── bes-backend/           # Python monorepo (PDM workspace)
│   ├── core/              # Core kernel package
│   ├── extensions/        # Business domain modules (one per BES module)
│   ├── instances/         # Client-specific bootstrappers
│   ├── onboarded/         # Client config JSONs
│   └── data/              # Seed data files
├── bes-frontend/                # Frontend monorepo (Nx workspace)
│   ├── apps/shell/        # Main shell application
│   └── libs/              # Shared UI + module-specific libraries
└── bvk-items/             # Documentation and commands
```

## 5. Naming Conventions

| Item | Convention | Example |
|:-----|:-----------|:--------|
| Extension directory | `snake_case` | `supply_chain`, `quality_management` |
| DB table name | `<module>_<entity>s` | `finance_accounts`, `sales_quotations` |
| API route prefix | `/api/v1/<module>` | `/api/v1/finance`, `/api/v1/sales` |
| Frontend lib | `@bes/<module>` | `@bes/finance`, `@bes/shared-ui` |
| Event type | `UPPER_SNAKE_CASE` | `ORDER_CONFIRMED`, `INVOICE_POSTED` |

## 6. Inter-Module Communication

- Modules communicate ONLY through the **Event Bus** (`core.events`).
- A module MUST NOT directly import or call another extension module's code.
- Events are fire-and-forget: if a subscriber module is not loaded, the event expires silently.
- Use `elevate_context()` when an event handler needs to write to restricted resources.
