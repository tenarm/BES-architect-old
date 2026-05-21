---
name: 4-plan-reviewer
description: "IMPLEMENTATION STEP 1: Review the feature plan, ensure architectural compliance, and generate proposed-plan.md."
---

# Skill: Plan Reviewer
**Lifecycle Position: STEP 4 of 6 — Architectural Audit & Roadmapping**
**Feeds into:** `5a-frontend-build` and `5b-backend-build`

Before writing code, this skill acts as an architectural gatekeeper. It splits the plan into actionable execution steps and tracks cross-module dependencies.

---

## Lifecycle Context
```
STEP 1: 1-plan-functionalities  →  1-functionalities.md
STEP 2: 2-plan-ui-ux            →  2-ui-ux-flow.md
STEP 3: 3-plan-backend-yaml     →  3-backend-plan.md
[YOU ARE HERE]
STEP 4: 4-plan-reviewer         →  proposed-plan.md & common-dependants.md
STEP 5: 5a/5b build             →  CODE
STEP 6: 6-bug-fixing            →  BUG FIXES
```

---

## About the BES Architecture
- This phase verifies adherence to `.agents/rules/` including `BESBase`, Money Rule, Soft Deletes, and `@bes/shared-ui`.
- Cross-module dependencies are critical in the Hub-and-Spoke model and must be tracked globally.

---

## Instructions for the Assistant

When reviewing the plan:

1. Read `1-functionalities.md`, `2-ui-ux-flow.md`, and `3-backend-plan.md`.
2. Use `graphify path` to verify relationships and check for dependencies on other modules.
3. Evaluate the plan for architectural compliance.
4. Generate the 4-section `proposed-plan.md` document below.
5. If the module relies on or impacts another module, ALSO append to `features-plan/common-dependants.md`.
6. Save to `features-plan/<module-name>/<feature-name>/proposed-plan.md`.

---

## Required 4-Section Structure for proposed-plan.md

## 1. Feature & Scope
Summarize what is being built.

## 2. Architectural Compliance Sign-off
Provide a checklist confirming adherence to the rules:
- **Money Rule**: Currency columns configured with standard precision parameters (`Numeric(20,4)` in DB, `Decimal` in Python, and `decimal.js`/`big.js` in UI).
- **Soft Deletes**: Active queries configured to filter by `is_deleted == False`.
- **Database Architecture**: Core tables correctly extended and tenant bounds (`subsidiary_id`) strictly scoped.
- **Subscription Tier Gate Check**: Confirm that licensing bounds are properly defined in `packages.json`, FastAPI endpoints enforce bounds using `require_licensed_feature()`, and the frontend UI gracefully degrades using lock 🔒 indicators and attractive upgrade overlays rather than throwing errors.
- **Process Pipeline Configuration**: Verify that workflow definitions are saved in `extensions/<module_name>/<module_name>/process_definitions/<module_name>.json`, steps align with backend events, and frontend triggers `loadModuleProcesses` dynamically.
  - Verify that step-level licensing parameters (`requiredModule` or `requiredFeature`) are configured for any cross-module or gated steps to ensure clean backend self-healing.
- **Layering Encapsulation Compliance**: Audit code/designs to ensure no hardcoded or custom inline `z-index` styles are added. Layering must rely entirely on internal stacking default configurations within `@bes/shared-ui` components.

## 3. Cross-Module Dependencies
List any dependencies that affect other modules. (These must also be copied to `common-dependants.md`).

## 4. Implementation Sequence
Provide the exact step-by-step roadmap for implementation, split by feature and phase (e.g., Phase 1 DB, Phase 2 APIs, Phase 3 UI).

---

### Execution Rules
- **Output files**: 
  - `features-plan/<module-name>/<feature-name>/proposed-plan.md`
  - `features-plan/common-dependants.md` (if applicable)
- **Next step**: Run `5a-frontend-build` and `5b-backend-build`.

