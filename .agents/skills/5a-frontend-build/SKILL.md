---
name: 5a-frontend-build
description: "IMPLEMENTATION STEP 2a: Work on frontend design and development based on the reviewed plan."
---

# Skill: Frontend Build
**Lifecycle Position: STEP 5a of 6 — UI Implementation**

This skill executes the frontend development by rigidly following the `proposed-plan.md` step-by-step roadmap.

---

## Lifecycle Context
```
STEP 1: 1-plan-functionalities  →  1-functionalities.md
STEP 2: 2-plan-ui-ux            →  2-ui-ux-flow.md
STEP 3: 3-plan-backend-yaml     →  3-backend-plan.md
STEP 4: 4-plan-reviewer         →  proposed-plan.md & common-dependants.md
[YOU ARE HERE]
STEP 5: 5a-frontend-build       →  CODE
STEP 6: 6-bug-fixing            →  BUG FIXES
```

---

## About the BES Architecture
- **Stack**: React (Nx Monorepo) frontend.
- **Shared UI**: ALL components come from `@bes/shared-ui`.
- **Module Libraries**: Each module lives in `bes-frontend/libs/<module>/`.
- **Shell UI**: Registered via `init<Module>Module()` and `ComponentRegistry.registerLazy()`.

---

## Instructions for the Assistant

When asked to build the frontend:

1. Read `features-plan/<module-name>/<feature-name>/proposed-plan.md` and `2-ui-ux-flow.md`.
2. Use Graphify (`graphify query`) to check for the existing UI state and shared components.
3. Execute the implementation steps defined in `proposed-plan.md` for the UI phase.
4. Scaffold or modify the required React libraries and components in `bes-frontend/libs/<module>/`.
5. Integrate with the backend APIs.
6. Register the newly created views into the Shell's `ComponentRegistry`.
7. Wire up process loading: ensure module navigation routes hook into `loadModuleProcesses` dynamically to populate `sessionStorage`.
8. Delegate layering to `@bes/shared-ui` components. Do not hardcode or override `z-index` values in local component styles.

---

### Execution Rules
- **Output**: Code changes in `bes-frontend/`.
- **Next step**: Testing, or `5b-backend-build` if not complete.
