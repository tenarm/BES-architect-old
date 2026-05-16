# Implementation Changes: Chart of Accounts (COA)

This document summarizes the technical changes required to implement the COA feature, consolidating the frontend and backend plans into a single execution roadmap.

## 1. Impacted Files (Backend)

### [MODIFY] `extensions/finance/finance/models.py`
- Update `Account` model to align with the plan:
  - Rename `code` -> `account_code`.
  - Rename `type` -> `account_type`.
  - Rename `parent_id` -> `parent_account_id`.
  - Add `description: Optional[str]`.
  - Add `is_active: bool = True`.

### [MODIFY] `extensions/finance/finance/schemas.py`
- Define `FinanceAccountCreate`, `FinanceAccountUpdate`, and `FinanceAccountRead`.
- Ensure `account_type` uses the `AccountType` enum.

### [MODIFY] `extensions/finance/finance/services.py`
- Implement `AccountService`:
  - `create_account`: Validates uniqueness of `account_code` per subsidiary and circular parent references.
  - `get_account_tree`: Recursive logic to build the hierarchy (limit 5 levels).
  - `soft_delete_account`: Validates no child accounts or GL transactions exist.

### [MODIFY] `extensions/finance/finance/router.py`
- Add endpoints:
  - `GET /accounts`: Supports `format=tree` or `list`.
  - `POST /accounts`: Create new.
  - `GET /accounts/{id}`: Detail.
  - `PUT /accounts/{id}`: Update.
  - `DELETE /accounts/{id}`: Soft-delete.
- Apply `@require_permission("finance:coa:read/write")`.

### [MODIFY] `extensions/finance/finance/events.py`
- Add emitters for `FINANCE_ACCOUNT_CREATED`, `FINANCE_ACCOUNT_UPDATED`, `FINANCE_ACCOUNT_DELETED`.

---

## 2. Impacted Files (Frontend)

### [NEW] `libs/finance/src/lib/features/coa/`
- `coa-manager.component.ts`: Main split-pane container.
- `coa-tree.component.ts`: Tree view using `@bes/shared-ui` tree component.
- `coa-form.component.ts`: Form for account details/creation.

### [MODIFY] `libs/finance/src/lib/finance.module.ts`
- Register COA routes: `finance/coa`.

### [MODIFY] `apps/shell/src/app/registry.ts` (or equivalent)
- Register Finance module if not already present.

---

## 3. Integration Checklist

### Database
- [ ] Create Alembic migration for renaming and adding columns to `finance_accounts`.
- [ ] Verify `subsidiary_id` is correctly indexed.

### API & Security
- [ ] Verify `X-Subsidiary-Id` is passed in all frontend requests.
- [ ] Test RBAC with `finance:coa:read` and `finance:coa:write`.

### Events
- [ ] Verify `FINANCE_ACCOUNT_CREATED` event payload.
- [ ] Test real-time tree refresh on the frontend via SSE/WebSockets (if implemented).

---

## 4. Step-by-Step Implementation Order

1. **Database**: Run migration to update `finance_accounts` table.
2. **Backend Schemas**: Define Pydantic models for COA.
3. **Backend Service**: Implement core hierarchy and validation logic.
4. **Backend Router**: Wire endpoints and permissions.
5. **Frontend Core**: Update `@bes/finance` library routes and registration.
6. **Frontend UI**: Build the COA Tree and Form components.
7. **Verification**: Run subsidiary isolation and hierarchy depth tests.
