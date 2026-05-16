# Feature Documentation: Purchase Invoices

## 1. Module
Finance

## 2. Name
Purchase Invoices

## 3. Description
The Purchase Invoices feature focuses specifically on the capture, validation, and processing of supplier bills that are backed by formalized purchasing workflows. While standard Accounts Payable (AP) often handles direct expenses (like utility bills) and payment runs, the Purchase Invoices feature enforces strict **2-way and 3-way matching** against Purchase Orders (POs) and Item Receipts. It ensures the organization only recognizes liability for goods and services that were actually ordered and received, automating inventory cost capitalization and variance tracking.

## 4. Depends on
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, allowing Finance and other modules to share the same physical database while maintaining logically isolated schemas like `core` (for master hubs) and `finance` (for transactional purchase invoices).
- **Other Dependencies:** 
  - `@bes/shared-ui` (Data grids, Matching Wizards, Process Chain UI)
  - `Purchasing Module` (Requires Purchase Orders and Item Receipts for matching)
  - `Inventory Module` (For item master data and cost allocation)
  - `Finance Module (Vendors, AP, GL)` (For supplier details and generating the final AP liability)
  - `Core Module` (Authentication, Subsidiary context)

## 5. Feature name, details - the UI as information
**Feature Name:** Purchase Invoice Processing
**Details & UI Information:**
- **Page Layout:** A primary dashboard showing invoices requiring matching, tolerance exceptions, and processed invoices.
- **PO Matching Wizard (Creation Flow):**
  - **Step 1: Select Vendor & PO.** User selects a Vendor, and a grid displays all open POs and pending Item Receipts.
  - **Step 2: Line Item Population.** Upon selecting a PO/Receipt, the system auto-populates the invoice lines with the expected quantities and agreed PO rates.
  - **Step 3: Variance Adjustment.** The user enters the actual invoice number and totals from the physical supplier bill. If the billed amount differs from the PO/Receipt amount, a variance is highlighted.
- **Purchase Invoice Form:**
  - **Header:** `Vendor`, `Vendor Invoice #`, `Invoice Date`, `Due Date`, `Total Amount Billed`, `Matched PO #`.
  - **Lines Grid:** `Item`, `PO Quantity`, `Received Quantity`, `Billed Quantity` (editable), `PO Rate`, `Billed Rate` (editable), `Amount`, `Tax Code`.
  - **Totals Panel:** Subtotal, Tax, Freight Allocation, Total Billed, and **Variance Amount**.
- **Empty States:** "No pending purchase orders to invoice."
- **Loading States:** Skeleton loading grids during the PO matching lookup.

**User Journey & UX Flow:**
User receives a physical bill from a supplier -> Navigates to Finance > Purchase Invoices -> Clicks "Create from PO" -> Enters PO number -> System loads the lines showing 10 laptops ordered, 10 received -> User confirms the billed rate matches the PO rate -> Submits the invoice -> System performs a 3-way match. Since everything matches, it auto-approves the invoice, creates the AP Liability, and updates the GL. If the billed rate was 10% higher than the PO, the system flags it as an "Exception" and routes it to a Procurement Manager for override approval.

## 6. YAML or sample data structure

```yaml
PurchaseInvoice:
  type: object
  properties:
    id:
      type: string
      format: uuid
    vendor_id:
      type: string
      format: uuid
    source_po_id:
      type: string
      format: uuid
      nullable: true
    vendor_invoice_number:
      type: string
    invoice_date:
      type: string
      format: date
    status:
      type: string
      enum: [DRAFT, EXCEPTION, MATCHED, APPROVED, VOIDED]
    total_billed_amount:
      type: string
      description: "String-based decimal, 4 places"
    variance_amount:
      type: string
      description: "String-based decimal, 4 places"
    subsidiary_id:
      type: string
      format: uuid
    lines:
      type: array
      items:
        $ref: '#/components/schemas/PurchaseInvoiceLine'

PurchaseInvoiceLine:
  type: object
  properties:
    id:
      type: string
      format: uuid
    purchase_invoice_id:
      type: string
      format: uuid
    item_id:
      type: string
      format: uuid
    source_po_line_id:
      type: string
      format: uuid
    billed_quantity:
      type: string
    billed_rate:
      type: string
    line_variance_amount:
      type: string
```

