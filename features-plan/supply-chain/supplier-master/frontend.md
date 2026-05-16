# Supplier / Vendor Master

## 1. Module
Supply Chain

## 2. Name
Supplier / Vendor Master

## 3. Description
The Supplier / Vendor Master feature serves as the centralized, authoritative source for all supplier-related information within the Business Execution System (BES). It provides a unified view of the vendors that the organization purchases goods and services from. By maintaining basic identity data in the global `core` hub and procurement-specific configurations in the `supply_chain` spoke, this feature ensures consistency across purchasing, inventory, and accounts payable processes, enabling effective supplier relationship management, performance tracking, and streamlined procurement operations.

## 4. Depends on
- **Core (Master Data):** Relies on the `core` schema for base vendor identity, primary contact information, and shared addresses.
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation. `core.vendors` acts as the single source of truth across all modules, while the `supply_chain` module extends this entity with domain-specific procurement attributes like preferred shipping methods and supplier lead times.
- **Other Dependencies:**
  - `@bes/shared-ui` for standardized Drawer-based forms and Process Transparency components.
  - `Finance Module (Accounts Payable)` for integration regarding payment terms, default currency, and AP ledger tracking.

## 5. Feature name, details - the UI as information
**Feature Name:** Supplier Profile & Management
**Details & UI Information:**
- **Resource List:** A searchable and filterable table displaying all suppliers. Columns include: `Supplier Name`, `Vendor Code`, `Category`, `Status` (Active/Inactive/On Hold), `Rating`, and `Current Balance`.
- **Creation/Edit Drawer:** A multi-tab Right Panel (Drawer) for managing vendor details:
  - **General Tab:** `Supplier Name`, `Internal Code`, `Tax/VAT ID`, `Supplier Category` (e.g., Manufacturer, Distributor, Service Provider), and `Primary Contact Information`.
  - **Procurement Settings Tab:** `Default Buyer`, `Incoterms`, `Minimum Order Value` (4-decimal precision), `Standard Lead Time` (Days), and `Supplier Rating`.
  - **Addresses Tab:** Management of multiple `Remit-To` and `Dispatch` addresses.
  - **Finance Tab:** (Read-only or linked views) `Payment Terms`, `Default Currency`, and `Bank Account Details` (managed via AP).
  - **History/Activity Tab:** Contains the vertical `Timeline` for audit traceability.
- **Empty States:** "No suppliers found. Start by registering your first vendor."
- **Loading States:** Skeleton screens for the resource list and shimmer effects for drawer tab content.

**User Journey & UX Flow:**
1. User navigates to **Supply Chain > Suppliers**.
2. User clicks **"Register Supplier"** -> A right-panel drawer opens.
3. User fills in **General Identity** details (saved to `core.vendors`).
4. User switches to **Procurement Settings** tab to define the `Minimum Order Value` and `Lead Time` (saved to `supply_chain.vendor_details`).
5. User clicks **"Save"** -> The system validates the data, generates a `supply_chain.vendor.registered` event, and refreshes the list.
6. User can later select a supplier to view their **Timeline** in the drawer to see who updated their minimum order value or placed them on hold.

## 6. YAML or sample data structure
### YAML Schema
```yaml
Vendor:
  id: uuid
  subsidiary_id: uuid
  name: string
  code: string
  tax_id: string
  status: enum [ACTIVE, INACTIVE, ON_HOLD]
  procurement_details:
    buyer_id: uuid
    minimum_order_value: decimal(20,4)
    standard_lead_time_days: integer
    incoterms: string
    supplier_rating: integer
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "vend-770e8400-e29b-41d4-a716-446655440000",
    "subsidiary_id": "sub-990e8400-e29b-41d4-a716-446655441111",
    "name": "Global Tech Supplies Ltd.",
    "code": "VEND-GTS-001",
    "tax_id": "VAT-987654321",
    "status": "ACTIVE",
    "procurement_details": {
      "buyer_id": "user-880e8400-e29b-41d4-a716-446655442222",
      "minimum_order_value": "5000.0000",
      "standard_lead_time_days": 14,
      "incoterms": "FOB",
      "supplier_rating": 4
    },
    "created_at": "2026-05-16T12:00:00Z",
    "updated_at": "2026-05-16T12:30:00Z"
  },
  "metadata": {
    "version": "1.0"
  },
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/supply-chain/vendors`**: Fetch a paginated list of suppliers with search and filter support.
- **GET `/api/v1/supply-chain/vendors/{id}`**: Fetch full vendor details, merging core identity and procurement extensions.
- **POST `/api/v1/supply-chain/vendors`**: Register a new supplier. Requires atomic write to `core.vendors` and `supply_chain.vendor_details`.
- **PUT `/api/v1/supply-chain/vendors/{id}`**: Update supplier details.
- **PATCH `/api/v1/supply-chain/vendors/{id}/status`**: Update vendor status (e.g., place ON_HOLD).
- **DELETE `/api/v1/supply-chain/vendors/{id}`**: Logically delete a supplier (sets `is_deleted = true`).

