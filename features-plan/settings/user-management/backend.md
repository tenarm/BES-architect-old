## 1. Module Overview
- **Extension Name**: `settings` (or `core` auth)
- **Backend Path**: `bes-backend/extensions/settings/` (handling the management APIs for core auth tables)
- **Goal**: Provides administrative APIs to manage users and roles (RBAC). Roles define granular permissions for system access.
- **pyproject.toml dependency**: Must list `core` as dependency.

## 2. File Structure
```
extensions/settings/
├── settings/
│   ├── __init__.py
│   ├── manifest.py    # ExtensionManifest
│   ├── models.py      # No new models (uses core.users, core.roles)
│   ├── schemas.py     # UserCreate, UserUpdate, RoleCreate, RoleUpdate
│   ├── services.py    # User and Role business logic
│   ├── router.py      # /api/v1/settings/auth/users, /roles
│   └── events.py      # Event Bus emitters & subscribers
└── pyproject.toml
```

## 3. Data Models (`models.py`)
- **Note**: `users` and `roles` are core foundational tables located in `core.models`.
- `core.users`: Inherits `BESBase`.
- `core.roles`: Inherits `BESBase`. Includes a `permissions` column (JSONB) to store dynamic module access rights.

## 4. Pydantic Schemas (`schemas.py`)
- `UserCreate`: NEVER include `id`, `created_at`, `is_deleted`. Fields: `full_name`, `email`, `role_id` (UUID).
- `UserUpdate`: Optional fields: `full_name`, `email`, `role_id`, `is_active`.
- `UserRead`: Output schema including `id`, `is_active`, `last_login`, `created_at`.
- `RoleCreate`: `name`, `description`, `permissions` (Dict[str, Dict[str, Dict[str, bool]]]).
- `RoleUpdate`: `name`, `description`, `permissions`.
- `RoleRead`: Output schema with complete permissions matrix.

## 5. Business Logic (`services.py`)
- Validation:
  - User emails must be unique system-wide.
  - Cannot deactivate or delete superusers or self.
  - Permission constraints: If `write` is granted, `read` MUST be automatically forced to true.
- Soft Delete: `is_deleted = True` or `is_active = False` for users.
- Transactions: Role permission updates must completely replace the JSONB document atomically.

## 6. API Routes (`router.py`)
- Router prefix: `/api/v1/settings/users` and `/api/v1/settings/roles`.
- `GET /users`: `PaginationParams` dependency. Filter `is_deleted == False`.
- `POST /users`: `require_permission("settings:user_management:write")`.
- `PATCH /users/{id}`: `require_permission("settings:user_management:write")`.
- `GET /roles`: `PaginationParams` dependency.
- `POST /roles`: `require_permission("settings:user_management:write")`.
- `PATCH /roles/{id}`: Update permissions matrix.
- ALL responses use `success_response(data=...)` or `paginated_response(data=..., total=...)`.
- **READONLY_EXTENSIONS**: All write endpoints return 403 Forbidden.

## 7. Events (`events.py`)
- Emits:
  - `CORE_USER_CREATED` (payload: UserRead)
  - `CORE_USER_UPDATED`
  - `CORE_ROLE_CREATED`
  - `CORE_ROLE_UPDATED` (payload: RoleRead)
- Subscribes:
  - `AUTH_LOGIN_SUCCESS`: Updates user's `last_login` timestamp.

## 8. Manifest (`manifest.py`)
- Subclass `ExtensionManifest`.
- Add routers to `get_router()`.
- Register event handler for `AUTH_LOGIN_SUCCESS` in `get_event_handlers()`.

## 9. RBAC & Permissions
- Permission strings: `settings:user_management:read`, `settings:user_management:write`.
- Add to `core/core/admin_permissions.json`.
- Actions are `manual`. System actions like login updates execute via `elevate_context()`.

## 10. Implementation Roadmap
Ordered checklist:
1. Ensure `core` defines `users` and `roles` with JSONB permissions.
2. Define Pydantic schemas for Users and Roles in `settings`.
3. Implement `UserService` and `RoleService` (`services.py`).
4. Wire `/users` and `/roles` endpoints (`router.py`) with RBAC.
5. Implement event emitters and `AUTH_LOGIN_SUCCESS` subscriber.
6. Register routers and event handlers in `manifest.py`.
7. Add permissions to `admin_permissions.json`.
