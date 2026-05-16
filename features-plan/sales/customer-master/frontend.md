# Customer Master

## 1. Module
Sales

## 2. Name
Customer Master

## 3. Description
The Customer Master feature serves as the centralized repository for all customer-related information within the Business Execution System (BES). It provides a 360-degree view of the customer, encompassing basic identity details (stored in the global `core` hub) and sales-specific configurations (stored in the `sales` spoke). This feature enables sales teams to manage customer relationships, credit limits, sales territories, and default billing/shipping preferences, ensuring consistent data across the entire organization.

## 4. Depends on
- **Core (Master Data):** Relies on the `core` schema for base customer identity and shared address records.
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, ensuring that `core.customers` remains the single source of truth while the `sales` module extends this entity with domain-specific attributes like credit policies and sales person assignments.
- **Other Dependencies:** 
  - `@bes/shared-ui` for standardized Drawer-based forms and Process Transparency components.
  - `Finance Module` for integration with Accounts Receivable and credit limit validations.

## 5. Feature name, details - the UI as information
**Feature Name:** Customer Profile & Management
**Details & UI Information:** 
- **Resource List:** A searchable and filterable table displaying all customers. Columns include: `Customer Name`, `Code`, `Category`, `Sales Person`, `Status` (Active/Inactive), and `Current Balance`.
- **Creation/Edit Drawer:** A multi-tab Right Panel (Drawer) for managing customer details:
  - **General Tab:** `Customer Name`, `Internal Code`, `Tax ID`, `Customer Category` (Individual/Corporate), and `Primary Contact Information`.
  - **Sales Settings Tab:** `Default Sales Person`, `Sales Territory`, `Price List`, `Default Discount %`, and `Credit Limit` (Numeric field with 4-decimal precision).
  - **Addresses Tab:** Management of multiple `Billing` and `Shipping` addresses.
  - **History/Activity Tab:** Contains the vertical `Timeline` for audit traceability.
- **Empty States:** "No customers found. Start by creating your first customer record."
- **Loading States:** Skeleton screens for the resource list and shimmer effects for the drawer tabs.

**User Journey & UX Flow:**
1. User navigates to **Sales > Customers**.
2. User clicks **"Create Customer"** -> A right-panel drawer opens.
3. User fills in **General Identity** details (saved to `core.customers`).
4. User switches to **Sales Settings** tab to define the `Credit Limit` and `Sales Person` (saved to `sales.customer_details`).
5. User clicks **"Save"** -> The system validates the data, generates a `customer_created` event, and refreshes the list.
6. User can later select a customer to view their **Timeline** in the drawer to see who changed their credit limit and when.

## 6. YAML or sample data structure
### YAML Schema
```yaml
Customer:
  id: uuid
  subsidiary_id: uuid
  name: string
  code: string
  tax_id: string
  status: enum [ACTIVE, INACTIVE]
  sales_details:
    sales_person_id: uuid
    credit_limit: decimal(20,4)
    default_discount: decimal(20,4)
    price_list_id: uuid
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "cust-550e8400-e29b-41d4-a716-446655440000",
    "subsidiary_id": "sub-990e8400-e29b-41d4-a716-446655441111",
    "name": "Acme Corporation",
    "code": "CUST-ACME-001",
    "tax_id": "TX-123456789",
    "status": "ACTIVE",
    "sales_details": {
      "sales_person_id": "user-880e8400-e29b-41d4-a716-446655442222",
      "credit_limit": "50000.0000",
      "default_discount": "10.0000",
      "price_list_id": "price-770e8400-e29b-41d4-a716-446655443333"
    },
    "created_at": "2026-05-15T10:00:00Z",
    "updated_at": "2026-05-15T10:30:00Z"
  },
  "metadata": {
    "version": "1.0"
  },
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/sales/customers`**: Fetch a paginated list of customers with search and filter support.
- **GET `/api/v1/sales/customers/{id}`**: Fetch full customer details, merging core identity and sales extensions.
- **POST `/api/v1/sales/customers`**: Create a new customer. Requires atomic write to `core.customers` and `sales.customer_details`.
- **PUT `/api/v1/sales/customers/{id}`**: Update customer details.
- **DELETE `/api/v1/sales/customers/{id}`**: Logically delete a customer (sets `is_deleted = true`).

