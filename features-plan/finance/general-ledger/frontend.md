# Feature Documentation: General Ledger (GL)

## 1. Module
Finance

## 2. Name
General Ledger

## 3. Description
The General Ledger (GL) is the central repository for accounting data within the BES. It serves as the master record of all financial transactions, utilizing double-entry bookkeeping to ensure the accounting equation (Assets = Liabilities + Equity) always balances. The GL consolidates financial activity from all other modules (Accounts Payable, Accounts Receivable, Inventory) and allows finance users to create manual Journal Entries (JEs) for adjustments, accruals, and period-end closing.

## 4. Depends on
- **Core Integration:** Core provides the unified `BESBase` PostgreSQL foundation, allowing Finance and other modules to share the same physical database while maintaining logically isolated schemas like `finance_journal_entries`.
- **Other Dependencies:** 
  - `@bes/shared-ui` (Data tables, Forms, Process Chain UI)
  - `Finance Module (Chart of Accounts)` (Provides the accounts for GL lines)
  - `Core Module` (Authentication, RBAC, Subsidiary context, Fiscal Periods)
  - `BES Audit System` (for tracking changes to JEs)

## 5. Feature name, details - the UI as information
**Feature Name:** Journal Entries Manager
**Details & UI Information:**
- **Page Layout:** Master-Detail or List view with a separate full-page form for entry creation.
- **Data Table View:** "All Journal Entries" list showing `Entry #`, `Date`, `Description`, `Total Debit`, `Total Credit`, `Status` (Draft, Posted), and `Source` (Manual, AP Invoice, etc.). Includes robust filtering (Date Range, Status, Account).
- **Journal Entry Form (Header):** 
  - `Entry Date` (Date picker)
  - `Posting Period` (Dropdown based on active fiscal periods)
  - `Memo/Description` (Textarea)
  - `Currency` (Dropdown, defaulting to subsidiary base currency)
- **Journal Entry Form (Lines Grid):**
  - Editable data grid for adding debits/credits.
  - Columns: `Account` (Searchable dropdown from COA), `Description`, `Debit` (Numeric input), `Credit` (Numeric input), `Entity` (Optional, e.g., Vendor or Customer), `Department/Class` (Optional classifications).
  - Footer showing `Total Debits`, `Total Credits`, and `Out of Balance` amount (must be 0.0000 to post).
- **Empty States:** "No journal entries found for this period."
- **Loading States:** Skeleton grid rows and disabled action buttons during save/post.

**User Journey & UX Flow:**
User navigates to Finance > General Ledger > Journal Entries -> Clicks "New Journal Entry" -> Fills header info (Date, Memo) -> Adds lines, selecting Accounts and entering Debits/Credits -> Ensures Debits equal Credits -> Saves as "Draft" -> Reviews entry -> Clicks "Post" -> System locks entry, updates account balances, and displays success toast.

## 6. YAML or sample data structure

```yaml
JournalEntry:
  type: object
  properties:
    id:
      type: string
      format: uuid
    entry_number:
      type: string
    transaction_date:
      type: string
      format: date
    memo:
      type: string
    status:
      type: string
      enum: [DRAFT, POSTED, REVERSED]
    subsidiary_id:
      type: string
      format: uuid
    lines:
      type: array
      items:
        $ref: '#/components/schemas/JournalEntryLine'

JournalEntryLine:
  type: object
  properties:
    id:
      type: string
      format: uuid
    journal_entry_id:
      type: string
      format: uuid
    account_id:
      type: string
      format: uuid
    debit:
      type: string
      description: "String-based decimal, 4 places"
    credit:
      type: string
      description: "String-based decimal, 4 places"
    memo:
      type: string
```

**Sample JSON Payload:**
```json
{
  "id": "e234bc56-7890-1234-5678-90abcdef1234",
  "entry_number": "JE-2026-0014",
  "transaction_date": "2026-05-15",
  "memo": "Monthly rent accrual",
  "status": "DRAFT",
  "subsidiary_id": "a0000000-0000-0000-0000-000000000001",
  "lines": [
    {
      "id": "line-1",
      "account_id": "account-rent-expense-uuid",
      "debit": "5000.0000",
      "credit": "0.0000",
      "memo": "Rent Expense"
    },
    {
      "id": "line-2",
      "account_id": "account-accrued-rent-uuid",
      "debit": "0.0000",
      "credit": "5000.0000",
      "memo": "Accrued Liability"
    }
  ]
}
```

## 7. Required APIs

- **`GET /api/v1/finance/journal-entries`**
  - **Description:** Fetch paginated list of journal entries.
- **`GET /api/v1/finance/journal-entries/{id}`**
  - **Description:** Get specific JE with all its lines.
- **`POST /api/v1/finance/journal-entries`**
  - **Description:** Create a new JE (Draft state).
  - **Validation:** Total Debits must equal Total Credits before allowing state change to POSTED.
- **`PUT /api/v1/finance/journal-entries/{id}`**
  - **Description:** Update a Draft JE.
- **`POST /api/v1/finance/journal-entries/{id}/post`**
  - **Description:** Post the entry to the ledger.
  - **Validation:** Debits = Credits. Period must be Open.
- **`DELETE /api/v1/finance/journal-entries/{id}`**
  - **Description:** Soft-delete a Draft JE. (Posted JEs cannot be deleted, only Reversed).

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
  - **Shared Hub:** The `finance_accounts` (COA) entity acts as the primary hub for all ledger activity.
  - **Isolated Spokes:** Transactional General Ledger tables (`finance_journal_entries`, `finance_journal_entry_lines`) act as spokes that reference the account hub.
