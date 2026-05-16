# User Management & RBAC

## 1. Module
Settings

## 2. Name
User Management & Role-Based Access Control (RBAC)

## 3. Description
The User Management & RBAC feature governs system access, authentication, and authorization across the entire Business Execution System (BES). It separates the concept of a digital identity (the User) from physical employment records (the Employee), allowing external stakeholders (auditors, contractors) to access the system without being on the payroll. This module implements a highly granular, context-aware RBAC matrix where permissions are scoped not just globally, but on a per-subsidiary basis. This ensures that a user can hold "Manager" rights in Subsidiary A while being restricted to "Read-Only" in Subsidiary B.

## 4. Depends on
- **Company / Entity Setup (Settings):** Relies heavily on the `core.subsidiaries` table to provide the context for scoped permissions. A user's roles are explicitly tied to specific subsidiary IDs.
- **Employee Master (HR):** (Optional but common) Integrating `core.users` with `hr.employees` to link a system login to an internal employment profile.
- **Core Integration:** Core provides the unified `BESBase` foundation. All authentication and authorization data natively resides in the `core` schema, acting as the absolute gatekeeper for every API request in the system.
- **Other Dependencies:**
  - `@bes/shared-ui` for complex role-matrix grids and access assignment drawers.
  - `Authentication Provider` (e.g., JWT, OAuth2, SAML) for managing the actual login handshakes.

## 5. Feature name, details - the UI as information
**Feature Name:** Identity & Access Management
**Details & UI Information:**
- **User Directory:** A data grid showing all individuals with system access. Columns: `Name`, `Email`, `Linked Employee`, `Status` (Active/Locked), and `Last Login`.
- **Role Matrix Dashboard:** A comprehensive grid where the Y-axis represents specific Roles (e.g., "Finance Manager", "Sales Viewer") and the X-axis represents distinct modules or actions (e.g., "Approve PO", "View GL"). Checkboxes dictate access.
- **Creation/Edit Drawer (User Provisioning):** A Right Panel (Drawer) for assigning access:
  - **Identity Tab:** `First/Last Name`, `Email` (Login ID), `Password Policy`, and `MFA Enforcement`.
  - **Subsidiary Access Tab:** A dynamic list where administrators assign the user to specific subsidiaries and select their corresponding `Role` within that scope.
  - **History/Activity Tab:** Contains the vertical `Timeline` for audit traceability regarding permission elevations or access revocations.
- **Empty States:** "No custom roles defined. You are currently operating with standard system roles."
- **Loading States:** Shimmer effects for the complex role matrix and loading spinners during permission validation.

**User Journey & UX Flow:**
1. User navigates to **Settings > User Management**.
2. User clicks **"Provision User"** -> Right-panel drawer opens.
3. Administrator enters the user's **Email** and basic details.
4. Administrator navigates to the **Access Tab**, selects "Subsidiary A", and assigns the "Inventory Manager" role. They then add "Subsidiary B" and assign the "Viewer" role.
5. User clicks **"Save & Send Invite"** -> System creates the `core.users` record, generates the mapping in `core.user_roles`, and emits a `settings.user.provisioned` event.
6. The new user receives an email, sets their password, and upon login, the frontend shell dynamically renders only the menus and features permitted by their active subsidiary context.

## 6. YAML or sample data structure
### YAML Schema
```yaml
User:
  id: uuid
  email: string
  first_name: string
  last_name: string
  status: enum [ACTIVE, LOCKED, PENDING_INVITE]
  mfa_enabled: boolean
  last_login_at: timestamp
  subsidiary_access:
    - subsidiary_id: uuid
      role_id: uuid
  metadata_: jsonb

Role:
  id: uuid
  name: string
  description: string
  is_system_role: boolean
  permissions:
    - module: string
      action: string (e.g., CREATE, READ, UPDATE, DELETE, APPROVE)
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "user-550e8400-e29b-41d4-a716-446655440000",
    "email": "johndoe@globex.com",
    "first_name": "John",
    "last_name": "Doe",
    "status": "ACTIVE",
    "subsidiary_access": [
      {
        "subsidiary_id": "sub-990e8400-e29b-41d4-a716-446655441111",
        "subsidiary_name": "Globex US",
        "role_id": "role-admin-uuid",
        "role_name": "System Admin"
      },
      {
        "subsidiary_id": "sub-880e8400-e29b-41d4-a716-446655442222",
        "subsidiary_name": "Globex UK",
        "role_id": "role-viewer-uuid",
        "role_name": "Read-Only Viewer"
      }
    ],
    "created_at": "2026-05-16T12:00:00Z"
  },
  "metadata": {
    "version": "1.0"
  },
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/core/users`**: Fetch a paginated list of users.
- **GET `/api/v1/core/roles`**: Fetch available roles and their permission mappings.
- **POST `/api/v1/core/users/provision`**: Create a new user identity and issue an invite.
- **PUT `/api/v1/core/users/{id}/roles`**: Update the subsidiary-to-role mappings for a specific user.
- **PATCH `/api/v1/core/users/{id}/status`**: Lock or unlock a user's account (e.g., during offboarding).

