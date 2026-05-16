# Sales Order Processing

## 1. Module
Sales

## 2. Name
Sales Order Processing

## 3. Description
The Sales Order Processing feature is the core engine for managing customer purchase commitments. It transforms accepted quotations or direct customer requests into formal internal orders. This module facilitates stock reservation, credit limit validation, and acts as the trigger for downstream activities such as warehouse picking (Delivery/Dispatch) and financial billing (Sales Invoicing). It provides real-time visibility into order status, fulfillment progress, and revenue commitments.

## 4. Depends on
- **Customer Master (Sales):** To validate customer status, credit limits, and retrieve billing/shipping logistics.
- **Item Master (Inventory):** To ensure item availability and retrieve product specifications.
- **Quotation Management (Sales):** Optional dependency; Sales Orders can be generated from accepted Quotations.
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, enabling transactional sales orders to reside in the `sales` schema while maintaining strict data integrity through foreign keys to `core.customers` and `inventory.items`.
- **Other Dependencies:** 
  - `@bes/shared-ui` for standardized Drawer-based forms, Resource Lists, and Process Transparency components.
  - `Inventory Module` for real-time stock availability checks and item reservations.

## 5. Feature name, details - the UI as information
**Feature Name:** Sales Order Management
**Details & UI Information:** 
- **Resource List:** A filterable table displaying all Sales Orders. Columns: `SO #`, `Customer`, `Date`, `Total Amount`, `Fulfillment %`, `Billing %`, and `Status` (Confirmed, Shipped, Invoiced, Closed).
- **Creation/Edit Drawer:** A multi-tab Right Panel (Drawer):
  - **General Tab:** Header details including `Customer`, `Order Date`, `Expected Delivery Date`, `PO Reference #`, and `Shipping Method`.
  - **Items Grid:** A dynamic entry table for order lines. Columns: `Item`, `Quantity`, `UOM`, `Unit Price` (4-decimal), `Discount`, `Tax`, and `Inventory Status` (Available/Backordered).
  - **Fulfillment Tab:** A summary view showing which items have been packed, shipped, and are still pending.
  - **History/Activity Tab:** Contains the vertical `Timeline` for process auditability (e.g., "Order Confirmed by Admin", "Packing Slip #101 Generated").
- **Empty States:** "No sales orders found. Ready to process your first order?"
- **Loading States:** Shimmer effects for the items grid and progress bars for the fulfillment status.

**User Journey & UX Flow:**
1. User navigates to **Sales > Sales Orders**.
2. User clicks **"Create Order"** (or converts from an existing Quotation).
3. User selects **Customer** -> System performs a real-time **Credit Limit Check**.
4. User adds **Items** -> System performs a real-time **Inventory Availability Check**.
5. User clicks **"Confirm Order"** -> System reserves stock and emits a `sales.order.confirmed` event.
6. The order appears in the **Warehouse Pending Pipeline** for picking.
7. As the warehouse ships items, the **Fulfillment %** updates in real-time on the SO resource list.

## 6. YAML or sample data structure
### YAML Schema
```yaml
SalesOrder:
  id: uuid
  subsidiary_id: uuid
  so_number: string
  customer_id: uuid
  quotation_id: uuid (optional)
  order_date: date
  status: enum [DRAFT, CONFIRMED, PARTIALLY_SHIPPED, SHIPPED, INVOICED, CLOSED]
  total_amount: decimal(20,4)
  items:
    - item_id: uuid
      quantity: decimal(20,4)
      shipped_quantity: decimal(20,4)
      invoiced_quantity: decimal(20,4)
      unit_price: decimal(20,4)
      line_total: decimal(20,4)
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "so-550e8400",
    "subsidiary_id": "sub-990e8400",
    "so_number": "SO-2026-00042",
    "customer_id": "cust-globex-uuid",
    "status": "CONFIRMED",
    "total_amount": "5400.0000",
    "items": [
      {
        "item_id": "item-server-uuid",
        "quantity": "2.0000",
        "shipped_quantity": "0.0000",
        "unit_price": "2700.0000",
        "line_total": "5400.0000"
      }
    ],
    "created_at": "2026-05-15T14:00:00Z"
  },
  "metadata": {},
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/sales/orders`**: Fetch paginated list of sales orders with fulfillment/billing status filters.
- **GET `/api/v1/sales/orders/{id}`**: Fetch full SO details and line item progress.
- **POST `/api/v1/sales/orders`**: Create new SO (supports atomicity with line items).
- **PUT `/api/v1/sales/orders/{id}/confirm`**: Confirm the order, triggering stock reservation.
- **PUT `/api/v1/sales/orders/{id}/close`**: Manually close an order (e.g., if partially cancelled).

