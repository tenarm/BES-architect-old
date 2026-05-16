# Backend Implementation Plan: Chart of Accounts (COA)

This document outlines the backend implementation for the Chart of Accounts (COA) feature within the Finance module, adhering to the BES Factory Architecture Rules.

## 1. Module Overview
- **Extension Name:** `finance`
- **Goal:** Provide a robust, hierarchical structure for organizing financial accounts, ensuring data integrity, multi-org isolation, and auditability.

## 2. API Design & Payloads
All endpoints are prefixed with `/api/v1/finance`.

### GET `/api/v1/finance/accounts`
- **Description:** Retrieve a list of accounts. Supports hierarchical tree view via query parameter.
- **Input (Params):** 
  - `format`: Optional string (`tree` or `list`, default: `list`)
  - `is_active`: Optional boolean filter
  - `PaginationParams`: For flat list view.
- **Desired Output:** `paginated_response` for `list` format, or `success_response` for `tree` format containing a nested array of `FinanceAccountRead` objects.

### POST `/api/v1/finance/accounts`
- **Description:** Create a new account.
- **Input (Schema):** `FinanceAccountCreate`
  ```python
  class FinanceAccountCreate(BaseModel):
      account_code: str
      name: str
      description: Optional[str] = None
      account_type: str  # Enum: ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE
      parent_account_id: Optional[uuid.UUID] = None
      is_active: bool = True
  ```
- **Desired Output:** `success_response` with the created `FinanceAccountRead` object.

### GET `/api/v1/finance/accounts/{id}`
- **Description:** Fetch details of a single account.
- **Desired Output:** `success_response` with `FinanceAccountRead`.

### PUT `/api/v1/finance/accounts/{id}`
- **Description:** Update an existing account.
- **Input (Schema):** `FinanceAccountUpdate`
  ```python
  class FinanceAccountUpdate(BaseModel):
      name: Optional[str] = None
      description: Optional[str] = None
      parent_account_id: Optional[uuid.UUID] = None
      is_active: Optional[bool] = None
  ```
- **Desired Output:** `success_response` with the updated `FinanceAccountRead`.

### DELETE `/api/v1/finance/accounts/{id}`
- **Description:** Soft-delete an account.
- **Desired Output:** `success_response` with `status: "deleted"`.

## 3. Business Logic (Services Layer)
Implemented in `finance/services.py`:

- **Uniqueness Validation:** Ensure `account_code` is unique within the current `subsidiary_id`.
- **Hierarchy Validation:**
    - `parent_account_id` (if provided) must exist and belong to the same subsidiary.
    - Prevent circular references (an account cannot be its own parent or ancestor).
    - **Depth Limit:** Enforce a maximum hierarchy depth of 5 levels.
- **Integrity Checks:**
    - **Soft Delete Check:** Prevent deletion if the account has active child accounts or associated General Ledger (GL) transactions.
    - **Type Modification Check:** Prevent changing `account_type` if transactions have already been posted to the account.
- **Tree Construction:** Logic to recursively build the tree structure for the `format=tree` API request.
- **Transactions:** All write operations wrapped in `AsyncSession` transactions.

## 4. RBAC & Security Considerations
- **Required Permissions:**
    - `GET`: `"finance:coa:read"`
    - `POST/PUT/DELETE`: `"finance:coa:write"`
- **Subsidiary Scoping:**
    - `finance_accounts` table inherits from `BESBase`.
    - All queries will automatically include `subsidiary_id` filtering via the `BaseRepository` scoping mechanism.
    - The `X-Subsidiary-Id` header is mandatory for all requests.

## 5. Audit Trail & Soft Deletion
- **Soft Deletion:** Physical deletion is forbidden. The `is_deleted` flag in `BESBase` will be used.
- **Automatic Tracking:** `created_at`, `updated_at`, and `created_by` (derived from the JWT `sub` claim) are automatically handled by `BESBase`.
- **Detailed Audit:** Changes to sensitive fields (`account_code`, `account_type`, `parent_account_id`) will be logged via the Event system for the central Audit log.

## 6. Events & Pub/Sub
- **Events Emitted:**
    - `FINANCE_ACCOUNT_CREATED`: `{ "id": UUID, "code": str, "name": str }`
    - `FINANCE_ACCOUNT_UPDATED`: `{ "id": UUID, "changes": dict }`
    - `FINANCE_ACCOUNT_DELETED`: `{ "id": UUID }`
- **Event Subscriptions:**
    - None currently required from other modules for the core COA management.

## 7. Implementation Roadmap
1. **Directory Structure:** Initialize `extensions/finance/` if not present.
2. **Models (`models.py`):** Define `FinanceAccount` table inheriting from `BESBase`.
3. **Schemas (`schemas.py`):** Define `FinanceAccountCreate`, `FinanceAccountUpdate`, and `FinanceAccountRead`.
4. **Service Layer (`services.py`):**
    - Implement `AccountService` with CRUD and hierarchy validation.
    - Implement tree formatting logic.
5. **Router (`router.py`):**
    - Setup FastAPI router with `require_permission` dependencies.
    - Wire endpoints to `AccountService` methods.
6. **Events (`events.py`):** Implement emitter functions for COA lifecycle events.
7. **Manifest (`manifest.py`):** Register the finance extension and its routes.
8. **Permissions:** Add `finance:coa:read` and `finance:coa:write` to the global permission registry.
9. **Migration:** Create and run the Alembic migration for `finance_accounts`.
