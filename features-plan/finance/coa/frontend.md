# Feature Documentation: Chart of Accounts (COA)

## 1. Module
Finance

## 2. Name
Chart of Accounts

## 3. Description
The Chart of Accounts (COA) is the foundational structural element of the BES Finance module. It provides a categorized listing of all accounts used by an organization's general ledger to record financial transactions. The COA feature allows finance administrators to create, read, update, and organize their accounting hierarchy (Assets, Liabilities, Equity, Revenue, and Expenses), enabling accurate financial reporting and transaction categorization.

## 4. Depends on
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, allowing Finance and other modules to share the same physical database while maintaining logically isolated schemas like `finance_accounts`.
- **Other Dependencies:** 
  - `@bes/shared-ui` (Tree views, Data tables, Forms)
  - `Core Module` (Authentication, RBAC, Subsidiary context)
  - `BES Audit System` (for tracking changes to accounts)

## 5. Feature name, details - the UI as information
**Feature Name:** COA Manager
**Details & UI Information:**
- **Page Layout:** Split view. Left pane contains a searchable, collapsible Tree View of the account hierarchy. Right pane displays the details of the selected account or a form to create/edit an account.
- **Tree View:** Nodes represent accounts. Parent accounts can be expanded to show child accounts. Visual indicators (icons) differentiate account types.
- **Data Table View:** An alternative flat list view with sortable columns: `Account Code`, `Name`, `Type`, `Parent Account`, `Status`.
- **Form Inputs:** 
  - `Account Code` (Text, unique per subsidiary)
  - `Account Name` (Text)
  - `Account Type` (Dropdown: Asset, Liability, Equity, Revenue, Expense)
  - `Parent Account` (Searchable dropdown, recursive tree structure)
  - `Description` (Textarea)
  - `Is Active` (Toggle)
- **Empty States:** "No accounts configured. Start by creating your first account."
- **Loading States:** Skeleton loaders for the tree view and right-pane form during data fetching.

**User Journey & UX Flow:**
User navigates to Finance > Chart of Accounts -> Sees the hierarchical tree of existing accounts -> Clicks "New Account" in the header -> Form appears in the right pane -> User enters Account Code (e.g., "1000"), Name ("Cash"), selects Type ("Asset"), and saves -> Success toast appears, and the new account is appended to the tree view in real-time.

## 6. YAML or sample data structure

```yaml
FinanceAccount:
  type: object
  properties:
    id:
      type: string
      format: uuid
    account_code:
      type: string
    name:
      type: string
    description:
      type: string
    account_type:
      type: string
      enum: [ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE]
    parent_account_id:
      type: string
      format: uuid
      nullable: true
    is_active:
      type: boolean
    subsidiary_id:
      type: string
      format: uuid
```

**Sample JSON Payload:**
```json
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "account_code": "1010",
  "name": "Operating Bank Account",
  "description": "Main checking account for operating expenses",
  "account_type": "ASSET",
  "parent_account_id": "c13ab20c-58cc-4372-a567-0e02b2c3d480",
  "is_active": true,
  "subsidiary_id": "a0000000-0000-0000-0000-000000000001",
  "current_balance": "150500.0000"
}
```
*(Note: `current_balance` is a computed field using the string-based Decimal "Money Rule" for display purposes, but is not stored on the COA table directly).*

## 7. Required APIs

- **`GET /api/v1/finance/accounts`**
  - **Description:** Fetch all accounts for the active subsidiary, formatted as a flat list or nested tree based on query params (`?format=tree`).
  - **Response:** Standard BES Envelope returning an array of `FinanceAccount` objects.
- **`GET /api/v1/finance/accounts/{id}`**
  - **Description:** Get specific account details.
- **`POST /api/v1/finance/accounts`**
  - **Description:** Create a new account.
  - **Validation:** `account_code` must be unique within the `subsidiary_id`. `parent_account_id` must exist and belong to the same subsidiary.
- **`PUT /api/v1/finance/accounts/{id}`**
  - **Description:** Update an existing account.
  - **Validation:** Cannot change `account_type` if transactions exist against this account.
- **`DELETE /api/v1/finance/accounts/{id}`**
  - **Description:** Soft-delete an account.
  - **Validation:** Fails if the account has any associated ledger transactions or active child accounts.

**Envelope Standard Example:**
```json
{
  "status": "success",
  "data": { ... },
  "metadata": { "timestamp": "2026-05-15T12:00:00Z" },
  "error": null
}
```

## 8. Database Tables & Architecture
- **Foundation**: All tables MUST inherit from `BESBase` (providing `id`, `created_at`, `updated_at`, `created_by`, `is_deleted`, `metadata_`).
- **Master Data Management (Hub-and-Spoke)**: 
  - **Shared Hub:** The `finance_accounts` entity acts as a primary hub for all financial transactions across BES. While logically part of the Finance schema, it is referenced by Sales (revenue accounts), Inventory (asset accounts), and AP/AR.
  - **Isolated Spokes:** Transactional tables in other modules reference these accounts via `account_id`.
