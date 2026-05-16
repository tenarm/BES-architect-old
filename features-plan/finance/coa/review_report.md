# Feature Plan Review: Chart of Accounts (COA)

## 1. Executive Summary
**Status**: **APPROVED**
The feature plan for the Chart of Accounts (COA) is well-architected, follows all BES Factory rules, and provides a clear roadmap for both frontend and backend development. It correctly addresses multi-tenancy, soft deletion, and data integrity (hierarchy limits).

---

## 2. Detailed Findings

### ✅ Architectural Strengths
- **Multi-Tenancy**: Explicitly incorporates `subsidiary_id` and mandatory `X-Subsidiary-Id` header context.
- **Hierarchy Integrity**: Includes validations for circular references and a 5-level depth limit.
- **Soft Delete**: Strictly follows the "Physical deletion is FORBIDDEN" rule.
- **Money Rule**: While COA is structural, the plan acknowledges the `Numeric(20,4)` requirement for balances and related transactional entities.
- **Layering**: Correct separation of Models, Schemas, Services, and Router.

### ❌ Critical Issues (Violations)
- *None.*

### ⚠️ Compatibility Gaps
- **Event Naming**: `frontend.md` refers to events using dot notation (e.g., `finance.account.created`), while `backend.md` correctly follows the `UPPER_SNAKE_CASE` rule (e.g., `FINANCE_ACCOUNT_CREATED`).
- **Field Naming**: The existing `Account` model in `finance/models.py` uses shorter field names (`code`, `type`, `parent_id`). The plan proposes more explicit names (`account_code`, `account_type`, `parent_account_id`).

### 💡 Recommendations
- **Standardize Event Names**: Ensure both frontend and backend use `FINANCE_ACCOUNT_CREATED`, etc.
- **Explicit Field Names**: Proceed with the plan's explicit field names (`account_code`, etc.) to improve long-term maintainability, even if it requires a database migration for the existing `finance_accounts` table.

---

## 3. Final Verdict
- **Status**: **APPROVED**
- **Next Steps**:
    1. Generate `changes.md` in `features-plan/finance/coa/`.
    2. Implement backend migrations and service logic.
    3. Register the new UI module and build the COA Manager.
