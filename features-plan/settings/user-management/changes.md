# Implementation Spec: User Management & RBAC

## 1. Impacted Backend Files
| File Path | Change Type | Implementation Notes |
| :--- | :--- | :--- |
| `bes-backend/core/models.py` | `[MODIFY]` | Ensure `users` and `roles` inherit `BESBase`. Verify `roles.permissions` is `JSONB`. |
| `bes-backend/extensions/settings/schemas.py` | `[MODIFY]` | Add `UserCreate`, `UserUpdate`, `RoleCreate`, `RoleUpdate`, `RoleRead` schemas. |
| `bes-backend/extensions/settings/services.py` | `[MODIFY]` | Add `UserService` and `RoleService`. Enforce email uniqueness, protect superusers, handle role atomic permission updates. |
| `bes-backend/extensions/settings/router.py` | `[MODIFY]` | Add `/users` and `/roles` routes. `require_permission("settings:user_management:write")`. Handle `READONLY_EXTENSIONS`. |
| `bes-backend/extensions/settings/events.py` | `[MODIFY]` | Emit `CORE_USER_CREATED/UPDATED`, `CORE_ROLE_CREATED/UPDATED`. Subscribe to `AUTH_LOGIN_SUCCESS`. |
| `bes-backend/extensions/settings/manifest.py` | `[MODIFY]` | Register routers and `AUTH_LOGIN_SUCCESS` subscriber. |
| `bes-backend/core/core/admin_permissions.json` | `[MODIFY]` | Add `settings:user_management:write`, `settings:user_management:read`. |

## 2. Impacted Frontend Files
| Nx Path | Change Type | `@bes/shared-ui` Components | Registration / Notes |
| :--- | :--- | :--- | :--- |
| `bes-frontend/libs/settings/src/lib/user-management/` | `[NEW]` | `Table`, `Drawer`, `Form` | New feature directory for Users & Roles. |
| `bes-frontend/libs/settings/src/lib/user-management/UserTable.tsx` | `[NEW]` | `Table`, `Skeleton` | List users. Handles `is_active` toggle. |
| `bes-frontend/libs/settings/src/lib/user-management/RoleDrawer.tsx` | `[NEW]` | `Drawer`, `Timeline` | Includes complex Permissions Matrix grid/tree. |
| `bes-frontend/libs/settings/src/index.ts` | `[MODIFY]` | | Export routes. |

## 3. Integration Checklist
- [ ] **Database**: Verify `roles.permissions` column is robustly handling JSONB.
- [ ] **Permissions**: Add `"settings:user_management:read"` and `write` to `admin_permissions.json`.
- [ ] **Bootstrap**: Verify Shell correctly restricts sidebar paths based on fetched user permissions.
- [ ] **Events**: Confirm `last_login` is updated on `AUTH_LOGIN_SUCCESS`.
- [ ] **Licensing**: Verify `READONLY_EXTENSIONS` disables permission editing UI.

## 4. Implementation Order
`Core Models Verification → Schemas → Services → APIs (RBAC) → Events → Manifest → Frontend Library → UI Components (Users first, then Roles Matrix) → Shell Auth Verification → QA`
