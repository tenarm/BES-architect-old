## 1. Module
`settings`

## 2. Name
User Management & RBAC

## 3. Description
The User Management & RBAC (Role-Based Access Control) feature provides administrators with the tools to manage the system's human capital and their access privileges. It covers user lifecycle management (invitation, activation, deactivation) and the definition of Roles. Roles are collections of granular permissions (`read`, `write`, `delete`) mapped to specific module resources. This feature ensures that users only see and interact with data they are authorized to access, maintaining system security and data integrity.

## 4. Depends On
- **Core Kernel:**
  - `User` & `Role` models: Foundation for authentication and authorization.
  - `RBAC Service`: To validate permissions at the API level.
  - `Bootstrap`: To load the current user's effective permissions into the Shell.
- **Cross-Module:**
  - `HR`: For linking users to employee profiles (optional, for Phase 2).
- **UI Libraries:**
  - `@bes/shared-ui`: `Table`, `Drawer`, `Form`, `Skeleton`, `Timeline`.

## 5. Feature UI — The UI as Information
**Feature Name:** User & Role Manager
**UI Details:**
- **User List Table:** Columns for `full_name`, `email`, `role`, `status` (Active/Inactive), and `last_login`.
- **User Detail Drawer:**
  - **Account Tab:** Basic info, email, and role assignment dropdown.
  - **Security Tab:** Password reset triggers, 2FA status (Day 2).
- **Role List Table:** Columns for `role_name`, `description`, and `user_count`.
- **Role Detail Drawer:**
  - **Permissions Matrix:** A hierarchical tree or grid where admins can toggle `Read`, `Write`, and `Delete` for every resource in every module (sourced from `admin_permissions.json`).
- **User Journey:**
  1. Admin goes to **Settings > Users**.
  2. Clicks **"Create User"**.
  3. Fills in name, email, and selects a **Role**.
  4. System creates the user, emits `CORE_USER_CREATED`, and sends an invite email.

## 6. Sample Data Structure (YAML + JSON)
```yaml
# Role Definition
id: "role-uuid-123"
name: "Finance Manager"
description: "Full access to finance, read-only elsewhere"
permissions:
  finance:
    coa: { read: true, write: true, delete: true }
    gl: { read: true, write: true, delete: false }
  sales:
    customer_master: { read: true, write: false, delete: false }
```

```json
{
  "status": "success",
  "data": {
    "id": "user-uuid-456",
    "full_name": "Jane Smith",
    "email": "jane@acme.com",
    "role_id": "role-uuid-123",
    "is_active": true
  }
}
```

## 7. Required APIs
- `GET /api/v1/auth/users`: List users.
- `GET /api/v1/auth/roles`: List roles.
- `POST /api/v1/auth/users`: Create/Invite user.
- `PATCH /api/v1/auth/users/{id}`: Update user/role assignment.
- `POST /api/v1/auth/roles`: Create new role.
- `PATCH /api/v1/auth/roles/{id}`: Update permissions matrix.

## 8. Database Tables & Architecture
- **Table:** `core.users` (Existing)
- **Table:** `core.roles` (Existing)
- **Permissions Storage:** Stored as a JSONB object in `roles.permissions` to allow for dynamic module expansion without schema changes.

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `CORE_USER_CREATED`, `CORE_USER_UPDATED`
  - `CORE_ROLE_CREATED`, `CORE_ROLE_UPDATED`
- **Listens To:**
  - `AUTH_LOGIN_SUCCESS`: To update `last_login` timestamp.

## 10. Business Rules & Validations
- **Superuser Immunity:** Superusers cannot be deactivated or deleted by other users.
- **Unique Email:** User emails must be unique system-wide.
- **Permission Inheritance:** If `write` is true, `read` MUST be automatically forced to true.
- **Soft Deletion:** Users are never physically deleted; `is_active` or `is_deleted` is used.

## 11. Security, Audit, and RBAC
- **Permission:** `settings:user_management:write`, `settings:user_management:read`.
- **Audit Trail:** Every permission change in a Role must be logged with a before/after diff of the JSONB object.
- **Context-Aware RBAC:** Ensure that a user cannot elevate their own role or permissions.

## 12. Process Transparency & Workflow Pipeline
- **Right Panel (Drawer) UX:**
  - **Macro View (`ProcessPipeline`):** Invited → Active → Deactivated.
  - **Micro View (`Timeline`):** History of role changes and login activity.
- **Shell Navigation:** Sidebar: **Settings > Security > User Management**.

## 13. Technical Implementation Roadmap (Day 1)
- **Phase 1:** Refine `core/auth.py` and `core/rbac.py` to support dynamic role fetching.
- **Phase 2:** Implement Role CRUD and Permissions Matrix API.
- **Phase 3:** Build User Management UI with `@bes/shared-ui`.
- **Phase 4:** Build Permissions Matrix UI (complex tree/grid).
- **Phase 5:** Integration with Bootstrap to ensure UI reflects new permissions immediately on login.

## 14. Verification & QA Strategy
- **Permission Check:** Create a role with "Sales Read-Only", assign to a user, and verify they cannot POST to `/api/v1/sales/*`.
- **Bootstrap Verification:** Verify that the Shell's sidebar hides modules for which the user has no `read` permission.
- **Audit Verification:** Change a role's permission and verify the change appears in the Role's Timeline.
