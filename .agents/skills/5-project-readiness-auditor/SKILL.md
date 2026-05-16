---
name: project-readiness-auditor
description: "LIFECYCLE STEP 4b (Optional): Perform a project-wide audit across all modules and feature plans to generate the project-readiness-report.md."
---

# Skill: Project Readiness Auditor
**Lifecycle Position: STEP 4b of 6 — Project Audit (Optional but Recommended)**
**Reads from:** All `features-plan/` documents and all `module-cross-features-changes.md` files (outputs of Steps 1–4a).
**Feeds into:** `master-implementation-architect` (which reads `project-readiness-report.md`)

This skill provides the CTO-level view. It scans the entire `features-plan/` directory and produces a single `project-readiness-report.md` that the `master-implementation-architect` uses to determine the global build sequence.

Run this after running `4-module-architect-reviewer` on all modules.

---

## Lifecycle Context
```
STEP 1: frontend-generate-feature-doc  →  frontend.md
STEP 2: backend-generate-feature-doc   →  backend.md
STEP 3: feature-plan-reviewer          →  changes.md
STEP 4a: module-architect-reviewer     →  module-cross-features-changes.md
[YOU ARE HERE]
STEP 4b: project-readiness-auditor     →  project-readiness-report.md
STEP 4c: master-implementation-architect →  master-implementation-roadmap.md
STEP 5: feature-implementation-orchestrator → CODE
STEP 6: post-implementation-documenter →  implementation-manual.md
```

---

## About the BES System (Audit Context)
- **Architecture**: Kernel-and-Plugin (Core + Extensions) multi-tenant SaaS.
- **Core Kernel** (`bes-backend/core/`): Auth (JWT), DB Pooling, RBAC, Event Bus, `BESBase`, `StandardResponse`, Pagination, File Storage, UOM Registry.
- **Extensions** (`bes-backend/extensions/`): Finance, Sales, Inventory, HR, Supply Chain, CRM, Settings.
- **Shell UI** (`bes-frontend/apps/shell/`): Dynamically loads licensed modules. Bootstrap controls active/readonly state.
- **Deployment**: DB-per-tenant, `ACTIVE_EXTENSIONS` / `READONLY_EXTENSIONS` env vars, PM2 (Day 1), Redis/Outbox (Day 2).
- **Known Modules**: `core`, `finance`, `sales`, `inventory`, `hr`, `supply-chain`, `crm`, `settings`.

**The 10 Golden Rules** — Checked globally:
1. `BESBase` on all tables.
2. Money Rule: `Numeric(20,4)` + `Decimal` + `decimal.js`.
3. Soft Deletes only.
4. Strict layer separation.
5. `StandardResponse` envelope always.
6. Pagination on all list endpoints.
7. RBAC via `require_permission()`.
8. Hub-and-Spoke MDM.
9. Event Bus for cross-module communication only.
10. UI graceful degradation for `READONLY_EXTENSIONS`.

---

## Instructions for the Assistant

When the user asks to "project readiness audit":

1. Scan all of `features-plan/` recursively.
2. Read all `module-cross-features-changes.md` files.
3. For each feature, check for `frontend.md`, `backend.md`, `changes.md`.
4. Run all audit dimensions below.
5. Write output to `features-plan/project-readiness-report.md`.

---

## Audit Dimensions

### A. Module Coverage
Are all 8 expected modules present?
`core`, `finance`, `sales`, `inventory`, `hr`, `supply-chain`, `crm`, `settings`

### B. Feature Completeness
For each module: Fully Planned / Backend Missing / Frontend Missing / Empty Placeholder.

### C. Industry-Standard Gap Analysis
- **Finance**: COA, GL, AR, AP, Bank & Cash, Tax Management, Purchase Invoices, Period Close.
- **Sales**: Customer Master, Quotation, Sales Order, Sales Invoice, Price List, Credit Limit.
- **Inventory**: Item Master, UOM, Warehouse, Goods Receipt, Goods Issue, Stock Valuation.
- **HR**: Employee Master, Leave Management, Payroll, Attendance.
- **Supply Chain**: Supplier Master, Purchase Order, RFQ, Goods Receipt (PO-linked).
- **CRM**: Lead, Opportunity, Contact, Activity Log.
- **Settings**: Company Setup, User Management, Role Management, Audit Log, Module Config.

### D. Golden Rules Cross-Module Scan
Consistent enforcement of all 10 rules across all available backend docs.

### E. Cross-Module Integration Health
- Matching event plans between modules that interact.
- No circular import dependencies.
- Global UOM registry referenced consistently.

### F. Production Readiness Indicators
- RBAC namespaces planned in `admin_permissions.json`.
- Critical business actions flagged for audit logging.
- All modules designed for independent `ACTIVE_EXTENSIONS` toggle.
- Verification & QA sections present in all features.

---

## Output: `features-plan/project-readiness-report.md`

## 1. Project Maturity Score
`🔴 Early Planning` (< 40%) / `🟡 Planning Phase` (40–70%) / `🟢 Implementation Ready` (> 70%)

## 2. Module Status Dashboard

| Module | Features Planned | Fully Documented | Backend Missing | Industry Gaps |
|:-------|:-----------------|:-----------------|:----------------|:--------------|

## 3. High-Level Findings
- 🛑 Critical Planning Gaps
- ⚠️ Integration Risks
- ✅ Architectural Strengths

## 4. Recommended Global Build Order
`Core → Settings → Shared Master Data → Finance → Inventory → HR → Sales → Supply Chain → CRM → Cross-Module Events → SSE Frontend`

## 5. Architect's Action Items
Specific, numbered tasks to bridge gaps before implementation begins.

---

### Execution Rules
- **Output file**: `features-plan/project-readiness-report.md`
- **Next step**: Run `master-implementation-architect` which reads this report + all module reviews to produce the final `master-implementation-roadmap.md`.
