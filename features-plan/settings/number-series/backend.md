## 1. Module Overview
- **Extension Name**: `settings`
- **Backend Path**: `bes-backend/extensions/settings/`
- **Goal**: Provides APIs for defining and managing sequence patterns (Number Series) for document identifiers across the BES.
- **pyproject.toml dependency**: Must list `core` as dependency.

## 2. File Structure
```
extensions/settings/
├── settings/
│   ├── __init__.py
│   ├── manifest.py    # ExtensionManifest
│   ├── models.py      # No new models (uses core.number_series)
│   ├── schemas.py     # NumberSeriesUpdate, NumberSeriesRead, PreviewRequest
│   ├── services.py    # Sequence generation and formatting logic
│   ├── router.py      # /api/v1/settings/number-series
│   └── events.py      # Event Bus emitters & subscribers
└── pyproject.toml
```

## 3. Data Models (`models.py`)
- **Note**: `core.number_series` is the central table for storing sequence states.
- Inherits `BESBase`.
- Fields: `code` (String, Unique), `prefix` (String), `suffix` (String), `padding` (Integer), `current_value` (BigInt), `reset_frequency` (Enum), `last_reset_date` (DateTime).

## 4. Pydantic Schemas (`schemas.py`)
- `NumberSeriesUpdate`: `prefix`, `suffix`, `padding`, `reset_frequency`. Cannot update `code`.
- `NumberSeriesRead`: Includes `id`, `code`, `current_value`, and formatted preview.
- `NumberSeriesPreview`: Request schema for testing patterns before saving.
- `NextNumberResponse`: Payload containing the formatted `next_number` string.

## 5. Business Logic (`services.py`)
- **Atomicity**: The `get_next_number(code)` service method must execute an atomic SQL `UPDATE ... RETURNING current_value` to ensure no duplicates in high-concurrency environments.
- **Formatting**: Engine logic to parse tokens like `{YYYY}` or `{MM}` based on current date.
- **Reset Logic**: Checks `last_reset_date` vs current date. If reset boundary passed (e.g., new year for `YEARLY`), resets `current_value` to 1 and updates `last_reset_date`.
- **Validation**: Prevent modifying `current_value` directly via standard API.

## 6. API Routes (`router.py`)
- Router prefix: `/api/v1/settings/number-series`.
- `GET /`: List all available series. `PaginationParams` dependency.
- `PATCH /{id}`: `require_permission("settings:number_series:write")`. Update pattern logic.
- `POST /preview`: Test a pattern with dummy tokens.
- `POST /next/{code}`: Internal-facing API for other modules to fetch numbers. Should use `require_permission` or internal authentication context.
- ALL responses use `success_response(data=...)` or `paginated_response(data=..., total=...)`.

## 7. Events (`events.py`)
- Emits:
  - `SETTINGS_SERIES_UPDATED` (payload: `NumberSeriesRead`)
- Subscribes: None required.

## 8. Manifest (`manifest.py`)
- Subclass `ExtensionManifest`.
- Export the `router` containing number series endpoints.

## 9. RBAC & Permissions
- Permission strings: `settings:number_series:write`.
- Add to `core/core/admin_permissions.json`.
- Actions are `manual` for pattern edits. `auto_trigger` via `elevate_context()` for internal `get_next_number` calls by other modules (e.g., Finance creating an invoice).

## 10. Implementation Roadmap
Ordered checklist:
1. Define `number_series` model in `core.models`.
2. Define Pydantic schemas in `settings/schemas.py`.
3. Implement `NumberSeriesService` featuring atomic increment logic in `services.py`.
4. Wire `/number-series` endpoints with RBAC (`router.py`).
5. Implement event emitters (`events.py`).
6. Register router in `manifest.py`.
7. Add permissions to `admin_permissions.json`.
