## 1. Module Overview
- **Extension Name**: `settings`
- **Backend Path**: `bes-backend/extensions/settings/`
- **Goal**: Provides administrative APIs to manage organizational structures (Subsidiaries), serving as the foundational tenant configuration for the BES multi-tenant architecture.
- **pyproject.toml dependency**: Must list `core` as dependency.

## 2. File Structure
```
extensions/settings/
├── settings/
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
- **Note**: As per the frontend spec, the `subsidiaries` table is a foundational master table that resides in the `core` schema (`core.models.subsidiaries`). The `settings` module does not define this model but rather imports it to provide the management APIs.
- Any future module-specific settings for subsidiaries would inherit `core.models.BESBase` and use `subsidiary_id` as a foreign key to the `core` table.
- Table naming convention for settings-specific extensions: `settings_<entity_plural>`.

## 4. Pydantic Schemas (`schemas.py`)
- `SubsidiaryCreate`: Input schema. NEVER include `id`, `created_at`, `is_deleted`, `subsidiary_id`. Fields: `name`, `legal_name`, `registration_number`, `tax_id`, `currency_code`, `fiscal_year_start`, `address_json` (Optional), `metadata_` (Optional).
- `SubsidiaryUpdate`: Partial update (all fields Optional).
- `SubsidiaryRead`: Output schema with all computed/joined fields, including `id`, `subsidiary_id`, `created_at`, `is_deleted`.

## 5. Business Logic (`services.py`)
- Input: `session: AsyncSession` + `SubsidiaryCreate`/`SubsidiaryUpdate`.
- Validation: raise `HTTPException` for rule violations.
  - `registration_number` must be unique across non-deleted entities.
  - `currency_code` must be a valid ISO 4217 code.
  - Deletion check: Cannot delete if it has active users or is the only entity.
- Transactions: Multi-step ops in a single `AsyncSession` commit.
- Soft Delete: `is_deleted = True`. Physical deletion FORBIDDEN.
- Permission Elevation: N/A for standard CRUD operations here.

## 6. API Routes (`router.py`)
- Router prefix: `/api/v1/settings/subsidiaries`.
- Thin layer only — no business logic.
- `GET /`: Uses `PaginationParams` dependency. Filter `.where(Subsidiary.is_deleted == False)`.
- `GET /{id}`: Fetch single subsidiary.
- `POST /`: `require_permission("settings:subsidiary:write")`.
- `PATCH /{id}`: `require_permission("settings:subsidiary:write")`.
- `DELETE /{id}`: `require_permission("settings:subsidiary:write")`. Soft delete.
- ALL responses use `success_response(data=...)` or `paginated_response(data=..., total=...)`.
- **READONLY_EXTENSIONS**: In read-only mode, `POST`, `PATCH`, and `DELETE` endpoints return 403.

## 7. Events (`events.py`)
- Emits:
  - `SETTINGS_SUBSIDIARY_CREATED` (payload: `SubsidiaryRead`)
  - `SETTINGS_SUBSIDIARY_UPDATED` (payload: `SubsidiaryRead`)
  - `SETTINGS_SUBSIDIARY_DELETED` (payload: `id`)
- Subscribes: Handlers registered via `event_bus.subscribe(...)` in `register_event_handlers()`. None required for this sub-feature.

## 8. Manifest (`manifest.py`)
- Subclass `ExtensionManifest`.
- Implement:
  - `module_name = "settings"`
  - `get_router()` returns `router` from `router.py`.
  - `get_models()` returns any settings-specific models.
  - `get_event_handlers()` returns an empty list/dict for this feature.
- Export `manifest` instance at module level.

## 9. RBAC & Permissions
- Permission strings: `settings:subsidiary:read`, `settings:subsidiary:write`.
- Add to `core/core/admin_permissions.json`.
- Distinguish `manual` (admin actions) vs `auto_trigger`.

## 10. Implementation Roadmap
Ordered checklist:
1. Create extension directory & `pyproject.toml` (if not already existing for `settings`).
2. Define `BESBase` models (using `core.models.subsidiaries`).
3. Define Pydantic schemas (`schemas.py`).
4. Implement service layer (`services.py`).
5. Wire endpoints with pagination & RBAC (`router.py`).
6. Implement event emitters & handlers (`events.py`).
7. Register `ExtensionManifest` (`manifest.py`).
8. Add permissions to `admin_permissions.json`.
