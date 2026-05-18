# 1. Backend Architecture & Security Rules — BES

These rules govern all Python/FastAPI code and architectural patterns in the `bes-backend/` directory.

## 1. Core-as-a-Package & Repository Layout
- **The Engine (`bes-core`)**: The `bes-backend/core/` package is the kernel (auth, DB, RBAC, events, base models). NEVER add domain-specific business logic here.
- **Extensions**: Business modules live in `bes-backend/extensions/<module_name>/`. They depend on `core`. Core MUST NOT import from extensions.
- **Dependency**: Extensions declare core in `pyproject.toml` (`dependencies = ["core"]`).
- **Naming**: Use `snake_case` for extensions, `<module>_<entity>s` for DB tables, and `UPPER_SNAKE_CASE` for events.

## 2. Extension Module Strict Layering
Every extension MUST follow this structure:
- **`models.py`**: ONLY database table classes inheriting from `BESBase`.
- **`schemas.py`**: Pydantic `*Create` and `*Read` classes for API validation. NEVER use ORM models as API input.
- **`services.py`**: All business logic, transaction safety, and cross-table operations.
- **`router.py`**: Thin HTTP layer. Calls services. NO business logic.
- **`events.py`**: Event bus subscribers and emitters.
- **`manifest.py`**: MUST export a `manifest` instance of `ExtensionManifest`.

## 3. Database, Models, and Isolation (Multi-Tenancy)
- **Base Inheritance**: ALL tables MUST inherit from `core.models.BESBase` (provides UUID `id`, `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `metadata_`).
- **Multi-Tenancy**: DB-per-tenant isolation. Within a DB, `subsidiary_id` scopes data. Queries automatically filter via `ContextAwareSecurityMiddleware` and `BaseRepository._apply_scopes()`.
- **Hub-and-Spoke MDM**: Core Master data (`customers`, `vendors`) resides in `core`. Module-specific tables use FKs to core.
- **Soft Deletes**: Physical deletion is FORBIDDEN. Use `is_deleted = True`. Queries must filter by `is_deleted == False`.
- **The Money Rule**: ALL financial amounts MUST use `sa_column=Column(Numeric(precision=20, scale=4))` in DB and `Decimal` in Python. NEVER use `float`.

## 4. API Standards & Pagination
- **Envelope**: ALL responses MUST use `StandardResponse` (`success_response`, `paginated_response`, `error_response`) returning `{ status, data, metadata, error }`.
- **Pagination**: ALL list endpoints MUST support pagination using `core.pagination.PaginationParams`. Max page size is 200.
- **Route Prefix**: `/api/v1/<module>`.

## 5. Security & RBAC
- **Secrets**: NEVER hardcode secrets. Use `os.environ["KEY"]` (fail-fast).
- **Authentication**: JWT Access (short-lived) and Refresh (long-lived) tokens. Passwords hashed via bcrypt.
- **RBAC**: Endpoints modifying sensitive data MUST use `require_permission()`. Format: `<module>:<resource>:<action>` (e.g., `finance:coa:write`).
- **System Context**: Use `elevate_context()` to bypass checks for event-driven automated operations.
- **Audit**: `X-Request-ID` logging, and `updated_at` auto-updates on modifications.

## 6. Inter-Module Communication
- Modules communicate ONLY through the asynchronous **Event Bus** (`core.events`).
- Modules MUST NOT directly import or call another extension module's code.

## 7. Import Ordering
1. Standard library (`os`, `uuid`, `datetime`, `decimal`)
2. Third-party (`fastapi`, `sqlmodel`)
3. Core package (`core.database`, `core.responses`)
4. Current module (`.models`, `.schemas`, `.services`)
