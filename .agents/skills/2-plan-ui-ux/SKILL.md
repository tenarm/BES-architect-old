---
name: 2-plan-ui-ux
description: "PLANNING STEP 2: Create the information architecture, UI Design, and UX flow in detail."
---

# Skill: Plan UI/UX
**Lifecycle Position: STEP 2 of 6 — UI/UX Planning**
**Feeds into:** `3-plan-backend-yaml` and `4-plan-reviewer`

This skill defines the user interface and journey for the functionalities mapped out in Step 1.

---

## Lifecycle Context
```
STEP 1: 1-plan-functionalities  →  1-functionalities.md
[YOU ARE HERE]
STEP 2: 2-plan-ui-ux            →  2-ui-ux-flow.md
STEP 3: 3-plan-backend-yaml     →  3-backend-plan.md
STEP 4: 4-plan-reviewer         →  proposed-plan.md & common-dependants.md
STEP 5: 5a/5b build             →  CODE
STEP 6: 6-bug-fixing            →  BUG FIXES
```

---

## About the BES Architecture
- **Shared UI**: ALL components come from `@bes/shared-ui` (Table, Drawer, Skeleton, Timeline, ProcessPipeline, etc.).
- **Process Transparency**: Workflow pipelines are visible. Drawers use `ProcessPipeline` for macros and `Timeline` for micros.
- **Licensing**: UI must degrade gracefully for `READONLY_EXTENSIONS`.

---

## Instructions for the Assistant

When the user asks to plan UI/UX:

1. Read the output of `features-plan/<module-name>/<feature-name>/1-functionalities.md`.
2. Generate the 5-section document below.
3. Save to `features-plan/<module-name>/<feature-name>/2-ui-ux-flow.md`.

---

## Required 5-Section Structure

## 1. Module & Feature Name
State the module and feature.

## 2. Information Architecture
Describe the hierarchy of pages, tabs, and drawer panels for the feature.

## 3. Process Chain & Workflow
- **Macro View (`FloatingProcessPipeline`)**: Glassmorphic vertical stepper overlay loaded on demand and cached in `sessionStorage` for immediate first paint, hydrated live via Server-Sent Events (SSE). Map out the visual states (Pending, Active, Waiting Approval, Complete, Failed).
- **Approval Registry Modals**: Identify any custom verification forms that need registration in the central `ComponentRegistry`.
- **Micro View (`Timeline`)**: Vertical historical log feed in a "History" secondary tab.
- **Layering Compliance**: Confirm that layering is delegated entirely to the built-in styles of the `@bes/shared-ui` components. Warn against specifying manual inline `z-index` properties.

## 4. Shared UI Components
Map out exactly which `@bes/shared-ui` components will be used. Do not propose custom non-shared components.

## 5. UI States & Walkthrough
Detail the step-by-step UX walkthrough and states:
- Empty states
- Loading skeletons
- Error boundaries
- Data Table columns
- Form inputs in the Drawer

---

### Execution Rules
- **Output file**: `features-plan/<module-name>/<feature-name>/2-ui-ux-flow.md`
- **Next step**: Run `3-plan-backend-yaml`.