## 8. Database Tables & Architecture
### Schema: Core Module
#### Table: `core.customers`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL) - Multi-org scoping.
- `name`: String (NOT NULL)
- `code`: String (UNIQUE)
- `tax_id`: String
- `status`: String (Default: 'ACTIVE')

### Schema: Sales Module
#### Table: `sales.customer_details`
- **Inherits**: `BESBase`
- `customer_id`: UUID (NOT NULL, FK to `core.customers.id`) - Linking to the core hub.
- `subsidiary_id`: UUID (NOT NULL)
- `sales_person_id`: UUID (FK to `core.users.id`)
- `credit_limit`: Numeric(20,4) (Default: 0.0000) - The **Money Rule** precision.
- `default_discount`: Numeric(20,4) (Default: 0.0000)
- `price_list_id`: UUID

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:** 
  - `sales.customer.created`: Triggered when a new customer is added.
  - `sales.customer.updated`: Triggered when any attribute (core or sales-specific) changes.
  - `sales.customer.credit_limit_changed`: Specific event for credit limit adjustments to trigger finance alerts.
- **Listens To:** 
  - `finance.invoice.paid`: React to update the "Current Balance" display in the customer list.
  - `core.user.status_changed`: To handle cases where a linked Sales Person is deactivated.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion is forbidden. Use `is_deleted` flag.
- **Money Rule:** All currency and percentage fields (Credit Limit, Discount) must enforce 4-decimal places (`Numeric(20,4)`).
- **Unique Identification:** Customer `code` must be unique within the `subsidiary_id` scope.
- **Credit Validation:** Sales orders cannot be confirmed if the customer's outstanding balance exceeds the `credit_limit`.

## 11. Security, Audit, and RBAC
- **Roles:** 
  - `Sales Manager`: Full CRUD access and credit limit adjustment rights.
  - `Sales Rep`: Create/Update customers, but cannot modify `credit_limit`.
  - `Viewer`: Read-only access.
- **Read-Only Licensing:** If the tenant has `READONLY_EXTENSIONS`, the "Save" button and editable fields are disabled; only the "Timeline" and "General" tabs remain visible for reference.
- **Audit Trail:** Every change to `credit_limit`, `status`, or `sales_person_id` must be logged in the central audit system with `previous_state` and `new_state`.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** If a customer is created as a "Draft" or requires approval for a high credit limit, it surfaces on the **Home Dashboard** under the "Pending Approvals" pipeline.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** Displayed at the top of the Customer Drawer. Shows stages: `Draft` → `Active` (or `On Hold`). Inline buttons for "Activate" or "Place on Hold" are available here.
  - **Micro View (`Timeline`):** Placed in the "History/Activity" tab of the drawer. Logs the full lineage: "John Doe changed Credit Limit from 10k to 50k", "System updated balance after Invoice #123".
- **Navigation:** Main Sidebar > Sales > Customers. Breadcrumb: `Sales / Customers / [Customer Name]`.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Create `core.customers` and `sales.customer_details` tables via Liquibase/Knex migrations inheriting from `BESBase`.
- **Phase 2: Core Logic & APIs:** Implement the Customer Service layer handling atomic transactions across the core/sales schemas.
- **Phase 3: Frontend Infrastructure:** Register the `CustomerMaster` component in the `@bes/sales` NX library.
- **Phase 4: UI Development:** Build the Customer List and the multi-tab Drawer using `@bes/shared-ui` components.
- **Phase 5: Event Integration:** Wire up SSE listeners to refresh the customer balance in real-time when finance events occur.

## 14. Verification & QA Strategy
- **Scoping Check:** Create customers in `Subsidiary A` and verify they are invisible to users in `Subsidiary B`.
- **Precision Check:** Enter a credit limit of `50000.12345` and verify the system rounds/truncates to `50000.1235` or `50000.1234` based on policy and stores it as `Numeric(20,4)`.
- **Functional Scenarios:**
  1. Create a customer with basic core details only.
  2. Update a customer to add sales-specific extension details.
  3. Verify the Audit Timeline correctly captures a change in `credit_limit`.
- **Integration Test:** Trigger a `finance.invoice.paid` event and verify the customer's balance updates on the UI without a page refresh.
