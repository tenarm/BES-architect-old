---
name: master-implementation-architect
description: "LIFECYCLE STEP 4c: Read all planning documents and existing code to produce the master-implementation-roadmap.md — the definitive sequenced action plan for implementation."
---

# Skill: Master Implementation Architect
**Lifecycle Position: STEP 4c of 6 — Strategy & Roadmap**
**Reads from:**
- `features-plan/project-readiness-report.md` (from Step 4b)
- All `features-plan/<module>/module-cross-features-changes.md` (from Step 4a)
- All `features-plan/<module>/<feature>/changes.md` (from Step 3)
- All `features-plan/<module>/<feature>/frontend.md` + `backend.md` (from Steps 1 & 2)
- Existing code in `bes-backend/extensions/` and `bes-frontend/libs/`

**Feeds into:** `ACTION-feature-implementation-orchestrator` (Step 5)

This is the "Master Controller" skill. It synthesizes everything — plans AND existing code — into a single, sequenced, optimized action plan. The resulting `master-implementation-roadmap.md` is the definitive document the developer follows to build the product.

---

## Lifecycle Context
```
STEP 1: frontend-generate-feature-doc  →  frontend.md
STEP 2: backend-generate-feature-doc   →  backend.md
STEP 3: feature-plan-reviewer          →  changes.md
STEP 4a: module-architect-reviewer     →  module-cross-features-changes.md
STEP 4b: project-readiness-auditor     →  project-readiness-report.md
[YOU ARE HERE]
STEP 4c: master-implementation-architect →  master-implementation-roadmap.md
STEP 5: feature-implementation-orchestrator → CODE
STEP 6: post-implementation-documenter →  implementation-manual.md
```

---

## Instructions for the Assistant

When the user asks to "create a master implementation plan" or "master architect plan":

### 1. Full Context Ingestion
Read ALL of the following in order:
1. `features-plan/project-readiness-report.md` — global priorities, Golden Rules compliance, critical gaps.
2. All `features-plan/<module>/module-cross-features-changes.md` — module-level shared services, event catalogs, dependency graphs.
3. All `features-plan/<module>/<feature>/changes.md` — individual feature specs and implementation orders.
4. All `frontend.md` and `backend.md` — domain details and UI nuances.
5. **Existing code scan**: `bes-backend/extensions/` and `bes-frontend/libs/` — what is already built, partially built, or needs refactoring.

### 2. Analysis & Strategy

#### A. Optimization of Existing Code
Identify anything in the current codebase that doesn't meet the Golden Rules:
- Missing `subsidiary_id` filters in services.
- Float arithmetic (violates Money Rule).
- Hard-coded strings instead of enums.
- Missing `require_permission()` guards.
- Old component patterns that should be migrated to `@bes/shared-ui`.

#### B. UI Excellence Plan
Define how every new screen will achieve "Best-in-Class" quality:
- **`@bes/shared-ui` mandate**: Every table, form, skeleton, and drawer comes from the shared library.
- **Transparency UX**: `ProcessPipeline` (macro status) + `Timeline` (micro audit) for all transactional features.
- **`ComponentRegistry.registerLazy()`**: All components lazily loaded to prevent bundle bloat.
- **READONLY degradation**: Every screen has a clearly planned readonly mode.

#### C. Implementation Sequencing
Build the global sequence based on:
1. Cross-module dependencies (e.g., Finance needs `core.customers` before AR).
2. Feature dependencies within modules (e.g., COA before GL).
3. Shared foundation work that unblocks multiple features.

### 3. Output: `features-plan/master-implementation-roadmap.md`

---

## Master Roadmap Structure

## 1. Executive Implementation Strategy
High-level approach (e.g., "Foundation-First Vertical Slicing") and key architectural decisions.

## 2. Existing Code Optimization Tasks
Checklist of refactoring tasks for currently implemented features before new work begins:
- [ ] Task: file, issue, fix required.

## 3. Sequential Action Plan
Each step is a fully defined work unit:

### Step [N]: [Feature Name] (`features-plan/<module>/<feature>/`)
- **Status**: `Ready` / `Needs More Planning` / `Blocked by Step X`
- **Source Documents**:
  - Primary: `features-plan/<module>/<feature>/changes.md`
  - Reference: `frontend.md`, `backend.md`
  - Module Context: `module-cross-features-changes.md`
- **Depends On**: [List prior steps]
- **Backend Focus**: Specific service/model details to implement. Shared service to extract.
- **UI Focus**: Specific `@bes/shared-ui` patterns. `ProcessPipeline` and `Timeline` integration points. Registry key.
- **Events**: Events to wire (emitter → subscriber).
- **Skill to Use**: `util-create-extension` / `util-create-ui-module` / `util-add-api-endpoint` / `ACTION-feature-implementation-orchestrator`.

## 4. Cross-Module Integration Milestones
Key points where multiple modules must be tested together:
- O2C: Sales → Inventory → Finance.
- P2P: Supply Chain → Inventory → Finance.

## 5. UI/UX Global Consistency Checklist
Final audit across all planned screens:
- [ ] All tables use `@bes/shared-ui` DataTable.
- [ ] All forms use Drawer with ProcessPipeline + Timeline tabs.
- [ ] All modules handle READONLY mode.
- [ ] All pending items surfaced in Home pipeline.

---

### Execution Rules
- **Output file**: `features-plan/master-implementation-roadmap.md`
- **Next step**: For each Step in the roadmap, trigger `ACTION-feature-implementation-orchestrator` — it will use the documents referenced in each step.
- **Utility skills available**: `util-create-extension`, `util-create-ui-module`, `util-add-api-endpoint`.