## 8. Database Tables & Architecture
### Schema: Core Module
#### Table: `core.vendors`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL) - Multi-org scoping.
- `name`: String (NOT NULL)
- `code`: String (UNIQUE)
- `tax_id`: String
- `status`: String (Default: 'ACTIVE')

### Schema: Supply Chain Module
#### Table: `supply_chain.vendor_details`
- **Inherits**: `BESBase`
- `vendor_id`: UUID (NOT NULL, FK to `core.vendors.id`) - Linking to the core hub.
- `subsidiary_id`: UUID (NOT NULL)
- `buyer_id`: UUID (FK to `core.users.id`)
- `minimum_order_value`: Numeric(20,4) (Default: 0.0000) - The **Money Rule** precision.
- `standard_lead_time_days`: Integer
- `incoterms`: String
- `supplier_rating`: Integer (1-5 scale)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `supply_chain.vendor.registered`: Triggered when a new vendor is added.
  - `supply_chain.vendor.updated`: Triggered when any attribute changes.
  - `supply_chain.vendor.status_changed`: Specific event for status changes (e.g., ON_HOLD) to immediately alert procurement teams and halt pending POs.
- **Listens To:**
  - `finance.ap_invoice.paid`: React to update potential rating metrics or balance displays.
  - `quality.inspection.failed`: To potentially trigger an automatic review or downgrade of `supplier_rating`.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion is forbidden. Use the `is_deleted` flag.
- **Money Rule:** All currency fields (Minimum Order Value) must enforce 4-decimal places (`Numeric(20,4)`).
- **Unique Identification:** Vendor `code` must be unique within the `subsidiary_id` scope.
- **Hold Status Validation:** Purchase Orders cannot be issued to vendors with a status of `ON_HOLD`.

## 11. Security, Audit, and RBAC
- **Roles:**
  - `Procurement Manager`: Full CRUD access and ability to change vendor status.
  - `Buyer`: Create/Update vendors, but cannot place vendors ON_HOLD.
  - `Viewer`: Read-only access.
- **Read-Only Licensing:** If the tenant has `READONLY_EXTENSIONS`, the "Save" and "Register Supplier" buttons are disabled; only the "Timeline" and "General" tabs remain visible for reference.
- **Audit Trail:** Every change to `status`, `minimum_order_value`, or `buyer_id` must be logged in the central audit system with `previous_state` and `new_state`.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** Vendors created as "Draft" or requiring compliance approval before activation surface on the **Home Dashboard** under the "Pending Supplier Approvals" pipeline.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** Displayed at the top of the Supplier Drawer. Shows stages: `Draft` → `Compliance Review` → `Active` (or `On Hold`). Inline buttons for "Approve" or "Place on Hold" are available here.
  - **Micro View (`Timeline`):** Placed in the "History/Activity" tab of the drawer. Logs the full lineage: "Jane Doe placed vendor On Hold due to quality issues", "System updated rating after 5 successful deliveries".
- **Navigation:** Main Sidebar > Supply Chain > Suppliers. Breadcrumb: `Supply Chain / Suppliers / [Supplier Name]`.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Create `core.vendors` and `supply_chain.vendor_details` tables via Liquibase/Knex migrations inheriting from `BESBase`.
- **Phase 2: Core Logic & APIs:** Implement the Vendor Service layer handling atomic transactions across the core and supply_chain schemas.
- **Phase 3: Frontend Infrastructure:** Register the `SupplierMaster` component in the `@bes/supply-chain` NX library.
- **Phase 4: UI Development:** Build the Supplier List and the multi-tab Drawer using `@bes/shared-ui` components.
- **Phase 5: Event Integration:** Wire up SSE listeners to dynamically update vendor status indicators across the application.

## 14. Verification & QA Strategy
- **Scoping Check:** Create vendors in `Subsidiary A` and verify they are invisible to users in `Subsidiary B`.
- **Precision Check:** Enter a minimum order value of `5000.12345` and verify the system rounds/truncates to `5000.1235` or `5000.1234` based on policy and stores it as `Numeric(20,4)`.
- **Functional Scenarios:**
  1. Create a supplier with basic core details only.
  2. Update a supplier to add procurement-specific extension details.
  3. Verify the Audit Timeline correctly captures a change in `status` to ON_HOLD.
- **Integration Test:** Trigger a status change to ON_HOLD and verify that the system blocks the creation of new Purchase Orders for this vendor.
