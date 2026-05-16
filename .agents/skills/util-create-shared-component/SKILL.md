---
name: create-shared-component
description: This skill walks you through creating a high-quality, reusable UI component in the @bes/shared-ui library.
---

# Skill: Create Shared UI Component

This skill instructs the AI assistant to expand the `@bes/shared-ui` library by building a premium, reusable component. This ensures that the design system stays consistent and that modules don't build "one-off" low-quality UI.

---

## Instructions for the Assistant

When you identify a missing generic component while building a module:

### 1. Design Strategy
- **Visual Excellence**: The component MUST look premium. Use modern typography (Inter/Roboto), subtle micro-animations, smooth transitions, and a curated color palette.
- **Generic Props**: Design the component to be flexible (e.g., customizable colors, sizes, and event handlers).
- **Aesthetics**: Avoid basic HTML looks. Use shadows, rounded corners (8px–12px), and hover effects.

### 2. Implementation Steps
1. **Create the Folder**: Create a new directory in `bes-frontend/libs/shared-ui/src/lib/components/<component-name>/`.
2. **Component File**: Create the React component file (e.g., `Button.tsx`).
3. **Styles**: Use Vanilla CSS or Tailwind utility classes that match the BES design tokens.
4. **Export**: Export the component from the folder's `index.ts`.
5. **Shared Library Export**: Ensure the component is exported from `bes-frontend/libs/shared-ui/src/index.ts` so modules can import it via `@bes/shared-ui`.

### 3. Verification
- Create a simple test page or use a component playground to verify the component renders correctly and handles props as expected.
- Check for responsiveness (Mobile/Desktop).