**Sample JSON Payload:**
```json
{
  "id": "pi-8877-6655",
  "vendor_id": "vendor-dell-uuid",
  "source_po_id": "po-10023",
  "vendor_invoice_number": "INV-DELL-9988",
  "invoice_date": "2026-05-15",
  "status": "MATCHED",
  "total_billed_amount": "10000.0000",
  "variance_amount": "0.0000",
  "subsidiary_id": "a0000000-0000-0000-0000-000000000001",
  "lines": [
    {
      "id": "pi-line-1",
      "item_id": "item-laptop-xps-uuid",
      "source_po_line_id": "po-line-1",
      "billed_quantity": "10.0000",
      "billed_rate": "1000.0000",
      "line_variance_amount": "0.0000"
    }
  ]
}
```

## 7. Required APIs

- **`GET /api/v1/finance/purchase-invoices`**
  - **Description:** Fetch Purchase Invoices. Filter by `status` (e.g., EXCEPTION).
- **`GET /api/v1/finance/purchase-invoices/{id}`**
  - **Description:** Get specific PI details with matching data.
- **`POST /api/v1/finance/purchase-invoices/match-po`**
  - **Description:** Given a PO ID, returns the theoretical invoice lines based on what has been received but not yet billed.
- **`POST /api/v1/finance/purchase-invoices`**
  - **Description:** Create the PI. The backend engine automatically calculates variances against the PO/Receipt and sets the status to MATCHED or EXCEPTION.
- **`POST /api/v1/finance/purchase-invoices/{id}/approve-exception`**
  - **Description:** Allows an authorized manager to override a tolerance exception and approve the invoice.

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
  - **Shared Hub:** Global entities (Master Data like `vendors` and `inventory_items`) MUST live in the shared `core` schema and act as the single source of truth.
  - **Isolated Spokes:** Transactional tables (like `finance_purchase_invoices`) reference the core master data via `vendor_id` and `item_id`.
- **Scoping**: Include `subsidiary_id` (UUID/String) on every table for organizational isolation.
- **Precision**: Enforce `Numeric(20,4)` for all financial/money columns per the "Money Rule".
- **File Storage**: If the feature requires file attachments, it MUST use the Centralized Storage Service (no custom blob columns).
- **Schema**: Finance Module (for transactions) & Core Module (for master data hubs).
- **Tables:** `finance_purchase_invoices`, `finance_purchase_invoice_lines`
- **Columns (`finance_purchase_invoices`):**
  - `vendor_id` (UUID, ForeignKey to `core.vendors`)
  - `source_po_id` (UUID, Nullable)
  - `vendor_invoice_number` (VARCHAR)
  - `total_billed_amount` (Numeric(20,4))
  - `variance_amount` (Numeric(20,4))
  - `status` (VARCHAR)
  - `subsidiary_id` (UUID, Not Null)
- **Columns (`finance_purchase_invoice_lines`):**
  - `purchase_invoice_id` (UUID, ForeignKey, Not Null)
  - `item_id` (UUID, ForeignKey to `inventory_items`, Nullable)
  - `source_po_line_id` (UUID, Nullable)
  - `billed_quantity` (Numeric(20,4))
  - `billed_rate` (Numeric(20,4))
  - `subsidiary_id` (UUID, Not Null)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `finance.pi.created`
  - `finance.pi.exception_flagged` (Triggers notification to Procurement Managers)
  - `finance.pi.approved` (Listened to by AP to create the payable liability, and by GL to reverse accrued "Received Not Invoiced" accounts)
