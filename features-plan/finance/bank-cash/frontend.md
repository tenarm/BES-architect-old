# Feature Documentation: Bank & Cash Management

## 1. Module
Finance

## 2. Name
Bank & Cash Management

## 3. Description
The Bank & Cash Management feature provides centralized control over an organization's liquidity. It handles the setup of internal bank accounts and petty cash funds, facilitates inter-bank funds transfers, and provides tools for importing electronic bank statements (e.g., CSV, OFX, MT940). The core of this feature is the **Bank Reconciliation** engine, which allows finance teams to match physical bank statement lines against system-recorded transactions (AP payments, AR receipts, JEs), ensuring the system's cash balance perfectly mirrors reality.

## 4. Depends on
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, allowing Finance and other modules to share the same physical database while maintaining logically isolated schemas like `finance_bank_accounts`.
- **Other Dependencies:** 
  - `@bes/shared-ui` (Data tables, Split-pane matching UI, Process Chain UI)
  - `Finance Module (COA, AP, AR, GL)` (Relies on payments and receipts generated in other sub-modules)
  - `Core Module` (Authentication, Subsidiary context)
  - `BES Audit System` (For tracking reconciliation approvals)

## 5. Feature name, details - the UI as information
**Feature Name:** Bank Reconciliations & Cash Workspace
**Details & UI Information:**
- **Page Layout:** A primary Dashboard showing KPI cards for all active bank accounts (Name, GL Balance, Bank Balance, Unreconciled Amount).
- **Bank Account Form:** Setup for new accounts. `Account Name`, `Account Number`, `Routing/Sort Code`, `Currency`, `Linked GL Account` (from COA).
- **Funds Transfer Form:** Simple form to move money between two internal bank accounts. `From Account`, `To Account`, `Amount`, `Date`, `Memo`.
- **Bank Reconciliation Workspace:**
  - **Header:** `Bank Account`, `Statement Date`, `Starting Balance`, `Ending Balance` (entered from physical statement).
  - **Matching Interface (Split Pane):**
    - **Left Side (Bank Data):** Imported lines from the bank statement (`Date`, `Description`, `Amount`).
    - **Right Side (System Data):** Unreconciled system transactions (`AP Payments`, `AR Receipts`, `JEs`).
    - **Actions:** Select a bank line, select a system line, click "Match". An "Auto-Match" button attempts to pair lines by date and exact amount.
  - **Footer:** `Cleared Balance`, `Statement Ending Balance`, `Difference` (must be 0.0000 to finalize).
- **Empty States:** "All transactions are reconciled for this period."
- **Loading States:** Progress bar during statement import and auto-matching algorithms.

**User Journey & UX Flow:**
User navigates to Finance > Bank & Cash -> Selects a Bank Account -> Clicks "New Reconciliation" -> Enters the Statement Ending Balance from their physical bank statement -> Uploads the CSV bank statement -> The Matching Workspace opens. The user clicks "Auto-Match" to clear 80% of transactions. For the remaining 20%, they manually pair bank fees or bundled deposits to system entries. Once the "Difference" hits 0.0000, they click "Complete Reconciliation", which locks the matched transactions.

## 6. YAML or sample data structure

```yaml
BankAccount:
  type: object
  properties:
    id:
      type: string
      format: uuid
    name:
      type: string
    account_number:
      type: string
    currency:
      type: string
    gl_account_id:
      type: string
      format: uuid
    subsidiary_id:
      type: string
      format: uuid

BankReconciliation:
  type: object
  properties:
    id:
      type: string
      format: uuid
    bank_account_id:
      type: string
      format: uuid
    statement_date:
      type: string
      format: date
    starting_balance:
      type: string
    ending_balance:
      type: string
    status:
      type: string
      enum: [DRAFT, IN_PROGRESS, COMPLETED]
    subsidiary_id:
      type: string
      format: uuid
```

**Sample JSON Payload:**
```json
{
  "id": "recon-1234-5678",
  "bank_account_id": "bank-operating-uuid",
  "statement_date": "2026-05-31",
  "starting_balance": "150000.0000",
  "ending_balance": "165400.0000",
  "status": "IN_PROGRESS",
  "subsidiary_id": "a0000000-0000-0000-0000-000000000001"
}
```

## 7. Required APIs

- **`GET /api/v1/finance/bank-accounts`**
  - **Description:** Fetch list of internal bank accounts.
- **`POST /api/v1/finance/bank-transfers`**
  - **Description:** Execute a funds transfer. Automatically generates a Journal Entry debiting the receiving bank's GL account and crediting the sending bank's GL account.
- **`POST /api/v1/finance/bank-reconciliations/import`**
  - **Description:** Upload a bank statement file (CSV/OFX) to populate unreconciled bank lines.
- **`POST /api/v1/finance/bank-reconciliations/{id}/match`**
  - **Description:** Link a bank statement line to a system transaction.
- **`POST /api/v1/finance/bank-reconciliations/{id}/complete`**
  - **Description:** Finalize the reconciliation. Validates that `Cleared Balance == Statement Ending Balance`.

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
  - **Shared Hub:** The `finance_bank_accounts` entity acts as a hub for all liquidity-related transactions (Transfers, Reconciliations).
  - **Isolated Spokes:** Transactional tables like `finance_ap_payments` and `finance_ar_payments` reference these bank accounts via `account_id`.
