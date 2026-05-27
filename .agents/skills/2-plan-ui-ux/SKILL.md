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
2. Generate the 6-section document below.
3. Save to `features-plan/<module-name>/<feature-name>/2-ui-ux-flow.md`.

---

## Required 6-Section Structure

## 1. Module & Feature Name
State the module and feature.

## 2. Information Architecture
Describe the hierarchy of pages, tabs, and drawer panels for the feature. 
*Compliance Check:* Outline how this architecture supports a **System-Directed Mental Model** matching the rest of the application.
*Context-Aware Routing & Navigation:*
- **Dedicated Route Mapping**: Align navigation with user expectations. If a sub-feature has a dedicated sidebar entry point (e.g., User Management), it deserves a dedicated page component to prevent layout overload. Minor, tightly-coupled settings (e.g., Tax Profiles, Calendars) should be consolidated into unified setup hubs (like Company Setup) to keep the sidebar clean.
- **Clean Content Partitioning**: Choose layout elements based on logical depth. Use horizontal tabs (`TabGroup`) to pivot between parallel datasets (e.g., Users vs. Roles). Use drawers or split-screens for detail drilling, maintaining a flat navigation structure without nested side bars.

## 3. Process Chain & Workflow
- **Macro View (`FloatingProcessPipeline`)**: Map out the visual process states (Pending, Active, Waiting Approval, Complete, Failed) for high-level workflow milestones.
- **Micro View (`Timeline`)**: Vertical historical event logs feed designed for the secondary "History" tab.

## 4. Shared UI Components
Map out exactly which `@bes/shared-ui` components will be used. Do not propose custom non-shared components.

## 5. UI-UX Flow & Walkthrough
Detail the step-by-step interactive UI-UX flow and visual states:
- **Interactive UI-UX Flow (User Journey)**: Describe the step-by-step user path from entering the screen, opening drawers/panels, mutating state, and closing interactions.
- **Context-Driven Layout Selection**: Explain why specific layout elements were selected:
  - *Side Drawers*: Best for side-by-side editing, auditing (Timeline), or inspecting items without losing grid context.
  - *Modals*: Best for transaction overrides, confirmation alerts, or quick standalone actions (e.g., password reset).
  - *Split-Screen / Dual Panels*: Best for high-frequency data entry where parent records and child grids must remain permanently visible.
- **Empty States**: Design for clean, empty-state illustrations with clear Call-to-Actions (CTAs) guiding users to create their first record.
- **Loading Skeletons**: Map out modern skeleton loading layouts matching the data structure to keep user perception fast.
- **Data Table Columns**: Structure columns to maximize **Density Over Whitespace** with tight paddings and visual status tags.
- **Form Inputs & Progressive Disclosure**: For forms exceeding 5-7 fields, detail how progressive disclosure (e.g., collapsible sections, tabs, or step wizards) is used to avoid cognitive overload.
- **Core Transactional Value UI Focus**: Prioritize the basic CRUD views (simple list grids, basic input forms) to keep the primary interface uncluttered. Move premium/complex elements (such as OTIF widgets, document drawers, or multi-step approval stages) into separate tabs, collapsible panels, or gated upgrade overlays.


## 6. Cognitive UX Design Compliance
Explain how the design balances density and usability by applying cognitive UX principles (Miller's Law, Fitts's Law, Error-Forgiving Design, Aesthetic-Usability Effect, Density Over Whitespace, and Persistent Context) in a smart, integrated way as defined in [2-frontend-architecture.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/2-frontend-architecture.md).

---

### Execution Rules
- **Output file**: `features-plan/<module-name>/<feature-name>/2-ui-ux-flow.md`
- **Next step**: Run `3-plan-backend-yaml`.
