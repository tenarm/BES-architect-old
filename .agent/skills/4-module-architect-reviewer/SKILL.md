---
name: module-architect-reviewer
description: "LIFECYCLE STEP 4a (Optional): Perform a module-wide architectural review across all features, find shared logic and cross-feature dependencies, and generate module-cross-features-changes.md."
---

# Skill: Module Architect Reviewer
**Lifecycle Position: STEP 4a of 6 — Module Audit (Optional but Recommended)**
**Reads from:** All `frontend.md`, `backend.md`, and `changes.md` files within a module (outputs of Steps 1–3).
**Feeds into:** `master-implementation-architect` (Step 4b)

This skill zooms out from a single feature and audits the entire module (e.g., all of `features-plan/finance/`). It finds shared logic, data model conflicts, event dependencies, and correct feature build ordering, then produces the `module-cross-features-changes.md` consumed by the Master Architect.

Run this once per module before triggering the Master Implementation Architect.

---

## Lifecycle Context
```
STEP 1: frontend-generate-feature-doc  →  frontend.md
STEP 2: backend-generate-feature-doc   →  backend.md
STEP 3: feature-plan-reviewer          →  changes.md
[YOU ARE HERE]
STEP 4a: module-architect-reviewer     →  module-cross-features-changes.md
STEP 4b: master-implementation-architect →  master-implementation-roadmap.md
STEP 5: feature-implementation-orchestrator → CODE
STEP 6: post-implementation-documenter →  implementation-manual.md
```

---

## About the BES Architecture (Module Level)
- **Kernel-and-Plugin**: Extensions MUST NOT import from each other. Only Event Bus for cross-module communication.
- **Hub-and-Spoke MDM**: Global master data in `core` schema. Module tables as transactional spokes.
- **Licensing**: Each module toggle independently via `ACTIVE_EXTENSIONS` / `READONLY_EXTENSIONS`.
- **Nx Frontend**: Each module has an Nx library (`@bes/<module>`) registered in Shell via `ComponentRegistry.registerLazy()`.
- **Known Modules**: `core`, `finance`, `sales`, `inventory`, `hr`, `supply-chain`, `crm`, `settings`.

---

## Instructions for the Assistant

When the user asks to "module architect review the <module> module":

1. List all feature subdirectories in `features-plan/<module>/`.
2. Read every `frontend.md`, `backend.md`, and `changes.md` within the module.
3. Read `.agent/rules/architecture.md`, `.agent/rules/backend.md`, `.agent/rules/security.md`.
4. Perform the analysis below.
5. Write output to `features-plan/<module>/module-cross-features-changes.md`.

---

## Module-Wide Analysis

### A. Feature Inventory
For each feature: does it have `frontend.md`? `backend.md`? `changes.md`? Clearly flag what is missing.

### B. Data Model Audit
- Table name conflicts between features?
- Redundant entities across features (e.g., two features defining a tax table)?
- All features correctly referencing `core` master data (not defining their own)?
- Consistent use of `metadata_` Ghost FKs?

### C. Service & Logic Reuse
- Repeated business logic that should be in a shared `utils.py`?
- Common CRUD patterns that should use `core.BaseRepository`?
- Multi-feature transaction chains (e.g., Invoice → GL Posting) that must be event-driven?

### D. API Cohesion
- All endpoints use the same prefix `/api/v1/<module>`?
- Consistent resource naming across features?
- RBAC namespace is `<module>:*:*` consistently?

### E. Internal Event Catalog
Map all `UPPER_SNAKE_CASE` events emitted and subscribed within the module. Identify circular event loops. Identify events expected from external modules.

### F. Feature Dependency Graph
Which features must be built before others? Which have no dependencies (can be parallelized)?

---

## Output: `features-plan/<module>/module-cross-features-changes.md`

## 1. Module Overview
High-level architectural maturity assessment.

## 2. Feature Inventory Table

| Feature | frontend.md | backend.md | changes.md | Status |
|:--------|:-----------|:----------|:----------|:-------|

## 3. Cross-Feature Findings
- 🔄 Shared components & services to build once
- ⚠️ Data model conflicts & redundancies
- 🔗 Feature dependency graph (build order)
- ⚡ Internal event catalog
- 🛑 Golden Rule violations

## 4. Phased Module Build Order
- **Phase 1: Shared Foundation** — Shared models, utilities, permissions namespace.
- **Phase 2: Master Data Features** — No intra-module dependencies.
- **Phase 3: Transactional Features** — Depend on master data or other modules.
- **Phase 4: Integration & Events** — Cross-module event wiring.

---

### Execution Rules
- **Output file**: `features-plan/<module>/module-cross-features-changes.md`
- **Next step**: Run this for each module, then run `master-implementation-architect` (Step 4b).
- **Also run**: `5-project-readiness-auditor` for the full cross-module picture.
