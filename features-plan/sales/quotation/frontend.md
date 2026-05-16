# Quotation Management

## 1. Module
Sales

## 2. Name
Quotation Management

## 3. Description
The Quotation Management feature allows sales teams to create, manage, and track formal price proposals (quotes) sent to prospective or existing customers. It serves as the initial step in the sales workflow, enabling the organization to capture customer requirements, apply pricing strategies, and negotiate terms. Once accepted, a quotation can be seamlessly converted into a Sales Order, ensuring data continuity and process transparency across the lead-to-cash lifecycle.

## 4. Depends on
- **Customer Master (Sales):** To link quotations to valid customer records and retrieve default billing/shipping information.
- **Item Master (Inventory):** To select products or services for the quotation.
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, allowing the `sales` module to store transactional quotation data in an isolated schema while referencing global `core.customers` and `inventory.items`.
- **Other Dependencies:** 
  - `@bes/shared-ui` for standardized Drawer-based forms and Process Transparency components.
  - `Pricing & Discount Management` for automated calculation of line-item and header-level discounts.

## 5. Feature name, details - the UI as information
**Feature Name:** Quotation Entry & Lifecycle
**Details & UI Information:** 
- **Resource List:** A table displaying all quotations. Columns: `Quote #`, `Date`, `Customer`, `Total Amount`, `Valid Until`, and `Status` (Draft, Sent, Accepted, Expired, Converted).
- **Creation/Edit Drawer:** A multi-tab Right Panel (Drawer):
  - **Details Tab:** Header information including `Customer Selection`, `Quote Date`, `Expiry Date`, `Currency`, and `Payment Terms`.
  - **Items Tab:** A dynamic grid to add products/services. Columns: `Item`, `Quantity`, `UOM`, `Unit Price` (4-decimal), `Discount %`, `Tax %`, and `Line Total`.
  - **Summary Tab:** Displays `Subtotal`, `Total Tax`, `Total Discount`, and `Grand Total` (all with 4-decimal precision).
  - **History/Activity Tab:** Contains the vertical `Timeline` showing the quote's evolution (e.g., "Sent to Customer", "Revised Price").
- **Empty States:** "No quotations found. Start by creating a proposal for your next lead."
- **Loading States:** Shimmer effects for the items grid and skeleton loaders for the header summary.

**User Journey & UX Flow:**
1. User navigates to **Sales > Quotations**.
2. User clicks **"New Quotation"** -> Drawer opens.
3. User selects a **Customer** (auto-fills terms) and sets the **Expiry Date**.
4. User adds **Items** to the grid, adjusting quantities and unit prices as needed.
5. System auto-calculates totals using the **Money Rule** (4-decimal places).
6. User clicks **"Send to Customer"** -> Status updates to `Sent`, and an event is emitted.
7. Customer accepts -> User clicks **"Accept"** -> Status updates to `Accepted`.
8. User clicks **"Convert to Order"** -> System generates a Sales Order and marks the quote as `Converted`.

## 6. YAML or sample data structure
### YAML Schema
```yaml
Quotation:
  id: uuid
  subsidiary_id: uuid
  quote_number: string
  customer_id: uuid
  date: date
  expiry_date: date
  status: enum [DRAFT, SENT, ACCEPTED, EXPIRED, CONVERTED]
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
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "quote-abc-123",
    "subsidiary_id": "sub-990e8400",
    "quote_number": "QT-2026-0001",
    "customer_id": "cust-globex-uuid",
    "status": "SENT",
    "total_amount": "1250.7500",
    "items": [
      {
        "item_id": "item-laptop-uuid",
        "quantity": "1.0000",
        "unit_price": "1200.0000",
        "discount_percent": "0.0000",
        "line_total": "1200.0000"
      }
    ],
    "created_at": "2026-05-15T12:00:00Z"
  },
  "metadata": {},
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/sales/quotations`**: Paginated list of quotes.
- **GET `/api/v1/sales/quotations/{id}`**: Detailed view including line items.
- **POST `/api/v1/sales/quotations`**: Create a new quote.
- **PUT `/api/v1/sales/quotations/{id}`**: Update existing quote.
- **POST `/api/v1/sales/quotations/{id}/send`**: Trigger 'Sent' status and notifications.
- **POST `/api/v1/sales/quotations/{id}/convert`**: Create a Sales Order from this quote.

## 8. Database Tables & Architecture
### Schema: Sales Module
#### Table: `sales.quotations`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `quote_number`: String (UNIQUE, NOT NULL)
- `customer_id`: UUID (FK to `core.customers`)
- `status`: String (Default: 'DRAFT')
- `total_amount`: Numeric(20,4)
- `expiry_date`: Date

#### Table: `sales.quotation_items`
- **Inherits**: `BESBase`
- `quotation_id`: UUID (FK to `sales.quotations`)
- `item_id`: UUID (FK to `inventory.items`)
- `quantity`: Numeric(20,4)
- `unit_price`: Numeric(20,4)
- `discount_amount`: Numeric(20,4)
- `tax_amount`: Numeric(20,4)
- `line_total`: Numeric(20,4)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:** 
  - `sales.quotation.created`: Published when a new draft is saved.
  - `sales.quotation.sent`: Published when the quote is sent to the customer.
  - `sales.quotation.converted`: Triggered when converted to a Sales Order.
- **Listens To:** 
  - `inventory.item.price_updated`: To alert the user if a draft quote contains items with outdated pricing.

## 10. Business Rules & Validations
- **Soft Deletes:** Use `is_deleted` flag for logical deletion.
- **Money Rule:** All financial calculations must enforce 4-decimal places (`Numeric(20,4)`).
- **Expiry Validation:** Quotations cannot be converted to orders if the `expiry_date` has passed.
- **Sequence Management:** `quote_number` must follow the organizational naming pattern.

## 11. Security, Audit, and RBAC
- **Roles:** 
  - `Sales Admin`: Full CRUD and status overrides.
  - `Sales Representative`: Can create/edit/send their own quotes.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, the "New Quotation" and "Edit" actions are hidden; users can only view existing quotes and their history.
- **Audit Trail:** Log all status transitions (e.g., Draft -> Sent) and price modifications in the central BES audit system.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** Draft quotations and those nearing expiry surface on the **Home Dashboard** "Action Items" widget.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** At the top of the Drawer. Stages: `Draft` → `Sent` → `Accepted` → `Converted`. Inline "Accept" and "Convert" buttons reside here.
  - **Micro View (`Timeline`):** In the "History" tab. Logs: "Quote sent to customer@email.com", "Price updated by Manager Jane", "Converted to SO #556".
- **Navigation:** Main Sidebar > Sales > Quotations.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Define the `quotations` and `quotation_items` tables inheriting from `BESBase`.
- **Phase 2: Core Logic & APIs:** Implement the Quotation service and conversion logic (Quote -> Order).
- **Phase 3: Frontend Infrastructure:** Register the Quotation module in the Sales library.
- **Phase 4: UI Development:** Build the grid-based item entry form and summary view.
- **Phase 5: Event Integration:** Implement notifications for expired quotes.

## 14. Verification & QA Strategy
- **Scoping Check:** Ensure quotes from `Subsidiary X` do not appear in `Subsidiary Y`.
- **Precision Check:** Verify that a 15.5% discount on a $1250.75 item correctly computes the 4-decimal line total.
- **Functional Scenarios:**
  1. Create a draft, add items, and verify calculations.
  2. Transition quote to 'Sent' and verify the status update.
  3. Attempt to convert an expired quote and verify rejection.
- **Integration Test:** Convert a quote to an order and ensure all line items are mapped correctly.
