# Sales Invoice & Billing

## 1. Module
Sales

## 2. Name
Sales Invoice & Billing

## 3. Description
The Sales Invoice & Billing feature is the final step in the sales execution process. It facilitates the transformation of fulfilled sales orders and delivered goods into formal commercial invoices. This module ensures that customers are accurately billed for products and services provided, capturing tax details, discounts, and payment terms. While the Sales module manages the creation and issuance of the invoice, it integrates directly with the Finance (Accounts Receivable) module to ensure that every issued invoice automatically reflects as a debt in the ledger.

## 4. Depends on
- **Sales Order Processing (Sales):** To retrieve order-based billing quantities and agreed-upon pricing.
- **Delivery / Dispatch Management (Sales):** To enable "Billing based on Delivery" (Invoicing only what has been shipped).
- **Customer Master (Sales):** To retrieve billing addresses, tax registrations, and default payment terms.
- **Accounts Receivable (Finance):** To post the final billing data to the ledger and track payments.
- **Core Integration:** Core provides the unified `BESBase` foundation, allowing the `sales` module to maintain its billing records in the `sales` schema while linking them to the `core.customers` hub.

## 5. Feature name, details - the UI as information
**Feature Name:** Billing & Invoicing Workspace
**Details & UI Information:** 
- **Billing Queue:** A specialized dashboard showing "Orders Ready for Billing" (Sales orders that have been partially or fully shipped but not yet invoiced).
- **Invoice Resource List:** A table of all sales invoices. Columns: `Invoice #`, `SO #`, `Customer`, `Date`, `Total Amount`, `Status` (Draft, Issued, Paid, Overdue), and `Payment Status`.
- **Creation/Edit Drawer:** A multi-tab Right Panel (Drawer):
  - **Billing Details Tab:** Header info including `Customer`, `Invoice Date`, `Due Date`, `Tax ID`, and `Payment Terms`.
  - **Invoice Items Grid:** Automatically populated from the linked Sales Order or Delivery Note. Columns: `Item`, `Billed Qty`, `Unit Price` (4-decimal), `Line Total`, `Tax Amount`, and `Discount`.
  - **Payment Info Tab:** Displays linked payments and credit notes (read-only from Finance).
  - **History/Activity Tab:** Contains the vertical `Timeline` (e.g., "Invoice Issued", "Sent to Customer Email", "Paid via Bank Transfer").
- **Empty States:** "No pending orders to bill. All fulfillments are currently invoiced."
- **Loading States:** Shimmer effects for the billing queue and real-time calculation indicators.

**User Journey & UX Flow:**
1. User navigates to **Sales > Billing Queue**.
2. User selects one or multiple **Shipped Orders** to generate a batch invoice.
3. User reviews the auto-populated **Invoice Draft** in the drawer.
4. System validates the totals using the **Money Rule** (4-decimal places).
5. User clicks **"Issue Invoice"** -> The system locks the record, generates the physical invoice document, and emits a `sales.invoice.issued` event.
6. The event triggers the **Finance AR Module** to recognize revenue and update the customer ledger.
7. User can track the **Payment Status** directly from the Sales Invoice view as finance records payments.

## 6. YAML or sample data structure
### YAML Schema
```yaml
SalesInvoice:
  id: uuid
  subsidiary_id: uuid
  invoice_number: string
  sales_order_id: uuid (optional)
  customer_id: uuid
  invoice_date: date
  due_date: date
  status: enum [DRAFT, ISSUED, PAID, CANCELLED]
  total_amount: decimal(20,4)
  items:
    - sales_order_item_id: uuid (optional)
      item_id: uuid
      quantity: decimal(20,4)
      unit_price: decimal(20,4)
      tax_rate: decimal(20,4)
      line_total: decimal(20,4)
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "inv-550e8400",
    "subsidiary_id": "sub-990e8400",
    "invoice_number": "SINV-2026-1004",
    "sales_order_id": "so-abc-123",
    "customer_id": "cust-globex-uuid",
    "status": "ISSUED",
    "total_amount": "1250.7500",
    "items": [
      {
        "item_id": "item-laptop-uuid",
        "quantity": "1.0000",
        "unit_price": "1200.0000",
        "tax_rate": "5.0000",
        "line_total": "1250.7500"
      }
    ],
    "created_at": "2026-05-16T10:00:00Z"
  },
  "metadata": {},
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/sales/billing-queue`**: Fetch orders eligible for billing.
- **GET `/api/v1/sales/invoices`**: Fetch list of invoices with advanced filtering.
- **POST `/api/v1/sales/invoices`**: Create an invoice (supports manual creation or from SO/Delivery).
- **PUT `/api/v1/sales/invoices/{id}/issue`**: Lock invoice and notify Finance.
- **GET `/api/v1/sales/invoices/{id}/pdf`**: Generate the print-ready billing document.

