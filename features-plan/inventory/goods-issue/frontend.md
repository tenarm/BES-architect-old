# Goods Issue (Outbound)

## 1. Module
Inventory

## 2. Name
Goods Issue (Outbound)

## 3. Description
The Goods Issue (Outbound) feature manages the physical removal of items from the warehouse for external fulfillment (Sales Orders) or internal consumption (Stock Transfers / Scrap). It serves as the bridge between sales commitment and physical delivery, providing warehouse teams with optimized picking lists and packing instructions. This module ensures that stock levels are decremented accurately in real-time, capturing serial numbers, batches, and specific bin locations from which the items were removed.

## 4. Depends on
- **Sales Order Processing (Sales):** To retrieve order-based shipment requirements and customer delivery details.
- **Item / Material Master (Inventory):** To validate item specifications and weight/volume for packing.
- **Warehouse & Location Management (Inventory):** To identify optimal picking paths and ensure stock is removed from the correct bins.
- **Core Integration:** Core provides the unified `BESBase` foundation, allowing `inventory.goods_issues` to maintain logical isolation via `subsidiary_id` while referencing global hub entities like `core.customers`.
- **Other Dependencies:** 
  - `@bes/shared-ui` for standardized Drawer-based forms and picking list visualizations.
  - `Shipping & Logistics APIs` (Optional): For generating shipping labels and tracking numbers.

## 5. Feature name, details - the UI as information
**Feature Name:** Outbound Fulfillment & Picking
**Details & UI Information:** 
- **Fulfillment Queue:** A dashboard view showing "Orders Ready for Picking" (Sales orders confirmed and ready for shipment).
- **Issue Resource List:** A table of all outbound issues. Columns: `GIN #` (Goods Issue Note), `Reference #` (SO/Transfer), `Customer/Destination`, `Ship Date`, `Fulfillment %`, and `Status` (Draft, Picking, Packed, Shipped).
- **Picking List Drawer:** A multi-tab Right Panel (Drawer):
  - **Pick Details Tab:** Optimized list of items to be picked. Columns: `Item`, `Target Bin`, `Quantity to Pick` (4-decimal), `UOM`, and `Picked Status` (Checklist).
  - **Packing Tab:** Verification of packed quantities. Includes fields for `Box Count`, `Total Weight` (4-decimal), and `Dimensions`.
  - **Shipping Tab:** Carrier selection, `Tracking Number`, and `Shipping Label` attachment.
  - **History/Activity Tab:** Contains the vertical `Timeline` for audit traceability.
- **Empty States:** "No pending shipments. All orders are currently fulfilled."
- **Loading States:** Real-time stock reservation indicators and shimmer effects for picking grids.

**User Journey & UX Flow:**
1. User navigates to **Inventory > Goods Issue > Fulfillment Queue**.
2. User selects a **Sales Order** and clicks **"Generate Picking List"**.
3. System generates an optimized path through the **Warehouse Hierarchy** (Bins/Aisles).
4. Warehouse personnel use the **Picking Checklist** to verify items removed from bins.
5. User updates status to **"Packed"** after verifying item counts and box dimensions.
6. User clicks **"Mark as Shipped"** -> System decrements **On-Hand Quantity**, emits an `inventory.goods_issue.shipped` event, and notifies the Sales module.
7. The order status on the Sales side updates to `Shipped` or `Partially Shipped`.

## 6. YAML or sample data structure
### YAML Schema
```yaml
GoodsIssue:
  id: uuid
  subsidiary_id: uuid
  gin_number: string
  reference_type: enum [SALES_ORDER, STOCK_TRANSFER, SCRAP]
  reference_id: uuid
  warehouse_id: uuid
  ship_date: date
  status: enum [DRAFT, PICKING, PACKED, SHIPPED]
  items:
    - item_id: uuid
      requested_qty: decimal(20,4)
      picked_qty: decimal(20,4)
      bin_id: uuid
      batch_number: string (optional)
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "gin-550e8400",
    "subsidiary_id": "sub-990e8400",
    "gin_number": "GIN-2026-9002",
    "reference_type": "SALES_ORDER",
    "reference_id": "so-abc-123",
    "ship_date": "2026-05-16",
    "status": "SHIPPED",
    "items": [
      {
        "item_id": "item-server-uuid",
        "requested_qty": "2.0000",
        "picked_qty": "2.0000",
        "bin_id": "bin-a1-uuid"
      }
    ],
    "created_at": "2026-05-16T11:00:00Z"
  },
  "metadata": {},
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/inventory/goods-issues`**: Fetch list of issues with fulfillment status filters.
- **GET `/api/v1/inventory/goods-issues/{id}/pick-list`**: Generate and retrieve an optimized picking list for a specific GIN.
- **POST `/api/v1/inventory/goods-issues`**: Create a GIN from a Sales Order or Stock Transfer.
- **PUT `/api/v1/inventory/goods-issues/{id}/ship`**: Finalize shipment and decrement stock levels.
- **GET `/api/v1/inventory/sales-orders/pending-fulfillment`**: Fetch SOs ready for picking.