## 8. Database Tables & Architecture
### Schema: Sales Module
#### Table: `sales.sales_orders`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `so_number`: String (UNIQUE, NOT NULL)
- `customer_id`: UUID (FK to `core.customers`)
- `quotation_id`: UUID (FK to `sales.quotations`, Optional)
- `order_date`: Date
- `status`: String
- `total_amount`: Numeric(20,4)

#### Table: `sales.sales_order_items`
- **Inherits**: `BESBase`
- `sales_order_id`: UUID (FK to `sales.sales_orders`)
- `item_id`: UUID (FK to `inventory.items`)
- `quantity`: Numeric(20,4)
- `shipped_quantity`: Numeric(20,4) (Default: 0.0000)
- `invoiced_quantity`: Numeric(20,4) (Default: 0.0000)
- `unit_price`: Numeric(20,4)
- `line_total`: Numeric(20,4)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:** 
  - `sales.order.confirmed`: Triggers the Inventory module to generate a Picking Slip.
  - `sales.order.fulfillment_updated`: Published when warehouse activity changes shipped quantities.
  - `sales.order.fully_billed`: Published when all items are invoiced.
- **Listens To:** 
  - `inventory.dispatch.completed`: To update `shipped_quantity` and SO status.
  - `finance.invoice.created`: To update `invoiced_quantity`.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion is forbidden; use `is_deleted`.
- **Money Rule:** All currency fields must use 4-decimal precision (`Numeric(20,4)`).
- **Credit Check:** Orders cannot be confirmed if the customer exceeds their credit limit plus a defined organization-level grace percentage.
- **Inventory Check:** System prevents confirmation of orders for "Stock-Out" items unless "Allow Negative Inventory" is enabled at the subsidiary level.

## 11. Security, Audit, and RBAC
- **Roles:** 
  - `Sales Coordinator`: Can create and confirm orders.
  - `Sales Manager`: Can override credit limit blocks and close orders manually.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, users can view order history and fulfillment progress but cannot create or confirm new orders.
- **Audit Trail:** Capture all status changes, price overrides, and fulfillment updates in the central audit system.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** Confirmed orders waiting for shipment surface on the **Warehouse Dashboard**. Orders with credit blocks surface on the **Finance Manager's Dashboard**.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** At the top of the SO Drawer. Stages: `Draft` → `Confirmed` → `Processing` → `Completed`. Inline "Confirm" and "Print Order" buttons available here.
  - **Micro View (`Timeline`):** In the "Activity" tab. Logs: "Order confirmed by John", "3 items shipped via Dispatch #778", "Invoice #990 generated".
- **Navigation:** Main Sidebar > Sales > Sales Orders.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `sales_orders` and `sales_order_items` tables with `BESBase` inheritance.
- **Phase 2: Core Logic & APIs:** Implement Credit Check service and Stock Reservation logic.
- **Phase 3: Frontend Infrastructure:** Register the SO component in the `@bes/sales` library.
- **Phase 4: UI Development:** Build the Order Entry form with real-time summary calculations.
- **Phase 5: Event Integration:** Wire up SSE listeners for warehouse and billing updates.

## 14. Verification & QA Strategy
- **Scoping Check:** Verify that orders are strictly isolated by `subsidiary_id`.
- **Precision Check:** Verify the "Money Rule" applies to complex multi-line order totals.
- **Functional Scenarios:**
  1. Create a Sales Order from an accepted Quotation and verify data mapping.
  2. Attempt to confirm an order for a customer over their credit limit.
  3. Verify fulfillment percentage updates correctly after a partial shipment event.
- **Integration Test:** Ensure `sales.order.confirmed` event triggers the expected downstream warehouse pipeline items.
