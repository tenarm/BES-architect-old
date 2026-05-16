# Feature Documentation: Tax Management

## 1. Module
Finance

## 2. Name
Tax Management

## 3. Description
The Tax Management feature centralizes the configuration, calculation, and reporting of indirect taxes (such as VAT, GST, Sales Tax, and Withholding Tax) across the entire BES ecosystem. It acts as a universal tax engine that automatically calculates tax liabilities or credits on all applicable sales and purchase transactions (Invoices, Receipts). It allows finance teams to define complex tax codes, effective-dated tax rates, and tax jurisdictions, ultimately generating the aggregated Tax Liability reports required for statutory government filings.

## 4. Depends on
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, allowing Finance and other modules to share the same physical database while maintaining logically isolated schemas like `finance_tax_codes`.
- **Other Dependencies:** 
  - `@bes/shared-ui` (Data tables, Forms, Date-range filters, Reporting grids)
  - `Finance Module (COA, AP, AR, GL)` (Relies on AP/AR to trigger tax events and maps taxes to specific GL Liability accounts)
  - `Core Module` (Authentication, Subsidiary context, Currency formatting)

## 5. Feature name, details - the UI as information
**Feature Name:** Tax Configuration & Reporting
**Details & UI Information:**
- **Page Layout:** A primary Dashboard split into two main sections: **Configuration** (managing codes/rates) and **Reporting** (generating tax returns).
- **Tax Code Form:**
  - `Code ID` (e.g., `VAT-20`, `GST-5`)
  - `Name` / `Description`
  - `Tax Type` (Dropdown: Sales, Purchase, Both)
  - `Linked GL Account` (Dropdown from COA, where the liability/credit is accrued)
- **Tax Rate History Grid:**
  - A nested grid within the Tax Code profile. Contains `Effective Date`, `End Date`, and `Percentage (%)`. Ensures historical transactions are not affected when tax rates change.
- **Tax Liability Report View:**
  - A comprehensive pivot-style grid filtering by `Date Range`.
  - Columns: `Tax Code`, `Taxable Base Amount`, `Calculated Tax`, `Adjustments`, `Total Tax Due`.
  - Drill-down capability: Clicking on a `Calculated Tax` number opens a modal showing the exact AP/AR invoices that contributed to that sum.
- **Empty States:** "No active tax codes. Please configure your jurisdiction's taxes."
- **Loading States:** Skeleton loaders for the reporting grid, which can be computationally heavy.

**User Journey & UX Flow:**
User navigates to Finance > Tax Management -> Clicks "New Tax Code" -> Enters "VAT Standard" and maps it to the "VAT Liability" GL account -> Sets an effective rate of 20% starting Jan 1st.
Later, when an AR Clerk creates a Sales Invoice and applies the "VAT Standard" code to a line item, the Tax Engine automatically calculates the 20% amount. At month-end, the Tax Manager runs the Tax Liability Report, reviews the aggregated totals by code, drills down to verify a few large transactions, and uses these numbers to file their external government return.

## 6. YAML or sample data structure

```yaml
TaxCode:
  type: object
  properties:
    id:
      type: string
      format: uuid
    code_id:
      type: string
    name:
      type: string
    tax_type:
      type: string
      enum: [SALES, PURCHASE, BOTH]
    gl_account_id:
      type: string
      format: uuid
    is_active:
      type: boolean
    subsidiary_id:
      type: string
      format: uuid
    rates:
      type: array
      items:
        $ref: '#/components/schemas/TaxRate'

TaxRate:
  type: object
  properties:
    id:
      type: string
      format: uuid
    tax_code_id:
      type: string
      format: uuid
    percentage:
      type: string
      description: "String-based decimal, 4 places"
    effective_from:
      type: string
      format: date
    effective_to:
      type: string
      format: date
      nullable: true
```

**Sample JSON Payload:**
```json
{
  "id": "tax-1122-3344",
  "code_id": "VAT-UK-STD",
  "name": "UK Standard VAT",
  "tax_type": "BOTH",
  "gl_account_id": "gl-vat-liability-uuid",
  "is_active": true,
  "subsidiary_id": "a0000000-0000-0000-0000-000000000001",
  "rates": [
    {
      "id": "rate-1",
      "percentage": "20.0000",
      "effective_from": "2020-01-01",
      "effective_to": null
    }
  ]
}
```

## 7. Required APIs

- **`GET /api/v1/finance/taxes/codes`**
  - **Description:** Fetch configured tax codes. Heavily cached for dropdown population in AP/AR forms.
- **`POST /api/v1/finance/taxes/codes`**
  - **Description:** Create a new tax code and its initial rate.
- **`POST /api/v1/finance/taxes/calculate`**
  - **Description:** Utility endpoint used by frontend forms. Takes a base `amount` and a `tax_code_id`, checks the `effective_date`, and returns the computed tax amount using 4-decimal precision.
- **`GET /api/v1/finance/taxes/reports/liability`**
  - **Description:** Analytical endpoint to aggregate tax ledgers over a specific `start_date` and `end_date`.

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
  - **Shared Hub:** Global `finance_tax_codes` act as a hub for all transaction-tax calculations across Sales and Finance modules.
  - **Isolated Spokes:** Invoices and Receipts in other modules reference these tax codes via `tax_code_id`.
