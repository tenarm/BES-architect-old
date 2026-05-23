---
name: util-check-shared-ui
description: "UTILITY STEP: Scan all features-plan/**/2-ui-ux-flow.md files for required `@bes/shared-ui` components, verify their existence in the shared library, and ensure they are generic, themed, and registered in the showcase app."
---

# Skill: Check Shared-UI Components

This skill provides instructions on how to systematically audit the frontend shared component library (`@bes/shared-ui`) against requirements documented in the feature planning files under `features-plan/`.

---

## 1. Audit Procedure

To run a Shared UI compliance audit, follow these steps:

### Step 1: Scan for Planning Documents
Search recursively inside `features-plan/` for all `2-ui-ux-flow.md` files.

### Step 2: Extract Component Requirements
Inspect the `Shared UI Components` section (usually Section 4) and list all components expected from `@bes/shared-ui`.
*Example list:*
- `TabGroup`, `TabList`, `Tab`, `TabPanels`, `TabPanel`
- `DataTable`
- `SlideOutDrawer`
- `FloatingProcessPipeline`
- `Timeline`
- `PremiumLockIndicator`
- `UpgradeGateOverlay`
- `LoadingSkeleton`
- `FeedbackAlert`

### Step 3: Check Shared-UI Library Exports
Cross-reference the extracted component names with actual exports in `bes-frontend/libs/shared-ui/src/index.ts` or component files under `bes-frontend/libs/shared-ui/src/lib/`.
* Identify any component that is documented as required but missing from the shared library.

### Step 4: Validate Theme & Aesthetics
Ensure that any new or existing components conform to the BES design system tokens defined in the CSS custom properties of the workspace (e.g. typography, dark modes, glassmorphism, Outfit font, and primary brand colors like `#162867`).

### Step 5: Update the Showcase App
For any verified or new component:
* Verify that it has an interactive demonstration section inside the showcase app (`bes-frontend/apps/showcase/`).
* If not, add or update the showcase view to display the component in various states (e.g. empty states, loaded states, warning/error states).

---

## 2. Guidelines for Component Construction

When creating missing components:
1. **Generic Design**: Make components highly reusable. Use standard props for class names, styling overrides, event handlers, and child elements.
2. **Company Theme**: Match primary theme variables (e.g. `--color-primary`, HSL-curated color shades, and glassmorphic micro-animations).
3. **No Overrides**: Avoid hardcoded inline styling, custom z-indices, or Tailwind utility classes unless explicitly part of the theme variables.