## 8. Database Tables & Architecture
### Schema: Core Module
#### Table: `core.users`
- **Inherits**: `BESBase`
- `email`: String (UNIQUE, NOT NULL)
- `password_hash`: String
- `first_name`: String
- `last_name`: String
- `status`: String (Default: 'PENDING_INVITE')
- `mfa_secret`: String (Encrypted)
- `last_login_at`: Timestamp

#### Table: `core.roles`
- **Inherits**: `BESBase`
- `name`: String (UNIQUE, NOT NULL)
- `description`: String
- `permissions_json`: JSONB (Stores the granular matrix of allowed actions)
- `is_system_role`: Boolean (Prevents deletion of foundational roles like 'Super Admin')

#### Table: `core.user_roles`
- **Inherits**: `BESBase`
- `user_id`: UUID (FK to `core.users`)
- `role_id`: UUID (FK to `core.roles`)
- `subsidiary_id`: UUID (FK to `core.subsidiaries`) - **CRITICAL:** Binds the role to a specific contextual scope.

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `settings.user.provisioned`: Signals an email service to send welcome/onboarding instructions.
  - `settings.user.roles_updated`: Forces a token refresh for active sessions of that user to immediately apply new permissions.
  - `settings.user.locked`: Immediately invalidates all active JWT tokens for that user.
- **Listens To:**
  - `hr.employee.activated`: To automatically trigger user provisioning based on HR data.
  - `hr.employee.terminated`: To immediately transition the user's `status` to `LOCKED`.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion of a user is strictly forbidden to preserve audit trails. Use `status = 'LOCKED'` and `is_deleted = true`.
- **System Role Protection:** Roles flagged with `is_system_role = true` cannot be modified or deleted to ensure baseline system stability.
- **Contextual Enforcement:** The backend middleware MUST validate that the action requested by the user matches the `permissions_json` of the `role_id` assigned to them *specifically for the `subsidiary_id` present in the request header*.
- **Self-Elevation Block:** Administrators cannot assign roles to themselves that grant higher privileges than their current session.

## 11. Security, Audit, and RBAC
- **Roles:**
  - `Super Admin`: Can manage roles, provision users globally, and bypass subsidiary scoping if explicitly configured.
  - `Entity Admin`: Can provision users, but *only* granting them access to the specific subsidiary the Admin controls.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active for a subsidiary, the backend automatically overrides all role permissions for that subsidiary, stripping out `CREATE`, `UPDATE`, and `DELETE` actions at the middleware level, regardless of the role matrix.
- **Audit Trail:** Extremely critical. Every assignment of a role, password reset, or status lock MUST be logged, capturing the `user_id` making the change and the exact timestamp.

## 12. Process Transparency & Workflow Pipeline
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** Typically not needed for user creation as it is immediate, unless an "Access Request" approval workflow is implemented.
  - **Micro View (`Timeline`):** Placed in the "Activity" tab of the User drawer. Logs lineage: "Invited by Admin Alice", "Accepted Invite", "Granted Finance Manager in Sub A by Bob", "Account Locked due to HR Termination".
- **Navigation:** Main Sidebar > Settings > User Management.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `users`, `roles`, and `user_roles`. Implement the core JWT authentication strategy and the strict Subsidiary-Role middleware interceptor.
- **Phase 2: Core Logic & APIs:** Build the User and Role services, ensuring self-elevation protections.
- **Phase 3: Frontend Infrastructure:** Register the User Management component in the `@bes/settings` library. Implement dynamic route guarding in the Shell based on decoded JWT permissions.
- **Phase 4: UI Development:** Build the Role Matrix dashboard and the Subsidiary Access assignment drawer.
- **Phase 5: Event Integration:** Wire up listeners to automatically lock accounts when the HR module emits a termination event.

## 14. Verification & QA Strategy
- **Scoping Enforcement Check:** Provision a user as Admin in Subsidiary A and Viewer in Subsidiary B. Authenticate as that user, switch context to Subsidiary B, and attempt a `POST` request to create a record. Verify the system returns a `403 Forbidden`.
- **Immutability Check:** Attempt to modify the permissions of the 'Super Admin' role (`is_system_role = true`) and verify the rejection.
- **Functional Scenarios:**
  1. Terminate a linked employee in the HR module and verify the associated `core.users` record is instantly locked.
  2. Update a user's role while they are actively logged in and verify the frontend forces a token refresh or logs them out.
- **Integration Test:** Verify that creating a user emits the correct event to trigger the automated delivery of an onboarding email.
