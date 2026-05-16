---
name: project-readiness-auditor
description: This skill instructs the AI assistant to perform a project-wide audit of all feature plans across all modules to assess overall completeness and production readiness.
---

# Skill: Project Readiness Auditor

This skill instructs the AI assistant to act as a CTO / Lead Architect and evaluate the entire `features-plan/` directory across all BES modules. The goal is to produce an honest, data-driven production-readiness assessment of the full system plan.

---

## About the BES System (Audit Context)

The BES Factory is a **Kernel-and-Plugin** multi-tenant SaaS platform:

- **Core Kernel** (`bes-backend/core/`): Auth (JWT), DB Pooling, RBAC, Event Bus, `BESBase`, `StandardResponse`, Pagination, File Storage, UOM Registry.
- **Extensions** (`bes-backend/extensions/`): Domain modules — Finance, Sales, Inventory, HR, Supply Chain, CRM, Settings.
- **Shell UI** (`bes-frontend/apps/shell/`): Dynamically loads licensed modules via `ComponentRegistry`. Bootstrap endpoint controls active/readonly state.
- **Deployment**: DB-per-tenant, `ACTIVE_EXTENSIONS` / `READONLY_EXTENSIONS` env vars, PM2 process manager (Day 1), Persistent Event Outbox / Redis (Day 2).

**Known Modules**: `core`, `finance`, `sales`, `inventory`, `hr`, `supply-chain`, `crm`, `settings`.

**The 10 Golden Rules (must be enforced project-wide):**
1. All tables inherit `BESBase` (UUID PK, audit fields, `subsidiary_id`, `metadata_` JSONB).
2. Money Rule: `Numeric(20,4)` backend, `Decimal` Python, `decimal.js` frontend.
3. Soft Deletes only — `is_deleted = True`.
4. Strict layering: models → schemas → services → router → events.
5. `StandardResponse` envelope on all API responses.
6. Pagination (`PaginationParams`) on all list endpoints.
7. RBAC via `require_permission()` — format `<module>:<resource>:<action>`.
8. Hub-and-Spoke MDM — master data in `core` schema, module tables as transactional spokes.
9. Event Bus only for cross-module communication (no direct extension imports).
10. UI graceful degradation for `READONLY_EXTENSIONS` modules.

---

## Instructions for the Assistant

When the user asks to "project readiness audit" or "assess system-wide completeness":

### 1. Preparation
- Scan the entire `features-plan/` directory recursively.
- For each module, list all feature folders and check for `frontend.md` and `backend.md`.
- Build a completeness matrix across all modules.

### 2. Audit Dimensions

#### A. Module Coverage
Are all 8 expected BES modules represented in `features-plan/`?
- `core`, `finance`, `sales`, `inventory`, `hr`, `supply-chain`, `crm`, `settings`
- Flag any module entirely absent from the plan.

#### B. Feature Completeness Per Module
For each module, identify:
- Features with both `frontend.md` + `backend.md` → **Fully Planned**
- Features with only `frontend.md` → **Backend Missing**
- Features with only `backend.md` → **Frontend Missing**
- Feature folders with no docs → **Empty / Placeholder**

#### C. Industry-Standard Gap Analysis
Based on each module's existing features, identify obviously missing features:
- **Finance**: Should have COA, GL, AR, AP, Bank & Cash, Tax Management, Purchase Invoices, Period Close.
- **Sales**: Should have Customer Master, Quotation, Sales Order, Sales Invoice, Price List, Credit Limit.
- **Inventory**: Should have Item Master, UOM, Warehouse, Goods Receipt, Goods Issue, Stock Valuation.
- **HR**: Should have Employee Master, Leave Management, Payroll, Attendance.
- **Supply Chain**: Should have Supplier Master, Purchase Order, RFQ, Goods Receipt (PO-linked).
- **CRM**: Should have Lead, Opportunity, Contact, Activity Log.
- **Settings**: Should have Company Setup, User Management, Role Management, Audit Log, Module Config.

#### D. Golden Rules — Cross-Module Compliance Scan
Scan all available backend docs for consistent application of:
- `BESBase` inheritance.
- Money Rule (`Numeric(20,4)`, `Decimal`).
- Soft Delete plan.
- RBAC coverage.
- `subsidiary_id` scoping.
- Hub-and-Spoke MDM (no rogue master data in module schemas).

#### E. Cross-Module Integration Health
- Do modules that interact have matching event plans? (e.g., Sales emitting `SALES_ORDER_CONFIRMED` must match Finance subscribing to it.)
- Are there planned cross-module dependency chains that could cause circular imports (forbidden)?
- Is the global UOM registry from `core` referenced consistently?

#### F. Production Readiness Indicators
- **RBAC**: Is every module's permission namespace planned in `admin_permissions.json`?
- **Audit Trail**: Are critical business actions (Invoice Posting, Approval, Stock Movement) flagged for audit logging?
- **Licensing**: Are all modules designed for independent `ACTIVE_EXTENSIONS` toggle?
- **Verification**: Does each feature have a "Verification & QA Strategy" section?
- **Migrations**: Is there a migration/self-healing plan (`SQLModel.metadata.create_all()` or Alembic) for all features?

### 3. Output

Write the report to: `features-plan/project-readiness-report.md`

---

## Project Readiness Report Structure

## 1. Project Maturity Score
Overall percentage completion and readiness tier:
- `🔴 Early Planning` (< 40%) — Major gaps
- `🟡 Planning Phase` (40–70%) — Core features planned, gaps remain
- `🟢 Implementation Ready` (> 70%) — Well-structured, minor gaps only

## 2. Module Status Dashboard

| Module | Features Planned | Fully Documented | Backend Missing | Frontend Missing | Industry Gaps |
|:-------|:-----------------|:-----------------|:----------------|:-----------------|:--------------|
| finance | 7 | 5 | 1 | 1 | Period Close |
| ... | ... | ... | ... | ... | ... |

## 3. High-Level Findings

### 🛑 Critical Planning Gaps
Blocking gaps that must be addressed before any implementation begins.

### ⚠️ Integration Risks
Cross-module event mismatches, missing dependency plans, or circular risk.

### ✅ Architectural Strengths
Modules or features that consistently and correctly apply all Golden Rules.

## 4. Priority Implementation Order
A project-wide dependency-sorted build order:
`Core Kernel → Settings → Shared Master Data (core.customers, core.vendors, core.products, core.uoms) → Finance Base → Inventory Base → HR Base → Sales (depends on Finance + Inventory) → Supply Chain (depends on Inventory) → CRM (depends on Sales) → Cross-Module Event Wiring → SSE Frontend Integration`

## 5. Architect's Action Items
Specific, numbered tasks for the team to bridge gaps before committing to implementation.
