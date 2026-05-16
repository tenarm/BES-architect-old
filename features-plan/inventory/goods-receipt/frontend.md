# Goods Receipt (Inbound)

## 1. Module
Inventory

## 2. Name
Goods Receipt (Inbound)

## 3. Description
The Goods Receipt (Inbound) feature is the entry point for all physical inventory entering the organization's facilities. It enables warehouse personnel to record the arrival of items from suppliers (via Purchase Orders) or other internal locations (via Stock Transfers). This module facilitates the verification of received quantities against ordered amounts, triggers quality inspection workflows, and ensures that stock levels are updated in real-time. It acts as the critical bridge between the Supply Chain (Procurement) and Inventory (Storage) modules.

## 4. Depends on
- **Purchase Order Management (Supply Chain):** To retrieve expected items, quantities, and vendor details for reconciliation.
- **Item / Material Master (Inventory):** To validate incoming item specifications and retrieve default storage parameters.
- **Warehouse & Location Management (Inventory):** To identify the target warehouse and specific put-away bins for received stock.
- **Core Integration:** Core provides the unified `BESBase` foundation, allowing `inventory.goods_receipts` to maintain logical isolation via `subsidiary_id` while referencing global hub entities like `core.vendors`.
- **Other Dependencies:** 
  - `@bes/shared-ui` for standardized Drawer-based forms and process-tracking components.
  - `Quality Management Module` (Optional): To trigger inspections for items marked as "Requires QC".

## 5. Feature name, details - the UI as information
**Feature Name:** Inbound Receipt & Verification
**Details & UI Information:** 
- **Receipt Resource List:** A table of all inbound receipts. Columns: `GRN #` (Goods Receipt Note), `Reference #` (PO/Transfer), `Vendor/Source`, `Date Received`, `Status` (Draft, Received, Inspected, Put-Away), and `Total Items`.
- **Creation/Edit Drawer:** A multi-tab Right Panel (Drawer):
  - **Header Tab:** Details including `Vendor`, `Reference Type` (Purchase Order / Stock Transfer), `Target Warehouse`, `Carrier Information`, and `Gate Entry #`.
  - **Items Grid:** A dynamic verification table. Columns: `Item`, `Ordered Qty`, `Received Qty` (4-decimal), `UOM`, `Status` (Ok/Damaged/Shortage), and `Target Bin`.
  - **Attachments Tab:** For uploading digitized copies of Delivery Challans, Lorry Receipts (LR), and photos of damaged goods (using Centralized Storage).
  - **History/Activity Tab:** Contains the vertical `Timeline` for audit traceability.
- **Empty States:** "No inbound shipments recorded. Start by receiving your first Purchase Order."
- **Loading States:** Shimmer effects for item reconciliation and barcode scanning indicators.

**User Journey & UX Flow:**
1. User navigates to **Inventory > Goods Receipt**.
2. User clicks **"New Receipt"** and selects a **Purchase Order** as the source.
3. System auto-populates the **Items Grid** with expected quantities.
4. User performs a physical count and enters the **Received Qty** (validating against the PO).
5. If items are damaged, the user marks the line status and uploads a photo.
6. User clicks **"Complete Receipt"** -> System updates **On-Hand Quantity**, emits an `inventory.goods_receipt.completed` event, and triggers a `finance.ar_invoice` update (if applicable for accruals).
7. If an item requires QC, its status remains `Received` (not yet available for sale) until the Quality module approves it.

## 6. YAML or sample data structure
### YAML Schema
```yaml
GoodsReceipt:
  id: uuid
  subsidiary_id: uuid
  grn_number: string
  source_type: enum [PURCHASE_ORDER, STOCK_TRANSFER, RETURN]
  source_id: uuid
  warehouse_id: uuid
  received_date: date
  status: enum [DRAFT, RECEIVED, INSPECTED, PUT_AWAY]
  items:
    - item_id: uuid
      ordered_qty: decimal(20,4)
      received_qty: decimal(20,4)
      damaged_qty: decimal(20,4)
      bin_id: uuid
      batch_number: string (optional)
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "grn-550e8400",
    "subsidiary_id": "sub-990e8400",
    "grn_number": "GRN-2026-5001",
    "source_type": "PURCHASE_ORDER",
    "source_id": "po-abc-123",
    "received_date": "2026-05-16",
    "status": "RECEIVED",
    "items": [
      {
        "item_id": "item-laptop-uuid",
        "ordered_qty": "10.0000",
        "received_qty": "10.0000",
        "damaged_qty": "0.0000",
        "bin_id": "bin-a1-uuid"
      }
    ],
    "created_at": "2026-05-16T09:00:00Z"
  },
  "metadata": {},
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/inventory/goods-receipts`**: Fetch list of receipts with source-based filtering.
- **GET `/api/v1/inventory/goods-receipts/{id}`**: Detailed view of GRN and item status.
- **POST `/api/v1/inventory/goods-receipts`**: Create a receipt (mapping from PO/Transfer).
- **PUT `/api/v1/inventory/goods-receipts/{id}/complete`**: Finalize receipt and update stock levels.
- **GET `/api/v1/inventory/purchase-orders/pending`**: Fetch POs that are eligible for receipt.

