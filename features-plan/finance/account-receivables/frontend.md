# Feature Documentation: Accounts Receivable (AR)

## 1. Module
Finance

## 2. Name
Accounts Receivable

## 3. Description
The Accounts Receivable (AR) feature manages the money owed to the organization by its customers for goods or services delivered. It encompasses the generation of customer invoices, tracking of outstanding balances, sending statements, and processing customer payments (receipts). The AR module integrates directly with the General Ledger (GL) to automate revenue recognition and AR asset tracking, ensuring a real-time view of cash flow and customer debt.

## 4. Depends on
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, allowing Finance and other modules to share the same physical database while maintaining logically isolated schemas like `core` (for the customer hub) and `finance` (for transactional AR invoices).
- **Other Dependencies:** 
  - `@bes/shared-ui` (Data tables, Forms, Process Chain UI, Dashboards)
  - `Finance Module (Chart of Accounts & General Ledger)` (For mapping revenue and auto-generating JEs)
  - `Core Module` (Authentication, RBAC, Subsidiary context)
  - `Customer Management Module` (For customer selection, billing addresses, and terms)
  - `Sales Module` (Optional dependency for generating Invoices from Sales Orders/Fulfillments)

## 5. Feature name, details - the UI as information
**Feature Name:** Accounts Receivable Workspace
**Details & UI Information:**
- **Page Layout:** Dashboard highlighting key AR metrics ("Total Receivables", "Overdue Invoices", "Average Days to Pay"), alongside tabs for Invoices, Payments, and Customer Aging.
- **AR Invoice Form (Header):**
  - `Customer` (Searchable dropdown from Customer Master)
  - `Invoice Number` (Auto-generated BES sequence)
  - `Invoice Date` & `Due Date` (Auto-calculated based on customer terms)
  - `Billing Address` (Auto-populated, editable)
- **AR Invoice Form (Lines Grid):**
  - Editable data grid for revenue/item lines.
  - Columns: `Item/Service` (Lookup), `Revenue Account` (COA lookup, often defaulted by item), `Quantity`, `Rate`, `Amount` (Auto-calc), `Tax Code`.
  - Footer showing `Subtotal`, `Tax`, and `Total Invoice Amount`.
- **Receive Payment UI:**
  - A form to record a customer payment. User selects the Customer, enters the `Payment Amount`, and a grid displays all open invoices for that customer. The user allocates the payment amount across the open invoices.
- **Empty States:** "No outstanding customer invoices."
- **Loading States:** Skeleton loading for KPI cards and data grids.

**User Journey & UX Flow:**
User navigates to Finance > Accounts Receivable > Invoices -> Clicks "New Invoice" (or generates one from a Sales Order) -> Selects Customer -> Adds line items/services -> Saves and Issues the invoice -> System auto-generates GL Journal Entry (Debit AR, Credit Revenue) -> Later, customer pays -> User navigates to "Receive Payment" -> Selects Customer -> Enters payment amount and applies it to the specific invoice -> System updates invoice to "Paid" and generates GL Journal Entry (Debit Cash, Credit AR).

## 6. YAML or sample data structure

```yaml
ARInvoice:
  type: object
  properties:
    id:
      type: string
      format: uuid
    customer_id:
      type: string
      format: uuid
    invoice_number:
      type: string
    invoice_date:
      type: string
      format: date
    due_date:
      type: string
      format: date
    status:
      type: string
      enum: [DRAFT, ISSUED, PARTIALLY_PAID, PAID, VOIDED]
    total_amount:
      type: string
      description: "String-based decimal, 4 places"
    subsidiary_id:
      type: string
      format: uuid
    lines:
      type: array
      items:
        $ref: '#/components/schemas/ARInvoiceLine'

ARInvoiceLine:
  type: object
  properties:
    id:
      type: string
      format: uuid
    ar_invoice_id:
      type: string
      format: uuid
    revenue_account_id:
      type: string
      format: uuid
    description:
      type: string
    quantity:
      type: string
      description: "String-based decimal, 4 places"
    rate:
      type: string
      description: "String-based decimal, 4 places"
    amount:
      type: string
      description: "String-based decimal, 4 places"
```

**Sample JSON Payload:**
```json
{
  "id": "ar-inv-9876-5432",
  "customer_id": "cust-globex-uuid",
  "invoice_number": "INV-10045",
  "invoice_date": "2026-05-15",
  "due_date": "2026-06-14",
  "status": "ISSUED",
  "total_amount": "5000.0000",
  "subsidiary_id": "a0000000-0000-0000-0000-000000000001",
  "lines": [
    {
      "id": "line-1",
      "revenue_account_id": "acc-consulting-revenue-uuid",
      "description": "Implementation Services",
      "quantity": "40.0000",
      "rate": "125.0000",
      "amount": "5000.0000"
    }
  ]
}
```

## 7. Required APIs

- **`GET /api/v1/finance/ar/invoices`**
  - **Description:** Fetch AR Invoices, support filtering by `status`, `customer_id`, and `aging_bracket`.
- **`GET /api/v1/finance/ar/invoices/{id}`**
  - **Description:** Get specific AR Invoice details with lines.
- **`POST /api/v1/finance/ar/invoices`**
  - **Description:** Create a new AR Invoice (Draft).
- **`POST /api/v1/finance/ar/invoices/{id}/issue`**
  - **Description:** Issue the invoice to the customer, locking its details and automatically generating the system GL Journal Entry (Debit AR, Credit Revenue).
- **`POST /api/v1/finance/ar/payments`**
  - **Description:** Receive a payment and apply it against one or multiple issued AR Invoices.

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
  - **Shared Hub:** Global `customers` entity MUST live in the shared `core` schema and act as the single source of truth for all modules.
  - **Isolated Spokes:** Transactional AR tables (like `finance_ar_invoices`) reference the core customer via `customer_id`.
