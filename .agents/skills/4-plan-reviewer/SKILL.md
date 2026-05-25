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
- This phase verifies adherence to `.agents/rules/` including `BESBase`, Money Rule, Soft Deletes, `@bes/shared-ui`, and `4-planning-dependencies.md`.
- Managing a massive app built in stages requires tracing schema/database references (e.g. settings configs, FK tables, master catalogs) to ensure base entities are implemented in the correct topological order.

---

## Instructions for the Assistant

When reviewing the plan:

1. Read `1-functionalities.md`, `2-ui-ux-flow.md`, and `3-backend-plan.md`.
2. Use `graphify path` to verify relationships and check for dependencies on other modules.
3. **Trace Domain, Schema, & Code Dependencies**: 
   - **Inbound (Prerequisites)**: Explicitly identify database schemas, settings configurations, master data catalogs, or endpoints that this feature relies on. Determine the exact build order required to prevent compilation or database migration failures.
   - **Outbound (Shareable Assets)**: Identify any new database tables, lookup endpoints, event models, configuration parameters, or UI widgets this feature introduces that will be referenced by upcoming features in later stages.
4. Evaluate the plan for architectural compliance.
5. Generate the 4-section `proposed-plan.md` document below.
6. **Update the Shared Reference Registry**: If the feature has dependencies or exposes shareable assets, update `features-plan/common-dependants.md` under its respective module/feature section. Maintain two distinct tables:
   - **Required External Dependencies (Inbound)**: Outline the required entities, their parent module, description of the reference, and the pre-requisite build order impact.
   - **Exposed Reusable Assets & Shared Reference Notes (Outbound)**: Detail the exposed models, APIs, events, or UI widgets, who consumes them, and implementation/integration notes for upcoming stages.
7. Save to `features-plan/<module-name>/<feature-name>/proposed-plan.md`.

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
- **Event-Driven Notification Setup**: Verify that default database notification rule seeds (`NotificationRule`) are correctly defined for emitted Pub/Sub events:
  - Channels are gated correctly based on tiers (e.g. `IN_APP` for basic, `EMAIL` gated to Pro/Premium).
  - Templates utilize clean Jinja2 styling and recipient dynamic paths are properly set via JSONPath (e.g. `$.data.created_by`).
  - The UI Notification Bell listens via the SSE `/stream` endpoint and handles micro-interactions (vibration/pulsing) and metadata-driven navigation.

## 3. Cross-Module Dependencies & Shared Reference Registry

### 3.1 Required External Dependencies & Build Prerequisites (Inbound)
List all external database models, settings configuration records, API routes, or event structures this feature references. State the specific parent module/feature and the pre-requisite build order impact (must be implemented before this feature). (These must also be copied to `common-dependants.md` under Required External Dependencies).

### 3.2 Exposed Reusable Assets & Shared Reference Notes (Outbound)
List database tables/models, configuration keys, API endpoints, event schemas, reusable UI components, or utilities introduced here that subsequent stages or other features will import/reference. Provide integration/referencing guidelines to keep the application build on track. (These must also be copied to `common-dependants.md` under Exposed Reusable Assets).

## 4. Implementation Sequence
Provide the exact step-by-step roadmap for implementation, split by feature and phase (e.g., Phase 1 DB, Phase 2 APIs, Phase 3 UI). Identify when stubbed endpoints or repository mocks are needed to isolate staged builds.

---

### Execution Rules
- **Output files**: 
  - `features-plan/<module-name>/<feature-name>/proposed-plan.md`
  - `features-plan/common-dependants.md` (if applicable)
- **Next step**: Run `5a-frontend-build` and `5b-backend-build`.