## 8. Database Tables & Architecture
### Schema: Inventory Module
#### Table: `inventory.goods_receipts`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `grn_number`: String (UNIQUE, NOT NULL)
- `source_type`: String (e.g., 'PO', 'TRANS')
- `source_id`: UUID (Refers to source document)
- `warehouse_id`: UUID (FK to `inventory.warehouses`)
- `status`: String
- `received_date`: Date

#### Table: `inventory.goods_receipt_items`
- **Inherits**: `BESBase`
- `goods_receipt_id`: UUID (FK to `inventory.goods_receipts`)
- `item_id`: UUID (FK to `core.items`)
- `quantity`: Numeric(20,4)
- `damaged_quantity`: Numeric(20,4) (Default: 0.0000)
- `bin_id`: UUID (FK to `inventory.locations`)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:** 
  - `inventory.goods_receipt.completed`: Triggers stock increment and notifies Purchasing.
  - `inventory.goods_receipt.quality_required`: Published if received items require QC inspection.
- **Listens To:** 
  - `purchase.order.confirmed`: To populate the "Pending Receipts" dashboard.
  - `quality.inspection.completed`: To transition GRN status from `Inspected` to `Put-Away`.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion forbidden; use `is_deleted`.
- **Money Rule:** All quantity and weight fields must use 4-decimal precision (`Numeric(20,4)`).
- **Tolerance Limit:** System prevents receiving quantities exceeding the PO amount by more than a defined percentage (e.g., 5% over-delivery tolerance).
- **Status Locking:** Once a GRN is `RECEIVED`, item quantities cannot be edited. Corrections require a reversal or debit note.

## 11. Security, Audit, and RBAC
- **Roles:** 
  - **Warehouse Clerk:** Can create and complete receipts.
  - **Inventory Manager:** Can approve over-deliveries and handle damaged goods reconciliation.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, users can view past GRNs and their audit history but cannot record new receipts.
- **Audit Trail:** Log exact timestamps of arrival and the user who verified the physical count.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** Purchase Orders confirmed but not yet received surface on the **Warehouse Inbound Dashboard**.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** At the top of the GRN Drawer. Stages: `Draft` → `Received` → `Inspected` → `Available`.
  - **Micro View (`Timeline`):** In the "Activity" tab. Logs: "Shipment arrived at Gate 1", "Count verified by Mark", "3 items marked as damaged".
- **Traceability:** end-to-end visualization showing: `Purchase Order` → **`Goods Receipt`** → `Quality Inspection` → `Purchase Invoice`.
- **Navigation:** Main Sidebar > Inventory > Goods Receipt.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `goods_receipts` and `items` extending `BESBase`.
- **Phase 2: Core Logic & APIs:** Implement the PO-to-GRN mapping service and quantity validation logic.
- **Phase 3: Frontend Infrastructure:** Register the GRN module in the Inventory library.
- **Phase 4: UI Development:** Build the Receipt Verification form with barcode support.
- **Phase 5: Event Integration:** Implement real-time stock updates and QC triggers.

## 14. Verification & QA Strategy
- **Scoping Check:** Verify that GRNs are strictly isolated by `subsidiary_id`.
- **Precision Check:** Test reception of fractional quantities (e.g., `150.2550 kg`) to verify 4-decimal accuracy.
- **Functional Scenarios:**
  1. Receive a partial shipment and verify the remaining PO quantity stays "Pending".
  2. Mark items as damaged and verify they are moved to a "Damage/Quarantine" bin status.
  3. Attempt to receive more than the allowed tolerance and verify the block.
- **Integration Test:** Verify that completing a GRN correctly triggers a `quality_required` event for specific item categories.