- **Scoping**: Include `subsidiary_id` (UUID/String) on every table for organizational isolation.
- **Precision**: Enforce `Numeric(20,4)` for all financial/money columns per the "Money Rule".
- **File Storage**: If the feature requires file attachments, it MUST use the Centralized Storage Service (no custom blob columns).
- **Schema**: Finance Module (for transactions) & Core Module (for master data hub).
- **Tables:** `finance_ar_invoices`, `finance_ar_invoice_lines`, `finance_ar_payments`
- **Columns (`finance_ar_invoices`):**
  - `customer_id` (UUID, ForeignKey to `core.customers`)
  - `invoice_number` (VARCHAR, Auto-generated sequence)
  - `invoice_date` (DATE)
  - `due_date` (DATE)
  - `total_amount` (Numeric(20,4))
  - `status` (VARCHAR)
  - `subsidiary_id` (UUID, Not Null)
- **Columns (`finance_ar_invoice_lines`):**
  - `ar_invoice_id` (UUID, ForeignKey, Not Null)
  - `revenue_account_id` (UUID, ForeignKey to `finance_accounts`, Not Null)
  - `quantity` (Numeric(20,4))
  - `rate` (Numeric(20,4))
  - `amount` (Numeric(20,4))
  - `subsidiary_id` (UUID, Not Null)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `finance.ar.invoice.created`
  - `finance.ar.invoice.issued` (Listened to by GL to auto-generate a Journal Entry)
  - `finance.ar.payment.received` (Listened to by GL to reduce AR asset and increase Cash)
- **Listens To:**
  - `sales.order.fulfilled` (To trigger draft AR invoice generation if automated billing is enabled).

## 10. Business Rules & Validations
- **GL Automation:** Issuing an AR Invoice automatically posts a Journal Entry. 
  - *Debit:* Accounts Receivable Asset Account
  - *Credit:* Revenue Account(s) (from Invoice Lines)
- **Payment Application:** A received payment cannot be applied for an amount greater than the open balance of the target invoice.
- **Immutability:** Once an AR Invoice is Issued, it cannot be modified. It must be voided or a Credit Memo must be issued.
- **Precision:** Enforce "Money Rule" (4 decimal places) on all amounts, quantities, and rates.

## 11. Security, Audit, and RBAC
- **Roles:**
  - **AR Clerk:** Can create Draft invoices, issue invoices, and record payments.
  - **Finance Manager:** Can void invoices, issue credit memos, and adjust AR aging settings.
  - **Viewer:** Read-only access to AR records.
- **Read-Only Licensing Mode:** "New Invoice" and "Receive Payment" buttons are completely hidden. Forms are rendered in a disabled state.
- **Audit Trail:** Strict logging required for status transitions (Draft -> Issued -> Paid), capturing the user responsible for recognizing revenue.

## 12. Process Transparency & Workflow Pipeline
To eliminate the "blackbox" nature of background processes and show architectural traceability as a Solution Architect, document how this feature integrates into the global workflow:
- **Pending Pipeline (Home Dashboard):** 
  - "Invoices Past Due" (Actionable pipeline for AR collections).
  - "Unapplied Payments" (Payments received but not fully applied to invoices).
- **Process UI Integration:** 
  - **Macro View (`ProcessPipeline`):** Use the horizontal tracker anchored at the **top of the Drawer** (above the form) to show the high-level status (Draft → Issued → Paid). This also serves as the action center for inline approvals.
  - **Micro View (`Timeline`):** Use the vertical activity feed placed inside a **secondary "History/Activity" tab** within the Drawer body. This prevents the dense 5Ws audit data (Who, What, When, Why) from cluttering the editable form details.
- **Traceability:** Extremely critical for end-to-end traceability. The Right-Panel Process Chain must visualize:
  `Sales Order` -> `Fulfillment` -> **`AR Invoice`** -> `System Journal Entry` -> `Customer Payment Receipt`. Users can click on the auto-generated GL Journal Entry directly from the AR Invoice view to see the exact financial impact on the ledger.
- **Shell UI & Navigation:** Sidebar -> Finance -> Accounts Receivable -> Invoices / Receive Payments.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation**: Create migrations for `finance_ar_invoices`, `lines`, and `payments` extending `BESBase`.
- **Phase 2: Core Logic & APIs**: Develop `ARInvoiceService` including the logic to auto-generate JEs upon issuance.
- **Phase 3: Frontend Infrastructure**: Register the AR feature under the `finance` library in NX.
- **Phase 4: UI Development**: Build the AR Dashboard, the Customer Invoice Form, and the Payment Application UI using `@bes/shared-ui`.
- **Phase 5: Event Integration**: Implement Pub/Sub to trigger GL entries and update AR aging metrics in real-time.

## 14. Verification & QA Strategy
- **Scoping Check**: Ensure AR clerks in Subsidiary A cannot invoice Customers strictly assigned to Subsidiary B.
- **Precision Check**: Test `Quantity` * `Rate` calculations using fractional amounts to verify 4-decimal math without rounding errors.
- **Functional Scenarios**:
  1. Create a draft invoice, issue it, and verify the corresponding GL Journal Entry balances.
  2. Attempt to apply a payment larger than the invoice balance (verify rejection).
  3. Execute a partial payment and verify the invoice status changes to `PARTIALLY_PAID` and outstanding balance is correct.
- **Integration Test**: Verify the Process Chain visualization accurately connects the Sales Order to the resulting AR Invoice and its downstream GL Entry.