## 8. Database Tables & Architecture
### Schema: Sales Module
#### Table: `sales.sales_invoices`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `invoice_number`: String (UNIQUE, NOT NULL)
- `customer_id`: UUID (FK to `core.customers`)
- `sales_order_id`: UUID (FK to `sales.sales_orders`, Optional)
- `invoice_date`: Date
- `due_date`: Date
- `total_amount`: Numeric(20,4)
- `status`: String

#### Table: `sales.sales_invoice_items`
- **Inherits**: `BESBase`
- `sales_invoice_id`: UUID (FK to `sales.sales_invoices`)
- `sales_order_item_id`: UUID (FK to `sales.sales_order_items`, Optional)
- `item_id`: UUID (FK to `inventory.items`)
- `quantity`: Numeric(20,4)
- `unit_price`: Numeric(20,4)
- `line_total`: Numeric(20,4)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:** 
  - `sales.invoice.created`: Published when a billing draft is initiated.
  - `sales.invoice.issued`: Triggers the Finance module to create an AR Invoice and GL entry.
  - `sales.invoice.cancelled`: Notifies Finance to issue a credit memo or void the AR entry.
- **Listens To:** 
  - `finance.ar.payment.received`: Updates the payment status and "Remaining Balance" on the Sales Invoice UI.
  - `inventory.dispatch.completed`: Triggers an update to the Billing Queue dashboard.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion forbidden; use `is_deleted`.
- **Money Rule:** All billing calculations must enforce 4-decimal places (`Numeric(20,4)`).
- **Billing Limit:** The quantity billed cannot exceed the quantity shipped (if billing-by-delivery is enforced).
- **Immutability:** Once an invoice is `ISSUED`, it cannot be edited. Corrections require cancellation and re-issuance.

## 11. Security, Audit, and RBAC
- **Roles:** 
  - **Billing Clerk:** Can manage the billing queue and issue invoices.
  - **Sales Manager:** Can approve discounts on invoices and cancel issued invoices.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, the billing queue is disabled; users can only view issued invoices and their history.
- **Audit Trail:** Capture all billing events, specifically tracking who issued the invoice and any manual adjustments to price or tax.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** Orders waiting to be billed appear in the "Ready for Invoicing" pipeline on the Sales Dashboard.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** At the top of the Invoice Drawer. Stages: `Draft` → `Issued` → `Paid`.
  - **Micro View (`Timeline`):** In the "Activity" tab. Logs: "Invoice SINV-1004 generated from SO-2026-42", "Finance confirmed payment of $1250.75".
- **Traceability:** End-to-end visualization showing: `Quotation` → `Sales Order` → `Delivery Note` → **`Sales Invoice`** → `AR Receipt`.
- **Navigation:** Main Sidebar > Sales > Invoices.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `sales_invoices` and items with `BESBase` inheritance.
- **Phase 2: Core Logic & APIs:** Implement the Billing Queue logic and the conversion from SO/Delivery to Invoice.
- **Phase 3: Frontend Infrastructure:** Register the Invoice module in the Sales library.
- **Phase 4: UI Development:** Build the Billing Queue dashboard and the multi-tab Invoice Drawer.
- **Phase 5: Event Integration:** Wire up SSE for real-time payment status updates from Finance.

## 14. Verification & QA Strategy
- **Scoping Check:** Verify that invoices are isolated by `subsidiary_id`.
- **Precision Check:** Test multi-line tax and discount calculations to verify 4-decimal accuracy.
- **Functional Scenarios:**
  1. Generate an invoice from a partially shipped Sales Order.
  2. Verify that issuing an invoice emits the event required for Finance AR creation.
  3. Verify that a payment recorded in Finance reflects correctly on the Sales Invoice status.
- **Integration Test:** Perform a full "Order to Cash" walkthrough, verifying the Process Chain at each step.