- **Scoping**: Include `subsidiary_id` (UUID/String) on every table for organizational isolation.
- **Precision**: Enforce `Numeric(20,4)` for all financial/money columns per the "Money Rule" (though COA itself stores structural data, calculated balances must follow this).
- **File Storage**: If the feature requires file attachments, it MUST use the Centralized Storage Service (no custom blob columns).
- **Schema**: Finance Module
- **Table Name:** `finance_accounts`
- **Columns:**
  - `account_code` (VARCHAR, Not Null)
  - `name` (VARCHAR, Not Null)
  - `description` (TEXT)
  - `account_type` (VARCHAR, Not Null)
  - `parent_account_id` (UUID, ForeignKey to `finance_accounts.id`, Nullable)
  - `is_active` (BOOLEAN, Default True)
  - `subsidiary_id` (UUID, Not Null, for organizational isolation)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `finance.account.created` (Payload: account ID, subsidiary ID)
  - `finance.account.updated` (Payload: account ID, fields changed)
  - `finance.account.deleted` (Payload: account ID)
- **Listens To:**
  - `subsidiary.context.changed` (To trigger a refetch of the COA tree for the newly selected subsidiary).

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion is strictly FORBIDDEN. Deleting an account sets `is_deleted = true`.
- **Constraint:** Accounts with associated GL entries cannot be soft-deleted; they can only be marked `is_active = false`.
- **Hierarchy Depth Limit:** Maximum nesting depth for parent-child accounts is 5 levels to prevent recursive querying performance degradation.
- **Uniqueness:** `account_code` must be unique per `subsidiary_id`.

## 11. Security, Audit, and RBAC
- **Roles:**
  - **Finance Admin:** Full CRUD access to the Chart of Accounts.
  - **Finance Viewer / General User:** Read-only access to view the COA structure and select accounts in dropdowns.
- **Read-Only Licensing Mode:** If the tenant has a `READONLY_EXTENSIONS` license, the "New Account", "Edit", and "Delete" buttons are hidden. Form fields become disabled.
- **Audit Trail:** Every creation, modification, or soft-deletion of an account logs an event to the BES audit system tracking `user_id`, `timestamp`, `previous_state`, and `new_state` (especially tracking changes to the hierarchical structure).

## 12. Process Transparency & Workflow Pipeline
To eliminate the "blackbox" nature of background processes and show architectural traceability as a Solution Architect, document how this feature integrates into the global workflow:
- **Pending Pipeline (Home Dashboard):** If a draft COA requires approval (e.g., enterprise feature for strict compliance), it will appear in the Home pending pipeline as "Pending Account Approval".
- **Process UI Integration:** 
  - **Macro View (`ProcessPipeline`):** Use the horizontal tracker anchored at the **top of the Drawer** (above the form) to show the high-level status (Draft → Pending → Approved). This also serves as the action center for inline approvals.
  - **Micro View (`Timeline`):** Use the vertical activity feed placed inside a **secondary "History/Activity" tab** within the Drawer body. This prevents the dense 5Ws audit data (Who, What, When, Why) from cluttering the editable form details.
- **Traceability:** The creation of an account does not typically span multiple modules on its own, but it is a prerequisite for the Journal Entry process. Audit history for an account can be visualized in a right-panel timeline.
- **Shell UI & Navigation:** Located under Sidebar -> Finance -> Chart of Accounts.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation**: Create the `finance_accounts` table migration extending `BESBase`.
- **Phase 2: Core Logic & APIs**: Develop the FinanceAccountService, recursive querying logic for the tree, and the REST endpoints.
- **Phase 3: Frontend Infrastructure**: Ensure the `finance` library is structured correctly in the NX workspace.
- **Phase 4: UI Development**: Build the split-pane COA Manager using `@bes/shared-ui` tree and form components.
- **Phase 5: Event Integration**: Hook up SSE for real-time tree updates when other users modify the COA.

## 14. Verification & QA Strategy
- **Scoping Check**: Create accounts under Subsidiary A. Log in as a user restricted to Subsidiary B and verify the accounts from A are invisible and API access is blocked (403).
- **Precision Check**: (N/A directly for COA structure, but critical for GL entries linking to these accounts).
- **Functional Scenarios**:
  1. Create a root account.
  2. Create a child account referencing the root.
  3. Attempt to create a child account exceeding the 5-level depth limit (verify rejection).
  4. Attempt to delete an account that has a child (verify rejection).
- **Integration Test**: Publish `finance.account.created` and verify the frontend tree view updates without a hard refresh.
