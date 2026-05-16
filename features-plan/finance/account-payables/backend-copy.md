# Backend Implementation Plan: Accounts Payable (AP)

This document outlines the backend implementation for the Accounts Payable (AP) feature within the Finance module, adhering to the BES Factory Architecture Rules.

## 1. Module Overview
- **Extension Name:** `finance`
- **Goal:** Manage vendor obligations, invoice approvals, and payment execution with automated General Ledger integration and strict multi-org isolation.

## 2. API Design & Payloads
All endpoints are prefixed with `/api/v1/finance/ap`.

### GET `/api/v1/finance/ap/invoices`
- **Description:** List AP invoices with filtering and pagination.
- **Input (Params):** 
  - `status`: Optional filter (DRAFT, PENDING_APPROVAL, etc.)
  - `vendor_id`: Optional filter
  - `due_date_from` / `due_date_to`: Optional date range
  - `PaginationParams`: Required for list view.
- **Desired Output:** `paginated_response` containing an array of `APInvoiceRead` objects.

### POST `/api/v1/finance/ap/invoices`
- **Description:** Create a new AP invoice in DRAFT status.
- **Input (Schema):** `APInvoiceCreate`
  ```python
  class APInvoiceCreate(BaseModel):
      vendor_id: uuid.UUID
      vendor_invoice_number: str
      invoice_date: datetime.date
      due_date: datetime.date
      total_amount: Decimal
      lines: List[APInvoiceLineCreate]
  ```
- **Desired Output:** `success_response` with the created `APInvoiceRead`.

### POST `/api/v1/finance/ap/invoices/{id}/approve`
- **Description:** Transition invoice to APPROVED status and trigger GL entry generation.
- **Desired Output:** `success_response` with the updated status and linked `journal_entry_id`.

### POST `/api/v1/finance/ap/payments`
- **Description:** Record a payment against one or more approved invoices.
- **Input (Schema):** `APPaymentCreate`
  ```python
  class APPaymentCreate(BaseModel):
      payment_date: datetime.date
      bank_account_id: uuid.UUID # Funding source from COA
      payment_method: str
      invoices: List[APPaymentAllocationCreate] # Mapping IDs and amounts
  ```
- **Desired Output:** `success_response` with the created `APPaymentRead`.

## 3. Business Logic (Services Layer)
Implemented in `finance/services.py`:

- **The Money Rule:** 
    - ALL amounts (`total_amount`, `amount` in lines, `payment_amount`) MUST use `Decimal` with `Numeric(20,4)` precision.
- **Validation:**
    - **Sum Check:** Total of line amounts must match the header `total_amount`.
    - **Vendor Validation:** Vendor must exist in the `core.vendors` hub and be active.
    - **Status Flow:** Only `DRAFT` invoices can be edited. Only `APPROVED` invoices can be paid.
- **GL Automation:**
    - On approval: Generate a Journal Entry:
        - **Debit:** Line item `expense_account_id`.
        - **Credit:** Subsidiary-specific `Accounts Payable` liability account.
    - On payment: Generate a Journal Entry:
        - **Debit:** `Accounts Payable` liability account.
        - **Credit:** `bank_account_id` (Cash/Bank).
- **Transactions:** 
    - Approval and Payment operations must be wrapped in `AsyncSession` transactions to ensure GL entries and status updates are atomic.

## 4. RBAC & Security Considerations
- **Required Permissions:**
    - `GET /invoices`: `"finance:ap:read"`
    - `POST /invoices`: `"finance:ap:write"`
    - `POST /approve`: `"finance:ap:approve"`
    - `POST /payments`: `"finance:ap:pay"`
- **Subsidiary Scoping:**
    - All AP tables (`finance_ap_invoices`, `finance_ap_invoice_lines`, `finance_ap_payments`) MUST inherit from `BESBase`.
    - Queries will automatically filter by `subsidiary_id` from the context.

## 5. Audit Trail & Soft Deletion
- **Soft Deletion:** Physical deletion is strictly forbidden. Use `is_deleted = True`.
- **Audit Logging:**
    - Status transitions (Draft -> Approved -> Paid) must be logged with the performing `user_id`.
    - `BESBase` handles `created_at`, `updated_at`, and `created_by`.

## 6. Events & Pub/Sub
- **Events Emitted:**
    - `FINANCE_AP_INVOICE_CREATED`: Initial bill capture.
    - `FINANCE_AP_INVOICE_APPROVED`: Trigger for GL posting.
    - `FINANCE_AP_PAYMENT_ISSUED`: Trigger for liability reduction.
- **Event Subscriptions:**
    - `CORE_VENDOR_DELETED`: To potentially flag or block invoices associated with deleted vendors (though soft-delete should handle this).

## 7. Implementation Roadmap
1. **Directory Structure:** Ensure `extensions/finance/` exists.
2. **Models (`models.py`):**
    - `APInvoice`, `APInvoiceLine`, `APPayment`, `APPaymentAllocation`.
3. **Schemas (`schemas.py`):**
    - `APInvoiceCreate`, `APInvoiceRead`, `APPaymentCreate`, etc.
4. **Service Layer (`services.py`):**
    - Implement `APInvoiceService` for bill management and GL integration.
    - Implement `APPaymentService` for payment execution and allocations.
5. **Router (`router.py`):**
    - Define AP-specific sub-router and wire to services.
    - Apply `require_permission` for each endpoint.
6. **Events (`events.py`):** Implement emitters for the AP lifecycle.
7. **Manifest (`manifest.py`):** Register the AP sub-router.
8. **Permissions:** Ensure `finance:ap:*` permissions are in `admin_permissions.json`.
9. **Migration:** Create Alembic migrations for the new AP tables.
