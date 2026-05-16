# Purchase Order Management

## 1. Module
Supply Chain

## 2. Name
Purchase Order Management

## 3. Description
The Purchase Order (PO) Management feature is the cornerstone of the procurement process within the Business Execution System (BES). It allows the organization to formally request and commit to purchasing goods or services from external suppliers. This module ensures procurement compliance by enforcing approval workflows, budget checks, and accurate pricing. It acts as the critical input for downstream operations, specifically informing the Inventory module (Goods Receipt) of expected incoming stock and the Finance module (Accounts Payable) of future liabilities.

## 4. Depends on
- **Supplier / Vendor Master (Supply Chain):** To link the PO to an approved vendor and retrieve default procurement terms.
- **Item / Material Master (Inventory):** To specify exactly what is being ordered, utilizing standard SKUs and UOMs.
- **Core Integration:** Core provides the unified `BESBase` foundation. `supply_chain.purchase_orders` relies on `core.vendors` and `core.items` via foreign keys, while maintaining strict isolation within the `supply_chain` schema.
- **Other Dependencies:**
  - `@bes/shared-ui` for standardized Drawer-based forms, grids, and Process Transparency components.
  - `Finance Module (Budgeting & Forecasting)` (Optional): To validate if the PO amount exceeds the allocated budget before approval.

## 5. Feature name, details - the UI as information
**Feature Name:** Purchase Order Entry & Fulfillment Tracking
**Details & UI Information:**
- **Resource List:** A robust data grid displaying all POs. Columns: `PO Number`, `Vendor`, `Order Date`, `Expected Delivery`, `Total Amount`, `Received %`, `Billed %`, and `Status` (Draft, Pending Approval, Issued, Partially Received, Fulfilled, Closed).
- **Creation/Edit Drawer:** A multi-tab Right Panel (Drawer) for complete PO management:
  - **Header Tab:** `Vendor Selection`, `Order Date`, `Expected Delivery Date`, `Payment Terms`, and `Shipping Address` (Target Warehouse).
  - **Items Grid:** Dynamic entry table for order lines. Columns: `Item`, `Quantity`, `UOM`, `Unit Price` (4-decimal), `Discount %`, `Tax %`, `Line Total`, and `Received Qty` (Read-only, updated by Inventory).
  - **Summary Tab:** Aggregated totals showing `Subtotal`, `Total Tax`, `Total Discount`, and `Grand Total` (enforcing the Money Rule).
  - **History/Activity Tab:** Contains the vertical `Timeline` for strict process auditability.
- **Empty States:** "No purchase orders found. Ready to restock? Create your first PO."
- **Loading States:** Shimmer effects for the items grid during price calculations and skeleton loaders for the header.

**User Journey & UX Flow:**
1. User navigates to **Supply Chain > Purchase Orders**.
2. User clicks **"Create PO"** -> Right-panel drawer opens.
3. User selects a **Vendor** -> System auto-fills payment terms and default buyer.
4. User adds **Items** -> System retrieves default purchase prices; user adjusts quantities.
5. User submits for approval -> Status becomes `Pending Approval` (routing to manager).
6. Manager approves -> User clicks **"Issue to Vendor"** -> Status updates to `Issued` and an event is emitted.
7. Later, Warehouse receives the items (Goods Receipt) -> System updates the `Received %` on the PO in real-time.

## 6. YAML or sample data structure
### YAML Schema
```yaml
PurchaseOrder:
  id: uuid
  subsidiary_id: uuid
  po_number: string
  vendor_id: uuid
  order_date: date
  expected_delivery_date: date
  status: enum [DRAFT, PENDING_APPROVAL, ISSUED, PARTIALLY_RECEIVED, FULFILLED, CLOSED, CANCELLED]
  currency_id: uuid
  total_amount: decimal(20,4)
  items:
    - item_id: uuid
      quantity: decimal(20,4)
      uom_id: uuid
      unit_price: decimal(20,4)
      discount_percent: decimal(20,4)
      tax_amount: decimal(20,4)
      line_total: decimal(20,4)
      received_quantity: decimal(20,4)
      billed_quantity: decimal(20,4)
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "po-550e8400-e29b-41d4-a716-446655440000",
    "subsidiary_id": "sub-990e8400-e29b-41d4-a716-446655441111",
    "po_number": "PO-2026-0042",
    "vendor_id": "vend-gts-001",
    "status": "ISSUED",
    "expected_delivery_date": "2026-06-01",
    "total_amount": "12500.5000",
    "items": [
      {
        "item_id": "item-laptop-uuid",
        "quantity": "10.0000",
        "unit_price": "1250.0500",
        "discount_percent": "0.0000",
        "line_total": "12500.5000",
        "received_quantity": "0.0000",
        "billed_quantity": "0.0000"
      }
    ],
    "created_at": "2026-05-16T14:00:00Z"
  },
  "metadata": {
    "version": "1.0"
  },
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/supply-chain/purchase-orders`**: Fetch paginated list of POs with advanced filters (by status, vendor, delivery date).
- **GET `/api/v1/supply-chain/purchase-orders/{id}`**: Fetch full PO details, including line items and fulfillment progress.
- **POST `/api/v1/supply-chain/purchase-orders`**: Create a new PO (supports atomic creation of header and lines).
- **PUT `/api/v1/supply-chain/purchase-orders/{id}/issue`**: Formally issue the PO to the vendor.
- **PUT `/api/v1/supply-chain/purchase-orders/{id}/close`**: Manually close or short-close a PO (e.g., if the vendor cannot fulfill the remaining balance).