## 8. Database Tables & Architecture
### Schema: Inventory Module
#### Table: `inventory.goods_issues`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `gin_number`: String (UNIQUE, NOT NULL)
- `reference_type`: String (e.g., 'SO', 'TRANS')
- `reference_id`: UUID (Refers to source document)
- `warehouse_id`: UUID (FK to `inventory.warehouses`)
- `status`: String
- `ship_date`: Date

#### Table: `inventory.goods_issue_items`
- **Inherits**: `BESBase`
- `goods_issue_id`: UUID (FK to `inventory.goods_issues`)
- `item_id`: UUID (FK to `core.items`)
- `quantity`: Numeric(20,4)
- `bin_id`: UUID (FK to `inventory.locations`)
- `serial_number`: String (Optional, for high-value tracking)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:** 
  - `inventory.goods_issue.shipped`: Triggers stock decrement and notifies Sales/Finance.
  - `inventory.goods_issue.picking_started`: Notifies Sales that the order is currently being fulfilled.
- **Listens To:** 
  - `sales.order.confirmed`: To populate the "Fulfillment Queue" dashboard.
  - `inventory.stock.reserved`: To highlight reserved quantities in the picking UI.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion forbidden; use `is_deleted`.
- **Money Rule:** All quantity, weight, and volume fields must use 4-decimal precision (`Numeric(20,4)`).
- **Negative Inventory Prevention:** System blocks the shipment if the `picked_qty` exceeds available `On-Hand` stock in the specified bin (unless subsidiary-level overrides are active).
- **Order Locking:** Once a Sales Order is in `Picking` status, its items and quantities are locked in the Sales module to prevent fulfillment mismatch.

## 11. Security, Audit, and RBAC
- **Roles:** 
  - **Warehouse Associate:** Can update picking checklists and pack items.
  - **Warehouse Manager:** Can authorize shipments, handle stock discrepancies, and manage carriers.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, users can view past fulfillment records and their audit history but cannot process new picking lists.
- **Audit Trail:** Log exact timestamps for Picking Start, Packing Completion, and Final Shipment.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** Confirmed Sales Orders waiting for fulfillment surface on the **Warehouse Outbound Dashboard** as the primary work queue.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** At the top of the GIN Drawer. Stages: `Draft` → `Picking` → `Packed` → `Shipped`.
  - **Micro View (`Timeline`):** In the "Activity" tab. Logs: "Picking list generated by Sarah", "Packed in Box #45 by Mark", "Picked up by FedEx Tracking #123".
- **Traceability:** end-to-end visualization showing: `Sales Order` → **`Goods Issue`** → `Carrier Pickup` → `Sales Invoice`.
- **Navigation:** Main Sidebar > Inventory > Goods Issue.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `goods_issues` and `items` extending `BESBase`.
- **Phase 2: Core Logic & APIs:** Implement the Picking Path optimization logic and stock decrement services.
- **Phase 3: Frontend Infrastructure:** Register the GIN module in the Inventory library.
- **Phase 4: UI Development:** Build the Fulfillment Queue and the interactive Picking Checklist Drawer.
- **Phase 5: Event Integration:** Wire up real-time status updates to the Sales Order UI.

## 14. Verification & QA Strategy
- **Scoping Check:** Verify that users in `Subsidiary A` cannot ship orders belonging to `Subsidiary B`.
- **Precision Check:** Test picking of fractional quantities (e.g., `12.5000 Units`) to verify 4-decimal accuracy in stock decrement.
- **Functional Scenarios:**
  1. Generate a GIN from a multi-line Sales Order and verify picking path sorting.
  2. Attempt to ship more than the available stock in a bin and verify the validation block.
  3. Verify that marking an issue as `Shipped` correctly updates the Sales Order's `shipped_quantity`.
- **Integration Test:** Verify that the `inventory.goods_issue.shipped` event triggers the expected status update on the home dashboard's pending pipeline.
