---
name: module-architect-reviewer
description: This skill instructs the AI assistant to perform a module-wide architectural review of all features, identifying gaps, redundancies, and cross-feature dependencies.
---

# Skill: Module Architect Reviewer

This skill instructs the AI assistant to act as a BES Solutions Architect and audit an entire module (e.g., all features under `features-plan/finance/`). The goal is to find shared opportunities, data model conflicts, cross-feature dependencies, and create a phased build roadmap.

---

## About the BES Architecture (Reviewer Context)

- **Kernel-and-Plugin**: Core Kernel (`bes-backend/core/`) is read-only for extensions. Domain logic lives in `bes-backend/extensions/<module>/`.
- **Hub-and-Spoke MDM**: Global master data lives in the `core` schema. Module tables are transactional spokes that FK to core.
- **Event Bus**: The ONLY cross-module communication channel. Modules MUST NOT directly import each other.
- **Licensing**: Each module is independently toggled via `ACTIVE_EXTENSIONS` / `READONLY_EXTENSIONS`. Plan features to be independently deployable.
- **Nx Frontend**: Each module has an Nx library (`@bes/<module>`) registered in the Shell via `ComponentRegistry.registerLazy()`.

**Known BES Modules**: `core`, `finance`, `sales`, `inventory`, `hr`, `supply-chain`, `crm`, `settings`.

---

## Instructions for the Assistant

When the user asks to "architect review the <module> module" or "audit all features in <module>," follow these steps:

### 1. Preparation
- List all feature subdirectories in `features-plan/<module>/`.
- For each feature, read both `frontend.md` and `backend.md` (flag missing docs).
- Read `.agent/rules/architecture.md`, `.agent/rules/backend.md`, `.agent/rules/security.md`.

### 2. Module-Wide Analysis

#### A. Data Model Audit
- **Table Conflicts**: Do any two features define tables with overlapping names or columns?
- **Redundant Entities**: Are similar entities defined independently (e.g., `finance_tax_codes` and `finance_tax_rates` both storing tax data)?
- **Core References**: Are all features correctly reading master data from `core` (customers, vendors, products, uoms) instead of defining their own?
- **`metadata_` Usage**: Are Ghost Foreign Keys (cross-module soft links via `metadata_` JSONB) used appropriately?

#### B. Service & Logic Reuse
- **Shared Utilities**: Identify logic that should be a shared `utils.py` within the module (e.g., tax calculation used by both Invoices and POs).
- **Base Services**: Is there repetitive CRUD logic that can use `core.BaseRepository`?
- **Transaction Chains**: Multi-feature operations (e.g., Sales Invoice → Finance GL Posting) must be coordinated via the Event Bus, not direct calls.

#### C. API Cohesion
- **Consistent Routing**: All endpoints under the same module must use the same prefix `/api/v1/<module>`.
- **Naming Conventions**: Resource paths should follow a consistent pattern (e.g., `/finance/accounts`, `/finance/invoices` — not `/finance/chart-of-accounts`).
- **RBAC Namespace**: All permissions should use the same module namespace (e.g., `finance:*:*`).

#### D. Event Flow Mapping
- Build a module-internal event catalog listing all `UPPER_SNAKE_CASE` events emitted and subscribed.
- Identify any circular event loops (Feature A emits → Feature B subscribes → Feature B emits → Feature A subscribes).
- Identify events expected from external modules (e.g., Finance listening for `SALES_ORDER_CONFIRMED` from Sales).

#### E. Feature Dependency Graph
- Identify which features must be built before others (e.g., Chart of Accounts before General Ledger; Vendors before Purchase Invoices).
- Flag features with no dependencies (can be built in parallel).

#### F. Licensing Independence
- Can each feature be loaded/unloaded independently via `ACTIVE_EXTENSIONS`?
- Do features have hard imports between each other (forbidden) vs. Event Bus communication (correct)?

### 3. Output

Write the analysis to: `features-plan/<module>/module-cross-features-changes.md`

---

## Module Cross-Features Report Structure

## 1. Module Overview
High-level summary of the module's state, completeness, and architectural maturity.

## 2. Feature Inventory

| Feature | Frontend Doc | Backend Doc | Status |
|:--------|:-------------|:------------|:-------|
| `coa`   | ✅ Present  | ✅ Present  | Ready  |
| `gl`    | ✅ Present  | ❌ Missing  | Incomplete |

## 3. Cross-Feature Findings

### 🔄 Shared Components & Services
List reusable logic, schemas, or UI components to build ONCE for the whole module.

### ⚠️ Data Model Conflicts & Redundancies
Table naming collisions, duplicate entities, or incorrect master data references.

### 🔗 Feature Dependency Graph
Ordered list: `Feature A → Feature B → Feature C` (must be built in this order).

### ⚡ Internal Event Catalog
Full list of events emitted and subscribed within this module, and events expected from external modules.

### 🛑 Architectural Violations
Any feature-level issues that violate BES Golden Rules.

## 4. Phased Module Implementation Roadmap

- **Phase 1: Shared Foundation** — Shared models, base services, `admin_permissions.json` namespace setup.
- **Phase 2: Master Data Features** — Independent features with no module dependencies.
- **Phase 3: Transactional Features** — Features depending on Master Data or other modules.
- **Phase 4: Integration & Events** — Cross-module event wiring and SSE validation.

## 5. Module Verification Strategy
How to test this module end-to-end as an integrated system (API smoke tests, event flow, subsidiary isolation, licensing toggle).