## 8. Database Tables & Architecture
### Schema: Supply Chain Module
#### Table: `supply_chain.purchase_orders`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `po_number`: String (UNIQUE, NOT NULL)
- `vendor_id`: UUID (FK to `core.vendors`)
- `order_date`: Date
- `expected_delivery_date`: Date
- `status`: String (Default: 'DRAFT')
- `total_amount`: Numeric(20,4)

#### Table: `supply_chain.purchase_order_items`
- **Inherits**: `BESBase`
- `purchase_order_id`: UUID (FK to `supply_chain.purchase_orders`)
- `item_id`: UUID (FK to `core.items`)
- `quantity`: Numeric(20,4)
- `received_quantity`: Numeric(20,4) (Default: 0.0000)
- `billed_quantity`: Numeric(20,4) (Default: 0.0000)
- `unit_price`: Numeric(20,4)
- `line_total`: Numeric(20,4)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `supply_chain.po.issued`: Triggers the Inventory module to anticipate incoming stock (shows up in Warehouse receiving queue).
  - `supply_chain.po.cancelled`: Removes the expected stock from the Inventory queue.
- **Listens To:**
  - `inventory.goods_receipt.completed`: Real-time update of `received_quantity` and triggers a status change to `PARTIALLY_RECEIVED` or `FULFILLED`.
  - `finance.ap_invoice.created`: Real-time update of `billed_quantity`.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion is forbidden. Use `is_deleted`.
- **Money Rule:** All pricing, tax, and total fields MUST use 4-decimal precision (`Numeric(20,4)`).
- **Vendor Validation:** A PO cannot be issued if the associated vendor's status is `ON_HOLD` or `INACTIVE`.
- **Immutability:** Once a PO reaches the `ISSUED` state, quantities and prices cannot be modified without creating a formal PO Revision or Change Order.
- **Short Closing:** If a vendor delivers 9 out of 10 items and cannot supply the last one, the PO can be "Short Closed," preventing further goods receipts.

## 11. Security, Audit, and RBAC
- **Roles:**
  - `Buyer`: Can create drafts and submit for approval.
  - `Procurement Manager`: Can approve, issue, and short-close POs.
  - `Warehouse Receiver`: Read-only access to PO details for cross-referencing during Goods Receipt.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, the "Create PO" and "Issue" actions are hidden; users can only view PO history and fulfillment progress.
- **Audit Trail:** Strict logging required for status transitions (especially Approval and Issuance) and any manual short-closures, capturing the user and timestamp.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** POs waiting for manager approval surface on the **Home Dashboard** under the "Action Items / Pending Approvals" pipeline.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** At the top of the PO Drawer. Stages: `Draft` → `Approval` → `Issued` → `Receiving` → `Completed`. Inline "Approve" and "Issue" buttons reside here based on RBAC.
  - **Micro View (`Timeline`):** In the "Activity" tab. Logs: "Approved by Manager Dave", "Issued to Vendor via Email", "5 items received in Warehouse via GRN-101".
- **Traceability:** Extremely critical. End-to-end visualization showing: **`Purchase Order`** → `Goods Receipt` → `Purchase Invoice (AP)` → `Payment`.
- **Navigation:** Main Sidebar > Supply Chain > Purchase Orders.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `purchase_orders` and `purchase_order_items` tables with `BESBase` inheritance.
- **Phase 2: Core Logic & APIs:** Implement PO lifecycle service (draft -> issue) and integration hooks for Inventory.
- **Phase 3: Frontend Infrastructure:** Register the PO component in the `@bes/supply-chain` library.
- **Phase 4: UI Development:** Build the Order Entry form with real-time summary calculations and the `ProcessPipeline` header.
- **Phase 5: Event Integration:** Wire up SSE listeners for Goods Receipt and AP Invoice updates to reflect real-time fulfillment/billing percentages.

## 14. Verification & QA Strategy
- **Scoping Check:** Verify that users in `Subsidiary A` cannot view or approve POs belonging to `Subsidiary B`.
- **Precision Check:** Test multi-line orders with fractional quantities and fractional discounts to ensure the 4-decimal math perfectly aligns with the Grand Total.
- **Functional Scenarios:**
  1. Create a PO, issue it, and verify it appears in the pending Goods Receipt queue in the Inventory module.
  2. Attempt to issue a PO to a vendor marked as `ON_HOLD` and verify the rejection.
  3. Short-close a partially received PO and verify that no further goods receipts can be processed against it.
- **Integration Test:** Execute a `inventory.goods_receipt.completed` event and verify the PO's `received_quantity` and status update automatically without a page refresh.