- **Scoping**: Include `subsidiary_id` (UUID/String) on every table for organizational isolation.
- **Precision**: Enforce `Numeric(20,4)` for all financial/money columns per the "Money Rule".
- **File Storage**: If the feature requires file attachments, it MUST use the Centralized Storage Service (no custom blob columns).
- **Schema**: Finance Module
- **Tables:** `finance_journal_entries`, `finance_journal_entry_lines`
- **Columns (`finance_journal_entries`):**
  - `entry_number` (VARCHAR, Auto-generated sequence)
  - `transaction_date` (DATE, Not Null)
  - `memo` (TEXT)
  - `status` (VARCHAR, Default 'DRAFT')
  - `source_transaction_id` (UUID, Nullable, polymorphic reference to AP Invoice, etc.)
  - `subsidiary_id` (UUID, Not Null)
- **Columns (`finance_journal_entry_lines`):**
  - `journal_entry_id` (UUID, ForeignKey, Not Null)
  - `account_id` (UUID, ForeignKey to `finance_accounts`, Not Null)
  - `debit` (Numeric(20,4), Default 0.0000)
  - `credit` (Numeric(20,4), Default 0.0000)
  - `memo` (TEXT)
  - `subsidiary_id` (UUID, Not Null)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `finance.gl.entry.created` (Payload: JE ID)
  - `finance.gl.entry.posted` (Payload: JE ID, Account impacts). *Critical event: triggers materialization of account balances.*
- **Listens To:**
  - Events from sub-ledgers (e.g., `finance.ap.invoice.approved`) to automatically generate system JEs.

## 10. Business Rules & Validations
- **Double-Entry Validation:** A Journal Entry CANNOT be posted unless `Sum(Debits) == Sum(Credits)`.
- **Decimal Precision:** Strict adherence to the "Money Rule" (4 decimal places for `debit` and `credit` columns).
- **Immutability:** Once `status` is 'POSTED', the entry and its lines are locked and CANNOT be updated or deleted. To correct an error, a new Reversal JE must be created.
- **Period Locking:** JEs cannot be posted into closed fiscal periods.
- **Soft Deletes:** Physical deletion is FORBIDDEN. Only Draft JEs can be marked `is_deleted = true`.

## 11. Security, Audit, and RBAC
- **Roles:**
  - **Finance Admin:** Full CRUD access, can Post JEs.
  - **Finance Staff:** Can create Draft JEs, but may require approval to Post.
  - **Viewer:** Read-only access to view JEs.
- **Read-Only Licensing Mode:** "New Journal Entry" and "Post" buttons hidden; grids disabled.
- **Audit Trail:** Every status change (Draft -> Posted), creation, or edit of lines logs an event to the BES audit system. Tracking `previous_state` and `new_state` is critical for JEs.

## 12. Process Transparency & Workflow Pipeline
To eliminate the "blackbox" nature of background processes and show architectural traceability as a Solution Architect, document how this feature integrates into the global workflow:
- **Pending Pipeline (Home Dashboard):** Draft JEs requiring review/approval appear in the Pending Pipeline widget on the Home Dashboard.
- **Process UI Integration:** 
  - **Macro View (`ProcessPipeline`):** Use the horizontal tracker anchored at the **top of the Drawer** (above the form) to show the high-level status (Draft → Pending → Posted). This also serves as the action center for inline approvals.
  - **Micro View (`Timeline`):** Use the vertical activity feed placed inside a **secondary "History/Activity" tab** within the Drawer body. This prevents the dense 5Ws audit data (Who, What, When, Why) from cluttering the editable form details.
- **Traceability:** If the JE originated from a source document (e.g., AP Invoice), the Process Chain UI visualizes the linkage: `Purchase Order -> AP Invoice -> Journal Entry`. Right-panel timeline on the JE screen shows who created it, when it was modified, and when it was posted.
- **Shell UI & Navigation:** Sidebar -> Finance -> General Ledger -> Journal Entries.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation**: Create `finance_journal_entries` and `finance_journal_entry_lines` DB migrations extending `BESBase`.
- **Phase 2: Core Logic & APIs**: Develop GL validation engine (Debits=Credits checker) and REST endpoints.
- **Phase 3: Frontend Infrastructure**: Register the GL feature under the `finance` library in the NX workspace.
- **Phase 4: UI Development**: Build the JE Form with the dynamic editable data grid for lines.
- **Phase 5: Event Integration**: Implement Pub/Sub for `finance.gl.entry.posted` to update ledger rollups.

## 14. Verification & QA Strategy
- **Scoping Check**: Ensure JEs are tightly scoped to `subsidiary_id` and lines only use COA accounts from that same subsidiary.
- **Precision Check**: Test boundary values (e.g., `0.3333` + `0.6667`) to verify 4-decimal precision holds without rounding errors.
- **Functional Scenarios**:
  1. Create a balanced Draft JE and Post it successfully.
  2. Attempt to Post an out-of-balance JE (verify validation error).
  3. Attempt to edit or delete a Posted JE (verify API and UI rejection).
  4. Attempt to Post to a closed fiscal period (verify rejection).
- **Integration Test**: Post an entry and verify via SSE that the process chain and account balances update immediately.