- **Scoping**: Include `subsidiary_id` (UUID/String) on every table for organizational isolation.
- **Precision**: Enforce `Numeric(20,4)` for all financial/money columns per the "Money Rule".
- **File Storage**: If the feature requires file attachments (e.g., bank statement PDFs), it MUST use the Centralized Storage Service (no custom blob columns).
- **Schema**: Finance Module
- **Tables:** `finance_bank_accounts`, `finance_bank_reconciliations`, `finance_bank_statement_lines`
- **Columns (`finance_bank_accounts`):**
  - `name` (VARCHAR)
  - `account_number` (VARCHAR, Encrypted/Masked if necessary)
  - `gl_account_id` (UUID, ForeignKey to `finance_accounts`)
  - `subsidiary_id` (UUID)
- **Columns (`finance_bank_reconciliations`):**
  - `bank_account_id` (UUID, ForeignKey)
  - `statement_date` (DATE)
  - `starting_balance` (Numeric(20,4))
  - `ending_balance` (Numeric(20,4))
  - `status` (VARCHAR)
- **GL Transaction Updates:** When a system transaction is matched, a boolean `is_cleared` flag on the `finance_journal_entry_lines` (or specific payment/receipt tables) is set to `True`, along with the `reconciliation_id`.

*(Note: Enforce `Numeric(20,4)` for all financial amounts to follow the "Money Rule").*

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `finance.bank.reconciliation.completed` (Triggers period-end close readiness checks)
  - `finance.bank.transfer.executed` (Triggers GL update)
- **Listens To:**
  - `finance.ap.payment.issued` (To add new transactions to the unreconciled system data pool).
  - `finance.ar.payment.received`

## 10. Business Rules & Validations
- **Reconciliation Lock:** Once a transaction is marked as `is_cleared` and the reconciliation is `COMPLETED`, that transaction (e.g., the AP Payment) is strictly locked. It cannot be voided, altered, or deleted because doing so would break the historical bank reconciliation.
- **Zero Difference Validation:** A reconciliation cannot be completed if `(Starting Balance + Matched Deposits - Matched Withdrawals) != Statement Ending Balance`.
- **Currency:** Bank transfers between accounts of different currencies must specify an exchange rate to calculate the exact GL impact.

## 11. Security, Audit, and RBAC
- **Roles:**
  - **Cash Manager / Finance Admin:** Can create bank accounts, execute transfers, and complete reconciliations.
  - **AR/AP Clerks:** Typically do not have access to Bank Reconciliations.
- **Read-Only Licensing Mode:** Forms disabled, "Complete" buttons hidden.
- **Audit Trail:** Strict logging required for bank account creation (to prevent fraudulent routing numbers) and for finalizing reconciliations.

## 12. Process Transparency & Workflow Pipeline
To eliminate the "blackbox" nature of background processes and show architectural traceability as a Solution Architect, document how this feature integrates into the global workflow:
- **Pending Pipeline (Home Dashboard):** 
  - "Unreconciled Bank Transactions" (Count of transactions older than X days).
  - "Pending Reconciliations" (Draft reconciliations that need to be finalized).
- **Process UI Integration:** 
  - **Macro View (`ProcessPipeline`):** Use the horizontal tracker anchored at the **top of the Drawer** (above the form) to show the high-level status (Draft → In Progress → Reconciled). This also serves as the action center for inline approvals.
  - **Micro View (`Timeline`):** Use the vertical activity feed placed inside a **secondary "History/Activity" tab** within the Drawer body. This prevents the dense 5Ws audit data (Who, What, When, Why) from cluttering the editable form details.
- **Traceability:** While Reconciliation is a terminal process, the right-panel timeline for a matched payment can explicitly state: `Authorized by User X -> Paid on Date Y -> Cleared Bank on Date Z (Reconciliation #1234)`.
- **Shell UI & Navigation:** Sidebar -> Finance -> Bank & Cash.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation**: Create DB migrations for `finance_bank_accounts` and `finance_bank_reconciliations` extending `BESBase`.
- **Phase 2: Core Logic & APIs**: Build the Auto-Matching Engine to heuristically pair bank lines with system lines.
- **Phase 3: Frontend Infrastructure**: Register under the `finance` NX library.
- **Phase 4: UI Development**: Build the Bank Dashboard and the complex Split-Pane Reconciliation Workspace using `@bes/shared-ui`.
- **Phase 5: Integration**: Ensure that completing a reconciliation correctly sets the `is_cleared` flags on the underlying AP/AR/GL transactions.

## 14. Verification & QA Strategy
- **Scoping Check**: Ensure Subsidiary A cannot see or transfer funds to Subsidiary B's bank accounts (unless specifically flagged as an Intercompany transfer).
- **Precision Check**: Test the reconciliation "Difference" calculation using fractional cents to ensure 4-decimal precision holds without rounding errors preventing completion.
- **Functional Scenarios**:
  1. Complete a perfect reconciliation where system transactions exactly match the bank statement.
  2. Attempt to complete a reconciliation with a $0.01 difference (verify validation error).
  3. Attempt to void an AP Payment that has already been `cleared` in a completed reconciliation (verify strict rejection).
- **Integration Test**: Verify the Process Chain accurately shows the "Cleared" status on the source AP/AR documents.