- **Listens To:**
  - `purchasing.receipt.created` (To update the "Available to Invoice" queues).

## 10. Business Rules & Validations
- **3-Way Matching Engine:** The system compares:
  1. PO Quantity & Rate
  2. Item Receipt Quantity
  3. Billed Quantity & Rate
- **Tolerance Levels:** Subsidiaries configure tolerance levels (e.g., "Allow up to 5% or $50 price variance"). If the billed amount exceeds this, the status becomes `EXCEPTION`.
- **GL Automation:** Upon approval, the PI triggers a GL entry that:
  - *Debits:* Accrued Purchases (Clearing the "Received Not Invoiced" liability generated during Item Receipt)
  - *Debits:* Price Variance Account (if any)
  - *Credits:* Accounts Payable
- **Immutability:** Approved PIs cannot be altered.

## 11. Security, Audit, and RBAC
- **Roles:**
  - **AP Clerk:** Can create PIs and match them. Cannot approve exceptions.
  - **Procurement Manager / Finance Admin:** Has authority to use the `approve-exception` API to override variances.
- **Read-Only Licensing Mode:** Forms disabled, action buttons hidden.
- **Audit Trail:** Strict logging when a user approves an `EXCEPTION`, capturing the exact variance amount and the user ID who authorized the overpayment.

## 12. Process Transparency & Workflow Pipeline
To eliminate the "blackbox" nature of background processes and show architectural traceability as a Solution Architect, document how this feature integrates into the global workflow:
- **Pending Pipeline (Home Dashboard):** 
  - "Invoices Pending Exception Approval" (For Managers).
  - "POs Ready to Invoice" (For AP Clerks).
- **Process UI Integration:** 
  - **Macro View (`ProcessPipeline`):** Use the horizontal tracker anchored at the **top of the Drawer** (above the form) to show the high-level status (Draft → Matched/Exception → Approved). This also serves as the action center for inline approvals.
  - **Micro View (`Timeline`):** Use the vertical activity feed placed inside a **secondary "History/Activity" tab** within the Drawer body. This prevents the dense 5Ws audit data (Who, What, When, Why) from cluttering the editable form details.
- **Traceability:** The ultimate showcase for Process Transparency. The Right-Panel Process Chain must visualize the exact lifecycle:
  `Purchase Order` -> `Item Receipt(s)` -> **`Purchase Invoice`** -> `GL Journal Entry` -> `AP Payment`. From the PI, users can see exactly which physical receipt lines matched to which invoice lines.
- **Shell UI & Navigation:** Sidebar -> Finance -> Purchase Invoices.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation**: Create DB migrations extending `BESBase`.
- **Phase 2: Core Logic & APIs**: Build the Matching Engine service that calculates 3-way match variances against POs and Receipts.
- **Phase 3: Frontend Infrastructure**: Register under the `finance` NX library.
- **Phase 4: UI Development**: Build the Matching Wizard UI, ensuring clear visual indicators for line-level variances.
- **Phase 5: Integration**: Implement Pub/Sub so that an approved PI automatically flows into the standard AP Payments queue.

## 14. Verification & QA Strategy
- **Scoping Check**: Ensure cross-subsidiary matching is blocked (cannot invoice a PO belonging to Sub A against a Vendor in Sub B).
- **Precision Check**: Test exact 4-decimal matching to ensure floating-point rounding doesn't trigger false exceptions.
- **Functional Scenarios**:
  1. **Perfect Match:** Invoice matches PO perfectly. Verify auto-approval.
  2. **Tolerance Match:** Invoice is $2 higher (within $5 tolerance). Verify auto-approval and variance booking.
  3. **Exception Match:** Invoice is $100 higher. Verify status goes to `EXCEPTION` and requires manager override.
- **Integration Test**: Verify the Process Chain accurately links the original PO to the resulting PI and GL entries.
