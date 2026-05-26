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

1. Read the global bug registry [features-plan/build-bugs.md](file:///Users/bvk/BVK_Workspace/BES/features-plan/build-bugs.md) and the compiled prevention rules [.agents/rules/5-build-bug-prevention-rules.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/5-build-bug-prevention-rules.md). Identify any historical frontend, UI, or config issues relevant to your module or components, and note them as active quality constraints in your checklist.
2. Read `features-plan/<module-name>/<feature-name>/proposed-plan.md` and `2-ui-ux-flow.md`.
3. Use Graphify (`graphify query`) to check for the existing UI state and shared components.
4. Execute the implementation steps defined in `proposed-plan.md` for the UI phase.
5. Scaffold or modify the required React libraries and components in `bes-frontend/libs/<module>/`.
6. **Keep components modular**: Decompose complex views or pages into small, focused, and reusable sub-components. Do not write massive, single-file components (e.g. mixing forms, tables, and modals in a single file). Organize them in logical sub-directories under `libs/<module>/src/lib/components/` or by feature (e.g. `libs/<module>/src/lib/<feature>/components/`).
7. Integrate with the backend APIs.
8. Register the newly created views into the Shell's `ComponentRegistry`.
9. Wire up process loading: ensure module navigation routes hook into `loadModuleProcesses` dynamically to populate `sessionStorage`.
10. Delegate layering to `@bes/shared-ui` components. Do not hardcode or override `z-index` values in local component styles.
11. Enforce all UX laws and cognitive load reductions from `.agents/rules/2-frontend-architecture.md` (Aesthetic-Usability, Miller's Law, Fitts's Law, Error-Forgiving Design, Density Over Whitespace, and Persistent Context side panels/drawers).
12. Keep code beginner-friendly: write functional components, use descriptive variable names, provide extensive inline documentation for state logic, and expose clear TypeScript interfaces.

---

### Execution Rules
- **Output**: Code changes in `bes-frontend/`.
- **Next step**: Testing, or `5b-backend-build` if not complete.
