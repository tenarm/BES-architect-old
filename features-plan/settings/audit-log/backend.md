## 1. Module Overview
- **Extension Name**: `settings` (Audit viewing functionality)
- **Backend Path**: `bes-backend/extensions/settings/`
- **Goal**: Provides read-only APIs for administrators to search, view, and analyze immutable system audit logs and field-level changes.
- **pyproject.toml dependency**: Must list `core` as dependency.

## 2. File Structure
```
extensions/settings/
├── settings/
│   ├── __init__.py
│   ├── manifest.py    # ExtensionManifest
│   ├── models.py      # No new models (uses core.audit_log)
│   ├── schemas.py     # AuditLogRead, FieldChangeRead
│   ├── services.py    # Audit query services
│   ├── router.py      # /api/v1/settings/audit
│   └── events.py      # Event Bus emitters & subscribers
└── pyproject.toml
```

## 3. Data Models (`models.py`)
- **Note**: The core audit tracking uses `core.audit_log` and `core.field_change_log`.
- `core.audit_log`: Append-only table. Fields: `id`, `timestamp`, `actor_id`, `actor_name`, `module`, `entity_type`, `entity_id`, `action`, `correlation_id`, `description`.
- `core.field_change_log`: Append-only. Fields: `audit_id`, `field_name`, `old_value` (JSON), `new_value` (JSON).

## 4. Pydantic Schemas (`schemas.py`)
- `FieldChangeRead`: Output schema for field mutations.
- `AuditLogRead`: Output schema. Contains standard audit details plus an optional list of `FieldChangeRead`.
- **Note**: There are NO `Create` or `Update` schemas in the settings extension because audit records are created internally by the `core.AuditService` during transactions.

## 5. Business Logic (`services.py`)
- Read-only querying. The service provides complex filtering by `module`, `user_id`, `entity_id`, and `date_range`.
- Masking: Service must ensure that any accidentally logged sensitive data (e.g., passwords) is masked before returning to the API.
- Causal Chains: Recursively query by `correlation_id` to build event chains.

## 6. API Routes (`router.py`)
- Router prefix: `/api/v1/settings/audit`.
- `GET /`: Global search. `PaginationParams` dependency. `require_permission("settings:audit_log:read")`.
- `GET /{entity_type}/{entity_id}`: Fetch timeline for a specific entity (used by UI Drawer Timelines). Does not require global audit permission, but may require read permission on the specific entity type.
- `GET /chain/{correlation_id}`: Fetch related events.
- `GET /stream`: SSE endpoint. Yields events from Redis/EventBus for dashboard activity feeds.
- ALL responses use `success_response(data=...)` or `paginated_response(data=..., total=...)`.

## 7. Events (`events.py`)
- Emits: None directly from this router (core emits `AUDIT_RECORDED`).
- Subscribes: None required for the REST API. The `/stream` SSE endpoint will dynamically subscribe to `AUDIT_RECORDED`.

## 8. Manifest (`manifest.py`)
- Subclass `ExtensionManifest`.
- Export the `router` containing audit endpoints.

## 9. RBAC & Permissions
- Permission strings: `settings:audit_log:read`.
- Add to `core/core/admin_permissions.json`.
- Actions are `manual`.

## 10. Implementation Roadmap
Ordered checklist:
1. Ensure `core` provides `audit_log` models and internal `AuditService`.
2. Define `AuditLogRead` Pydantic schemas (`schemas.py`).
3. Implement `AuditQueryService` featuring filtering and correlation chains (`services.py`).
4. Wire `/audit` endpoints with RBAC (`router.py`).
5. Implement SSE endpoint for `/stream`.
6. Register router in `manifest.py`.
7. Add permissions to `admin_permissions.json`.
