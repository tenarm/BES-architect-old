# Feature Documentation: Accounts Payable (AP)

## 1. Module
Finance

## 2. Name
Accounts Payable

## 3. Description
The Accounts Payable (AP) feature manages an organization's short-term obligations to its vendors and suppliers. It encompasses the end-to-end lifecycle of capturing vendor bills (Invoices), routing them for approval, scheduling payments based on vendor terms, and ultimately executing the payments. The AP module integrates seamlessly with the General Ledger (GL) to automate the recording of accrued expenses and liability reductions, ensuring accurate financial reporting.

## 4. Depends on
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, allowing Finance and other modules to share the same physical database while maintaining logically isolated schemas like `core` (for the vendor hub) and `finance` (for transactional AP bills).
- **Other Dependencies:** 
  - `@bes/shared-ui` (Data tables, Forms, Process Chain UI, Wizards)
  - `Finance Module (Chart of Accounts & General Ledger)` (For mapping expenses and auto-generating JEs)
  - `Core Module` (Authentication, RBAC, Subsidiary context)
  - `Vendor Management Module` (For selecting vendors and auto-populating payment terms)
  - `Purchasing Module` (Optional dependency for 2-way or 3-way PO matching)

## 5. Feature name, details - the UI as information
**Feature Name:** Accounts Payable Workspace
**Details & UI Information:**
- **Page Layout:** Dashboard with KPI cards at the top (e.g., "Total Outstanding", "Due this Week"), followed by a tabbed interface (Unpaid Invoices, Paid Invoices, Approvals).
- **AP Invoice Form (Header):**
  - `Vendor` (Searchable dropdown from Vendor Master)
  - `Invoice Number` (Vendor's reference number)
  - `Invoice Date` & `Due Date` (Auto-calculated based on vendor terms)
  - `Payment Terms` (Dropdown)
- **AP Invoice Form (Lines Grid):**
  - Editable data grid for expense lines.
  - Columns: `Expense Account` (COA lookup), `Description`, `Amount` (Numeric input), `Department/Class` (For cost center allocation).
  - Footer showing `Total Invoice Amount`.
- **Bill Payment Wizard:**
  - A multi-step flow to select unpaid invoices across vendors, group them by vendor, select a funding source (e.g., Operating Bank Account), and generate Payment records.
- **Empty States:** "You have no outstanding bills to pay."
- **Loading States:** Skeleton loading for KPI cards and data grids.

**User Journey & UX Flow:**
User navigates to Finance > Accounts Payable > Bills -> Clicks "New Bill" -> Selects Vendor -> Enters Vendor Invoice # and Amount -> Adds expense lines mapping to the correct COA accounts -> Submits for Approval. 
Once approved, the AP Manager navigates to the "Pay Bills" wizard -> Selects the approved bill -> Selects Bank Account -> Executes Payment -> The system automatically updates the bill status to "Paid" and generates the corresponding GL Journal Entries.

## 6. YAML or sample data structure

```yaml
APInvoice:
  type: object
  properties:
    id:
      type: string
      format: uuid
    vendor_id:
      type: string
      format: uuid
    vendor_invoice_number:
      type: string
    invoice_date:
      type: string
      format: date
    due_date:
      type: string
      format: date
    status:
      type: string
      enum: [DRAFT, PENDING_APPROVAL, APPROVED, PARTIALLY_PAID, PAID, CANCELLED]
    total_amount:
      type: string
      description: "String-based decimal, 4 places"
    subsidiary_id:
      type: string
      format: uuid
    lines:
      type: array
      items:
        $ref: '#/components/schemas/APInvoiceLine'

APInvoiceLine:
  type: object
  properties:
    id:
      type: string
      format: uuid
    ap_invoice_id:
      type: string
      format: uuid
    expense_account_id:
      type: string
      format: uuid
    amount:
      type: string
      description: "String-based decimal, 4 places"
    memo:
      type: string
```

**Sample JSON Payload:**
```json
{
  "id": "ap-inv-1234-5678",
  "vendor_id": "vendor-acme-corp-uuid",
  "vendor_invoice_number": "INV-2026-991",
  "invoice_date": "2026-05-01",
  "due_date": "2026-05-31",
  "status": "APPROVED",
  "total_amount": "12500.0000",
  "subsidiary_id": "a0000000-0000-0000-0000-000000000001",
  "lines": [
    {
      "id": "line-1",
      "expense_account_id": "acc-software-expense-uuid",
      "amount": "12500.0000",
      "memo": "Annual Cloud Hosting Renewal"
    }
  ]
}
```

## 7. Required APIs

- **`GET /api/v1/finance/ap/invoices`**
  - **Description:** Fetch AP Invoices, support filtering by `status`, `vendor_id`, and `due_date`.
- **`GET /api/v1/finance/ap/invoices/{id}`**
  - **Description:** Get specific AP Invoice details with lines.
- **`POST /api/v1/finance/ap/invoices`**
  - **Description:** Create a new AP Invoice (Draft).
  - **Validation:** Vendor must be active. Sum of line amounts must equal the `total_amount` header (if provided).
- **`POST /api/v1/finance/ap/invoices/{id}/approve`**
  - **Description:** Approve the invoice, locking its details and automatically generating the system GL Journal Entry (Debit Expense, Credit AP).
- **`POST /api/v1/finance/ap/payments`**
  - **Description:** Issue a payment against one or multiple approved AP Invoices.

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
  - **Shared Hub:** Global `vendors` entity MUST live in the shared `core` schema and act as the single source of truth for all modules.
  - **Isolated Spokes:** Transactional AP tables (like `finance_ap_invoices`) reference the core vendor via `vendor_id`.
- **Scoping**: Include `subsidiary_id` (UUID/String) on every table for organizational isolation.
- **Precision**: Enforce `Numeric(20,4)` for all financial/money columns per the "Money Rule".
- **File Storage**: If the feature requires file attachments (e.g., vendor PDFs), it MUST use the Centralized Storage Service (no custom blob columns).
- **Schema**: Finance Module (for transactions) & Core Module (for master data hub).
- **Tables:** `finance_ap_invoices`, `finance_ap_invoice_lines`, `finance_ap_payments`
- **Columns (`finance_ap_invoices`):**
  - `vendor_id` (UUID, ForeignKey to `core.vendors`)
  - `vendor_invoice_number` (VARCHAR)
  - `invoice_date` (DATE)
  - `due_date` (DATE)
  - `total_amount` (Numeric(20,4))
  - `status` (VARCHAR)
  - `subsidiary_id` (UUID, Not Null)
- **Columns (`finance_ap_invoice_lines`):**
  - `ap_invoice_id` (UUID, ForeignKey, Not Null)
  - `expense_account_id` (UUID, ForeignKey to `finance_accounts`, Not Null)
  - `amount` (Numeric(20,4))
  - `subsidiary_id` (UUID, Not Null)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `finance.ap.invoice.created`
  - `finance.ap.invoice.approved` (Listened to by GL to auto-generate a Journal Entry)
  - `finance.ap.payment.issued` (Listened to by GL to reduce AP liability and reduce Cash)
- **Listens To:**
  - `vendor.details.updated` (To ensure vendor terms and active status are synchronized).

## 10. Business Rules & Validations
- **GL Automation:** Approving an AP Invoice automatically posts a Journal Entry. The AP user does not manually create the GL entry.
  - *Debit:* Expense Account(s) (from Invoice Lines)
  - *Credit:* Accounts Payable Liability Account (configured at subsidiary level)
- **Payment Application:** Payments cannot exceed the outstanding balance of the invoice.
- **Immutability:** Once an AP Invoice is Approved, it cannot be modified. It must be cancelled or a Vendor Credit must be issued.
- **Precision:** Enforce "Money Rule" (4 decimal places) on all amounts.

## 11. Security, Audit, and RBAC
- **Roles:**
  - **AP Clerk:** Can create Draft invoices and submit them. Cannot approve or pay.
  - **AP Manager:** Can approve invoices and execute bill payments.
  - **Viewer:** Read-only access to AP records.
- **Read-Only Licensing Mode:** "New Bill", "Approve", and "Pay" buttons are entirely removed from the UI.
- **Audit Trail:** Strict logging required for status transitions (e.g., Draft -> Pending -> Approved), capturing the exact user who approved the financial liability.

## 12. Process Transparency & Workflow Pipeline
To eliminate the "blackbox" nature of background processes and show architectural traceability as a Solution Architect, document how this feature integrates into the global workflow:
- **Pending Pipeline (Home Dashboard):** 
  - "Invoices Pending Your Approval" (For AP Managers).
  - "Bills Due This Week" (Actionable pipeline for AP Managers).
- **Process UI Integration:** 
  - **Macro View (`ProcessPipeline`):** Use the horizontal tracker anchored at the **top of the Drawer** (above the form) to show the high-level status (Draft → Pending Approval → Approved → Paid). This also serves as the action center for inline approvals.
  - **Micro View (`Timeline`):** Use the vertical activity feed placed inside a **secondary "History/Activity" tab** within the Drawer body. This prevents the dense 5Ws audit data (Who, What, When, Why) from cluttering the editable form details.
- **Traceability:** Extremely critical for AP. The Right-Panel Process Chain must visualize:
  `Purchase Order (if matched)` -> `Item Receipt` -> **`AP Invoice`** -> `System Journal Entry` -> `Bill Payment`. Users must be able to click on the auto-generated GL Journal Entry directly from the AP Invoice view to see the exact financial impact.
- **Shell UI & Navigation:** Sidebar -> Finance -> Accounts Payable -> Bills / Payments.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation**: Create migrations for `finance_ap_invoices`, `lines`, and `payments` extending `BESBase`.
- **Phase 2: Core Logic & APIs**: Develop `APInvoiceService` including the logic to auto-generate JEs upon approval.
- **Phase 3: Frontend Infrastructure**: Register the AP feature under the `finance` library in NX.
- **Phase 4: UI Development**: Build the AP Dashboard, the Invoice Form, and the "Pay Bills" Wizard using `@bes/shared-ui`.
- **Phase 5: Event Integration**: Implement Pub/Sub to trigger GL entries and update dashboard KPIs in real-time.

## 14. Verification & QA Strategy
- **Scoping Check**: Ensure AP clerks in Subsidiary A cannot see or select Vendors strictly assigned to Subsidiary B.
- **Precision Check**: Enter line items with fractional cents and ensure the system correctly sums to the `total_amount` using 4-decimal math.
- **Functional Scenarios**:
  1. Create a bill, submit for approval, approve it, and verify the corresponding GL Journal Entry balances.
  2. Attempt to pay a bill that is only in "Draft" status (verify rejection).
  3. Execute a partial payment and verify the invoice status changes to `PARTIALLY_PAID` and outstanding balance is correct.
- **Integration Test**: Verify the Process Chain visualization accurately connects the AP Invoice to its downstream Payment and GL Entry.