- **Scoping**: Include `subsidiary_id` (UUID/String) on every table for organizational isolation.
- **Precision**: Enforce `Numeric(20,4)` for all financial/money columns per the "Money Rule".
- **File Storage**: If the feature requires file attachments, it MUST use the Centralized Storage Service (no custom blob columns).
- **Schema**: Finance Module
- **Tables:** `finance_tax_codes`, `finance_tax_rates`, `finance_tax_ledgers`
- **Columns (`finance_tax_codes`):**
  - `code_id` (VARCHAR)
  - `name` (VARCHAR)
  - `gl_account_id` (UUID, ForeignKey to `finance_accounts`)
  - `subsidiary_id` (UUID)
- **Columns (`finance_tax_rates`):**
  - `tax_code_id` (UUID, ForeignKey)
  - `percentage` (Numeric(20,4))
  - `effective_from` (DATE)
  - `effective_to` (DATE, Nullable)
- **Columns (`finance_tax_ledgers`):**
  - Hidden table that mirrors GL entries specifically for rapid tax reporting. Links `source_transaction_id` (e.g., AP Invoice ID) to the `tax_code_id`, `taxable_base_amount`, and `tax_amount`.

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `finance.tax.code.updated` (Triggers frontend clients to invalidate their cached tax dropdowns)
- **Listens To:**
  - `finance.ap.invoice.approved` & `finance.ar.invoice.issued` (Triggers the tax engine to write records to the `finance_tax_ledgers` table for reporting).

## 10. Business Rules & Validations
- **Effective Dating Rule:** The tax rate applied to a transaction is strictly determined by the `transaction_date` on the invoice, NOT the date the invoice is entered into the system. This ensures historical corrections use the correct historical rate.
- **Tax Overrides:** Users can manually override a calculated tax amount on an invoice (e.g., due to vendor rounding differences), but the override cannot exceed a globally configured tolerance (e.g., `0.05` cents).
- **Rate Continuity:** A single `tax_code` cannot have overlapping `effective_from` and `effective_to` dates in its rates array.

## 11. Security, Audit, and RBAC
- **Roles:**
  - **Tax Manager / Finance Admin:** Full access to create codes, change rates, and view liability reports.
  - **AP/AR Clerks:** Read-only access to select tax codes in dropdowns. No access to the Tax Configuration dashboard.
- **Read-Only Licensing Mode:** Forms disabled, rate grids become uneditable.
- **Audit Trail:** Extremely strict logging when a new `TaxRate` is added or modified. The system must log the exact user who altered the tax calculus for the organization.

## 12. Process Transparency & Workflow Pipeline
To eliminate the "blackbox" nature of background processes and show architectural traceability as a Solution Architect, document how this feature integrates into the global workflow:
- **Pending Pipeline (Home Dashboard):** 
  - "Upcoming Tax Deadlines" (If statutory filing dates are configured in the system).
- **Process UI Integration:** 
  - **Macro View (`ProcessPipeline`):** Use the horizontal tracker anchored at the **top of the Drawer** (above the form) to show the high-level status (Draft → Active → Deprecated). This also serves as the action center for inline approvals.
  - **Micro View (`Timeline`):** Use the vertical activity feed placed inside a **secondary "History/Activity" tab** within the Drawer body. This prevents the dense 5Ws audit data (Who, What, When, Why) from cluttering the editable form details.
- **Traceability:** Tax isn't an independent workflow, but it intercepts others. The Process Chain UI for an AP/AR Invoice should explicitly show a sub-node indicating: `Tax Engine Applied [VAT-20] -> Tax Ledger Entry Recorded`.
- **Shell UI & Navigation:** Sidebar -> Finance -> Tax Management.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation**: Create DB migrations for `finance_tax_codes`, `rates`, and `ledgers` extending `BESBase`.
- **Phase 2: Core Logic & APIs**: Build the core `TaxCalculatorEngine` that takes an amount, code, and date, and returns the exact 4-decimal tax figure.
- **Phase 3: Frontend Infrastructure**: Register under the `finance` NX library.
- **Phase 4: UI Development**: Build the Configuration Dashboard and the complex Liability Reporting pivot grid using `@bes/shared-ui`.
- **Phase 5: Integration**: Inject the `TaxCalculatorEngine` into the AP and AR invoice saving workflows to auto-populate the `finance_tax_ledgers`.

## 14. Verification & QA Strategy
- **Scoping Check**: Ensure Subsidiary A cannot see or apply Tax Codes strictly assigned to Subsidiary B.
- **Precision Check**: Test the Tax Calculator with fractional bases (e.g., `$10.33` @ `17.5%` rate) to ensure exact 4-decimal rounding aligns perfectly with the GL Journal Entry.
- **Functional Scenarios**:
  1. Create a transaction using an active tax rate and verify the GL accrues the correct liability.
  2. Create a backdated transaction and verify the engine selects the historical tax rate, not the current one.
  3. Attempt to create overlapping effective dates for a single tax code (verify validation rejection).
- **Integration Test**: Run the Tax Liability Report and ensure the total exactly matches the balance of the linked GL Tax Liability account in the main General Ledger.
