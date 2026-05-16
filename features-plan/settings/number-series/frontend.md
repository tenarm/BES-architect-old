## 1. Module
`settings`

## 2. Name
Number Series / Sequence Management

## 3. Description
The Number Series management feature provides a centralized engine for generating unique, sequential document identifiers across the BES (e.g., Invoice Numbers, Purchase Order IDs, Journal Vouchers). It allows administrators to define patterns (prefixes, suffixes, padding) and reset logic (monthly, yearly, or never) for different document types, ensuring compliance with local accounting standards and organizational naming conventions.

## 4. Depends On
- **Core Kernel:**
  - `BESBase`: For audit fields.
  - `Sequence Engine`: A specialized core service to handle atomic increments and pattern formatting.
- **Cross-Module:**
  - Used by `Finance`, `Sales`, `Inventory`, `Purchase` to fetch the "Next Number" during document creation.
- **UI Libraries:**
  - `@bes/shared-ui`: `Table`, `Drawer`, `Form`, `Skeleton`.

## 5. Feature UI — The UI as Information
**Feature Name:** Sequence Configurator
**UI Details:**
- **Series Table:** Columns for `code` (e.g., INV), `description`, `current_value`, `preview` (e.g., INV-2023-0001).
- **Configuration Drawer:**
  - **Pattern Tab:**
    - `Prefix`: Static text or dynamic tokens (e.g., `{YYYY}`, `{MM}`).
    - `Suffix`: Static text.
    - `Padding`: Number of digits (e.g., 4 for `0001`).
    - `Starting Number`: Usually 1.
  - **Reset Tab:** Reset frequency (Never, Daily, Monthly, Yearly).
- **User Journey:**
  1. Admin goes to **Settings > Number Series**.
  2. Selects "Sales Invoice".
  3. Updates prefix to `SINV/{YYYY}/`.
  4. Saves. Next invoice generated will follow this new pattern.

## 6. Sample Data Structure (YAML + JSON)
```yaml
# Number Series Definition
id: "seq-uuid-789"
code: "SALES_INVOICE"
description: "Primary Sales Invoices"
prefix: "INV/{YYYY}/"
suffix: "-BES"
padding: 5
current_value: 124
last_reset_date: "2023-01-01"
reset_frequency: "Yearly"
```

```json
{
  "status": "success",
  "data": {
    "next_number": "INV/2023/00125-BES"
  }
}
```

## 7. Required APIs
- `GET /api/v1/settings/number-series`: List all series.
- `PATCH /api/v1/settings/number-series/{id}`: Update pattern or reset logic.
- `POST /api/v1/settings/number-series/preview`: Test a pattern string to see how it renders.
- **Internal API:** `POST /api/v1/settings/number-series/next/{code}`: Atomic increment and return formatted string.

## 8. Database Tables & Architecture
- **Table:** `core.number_series`
  - `code`: `String` (Unique key, e.g., 'FIN_JE').
  - `prefix`: `String`.
  - `suffix`: `String`.
  - `padding`: `Integer`.
  - `current_value`: `BigInt`.
  - `reset_frequency`: `Enum` (NONE, DAILY, MONTHLY, YEARLY).
  - `last_reset_date`: `DateTime`.

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `SETTINGS_SERIES_UPDATED`
- **Listens To:**
  - None.

## 10. Business Rules & Validations
- **Atomicity:** Generating the next number MUST be thread-safe/transaction-safe to prevent duplicate IDs under high load (Postgres `UPDATE ... RETURNING` or sequences).
- **Immutability:** Once a number is assigned to a posted document, it cannot be changed.
- **Gaps:** In some jurisdictions, gaps in numbering are forbidden; the system must ensure sequential integrity.

## 11. Security, Audit, and RBAC
- **Permission:** `settings:number_series:write`.
- **Audit Trail:** Log any manual override of `current_value` (highly sensitive action).
- **Access Control:** Only system-level administrators should modify sequence patterns.

## 12. Process Transparency & Workflow Pipeline
- **Right Panel (Drawer) UX:**
  - **Macro View (`ProcessPipeline`):** Not applicable (static config).
  - **Micro View (`Timeline`):** History of pattern changes.
- **Shell Navigation:** Sidebar: **Settings > Configuration > Number Series**.

## 13. Technical Implementation Roadmap (Day 1)
- **Phase 1:** Create `number_series` table in `core`.
- **Phase 2:** Implement the atomic `get_next_number` logic in the backend.
- **Phase 3:** Create the Series Table and Configuration Drawer.
- **Phase 4:** Implement pattern token parsing (e.g., replacing `{YYYY}` with current year).
- **Phase 5:** Integrate with Sales/Finance modules to consume the service.

## 14. Verification & QA Strategy
- **Token Check:** Verify `{YYYY}` correctly renders the current year.
- **Concurrency Test:** Simulate 10 simultaneous requests for a number and verify no duplicates.
- **Reset Test:** Mock system date to next year and verify sequence starts back at 1.
- **Padding Test:** Verify padding of 3 turns `5` into `005`.
