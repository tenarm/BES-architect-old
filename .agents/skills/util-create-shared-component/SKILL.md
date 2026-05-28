---
name: create-shared-component
description: This skill walks you through creating a high-quality, reusable UI component in the @bes/shared-ui library.
---

# Skill: Create Shared UI Component

This skill guides building a premium, reusable component for the `@bes/shared-ui` library. All modules import from this shared library to maintain design consistency. Follow every step.

---

## Prerequisites
- The `@bes/shared-ui` library exists at `bes-frontend/libs/shared-ui/`
- You have identified a component need from a `2-ui-ux-flow.md` plan or during module development

---

## Steps

### Step 1: Research — Verify Component Doesn't Already Exist

Before creating anything:

1. Check the shared-ui barrel export: `bes-frontend/libs/shared-ui/src/index.ts`
2. Check existing components directory: `bes-frontend/libs/shared-ui/src/lib/components/`
3. Use Graphify: `graphify query "Does a <ComponentName> component exist in shared-ui?"`
4. Check `features-plan/common-dependants.md` for any outbound shared-ui entries from other modules.

If a similar component exists, extend it rather than creating a duplicate.

### Step 2: Design — Define the Interface

Before writing code, define:

1. **TypeScript Props Interface** (strict mode, no `any` types per Rule 2 §6):
   ```typescript
   interface ComponentNameProps {
     /** Unique ID for testing (required per Rule 2 §6) */
     id: string;
     /** Component-specific props... */
     variant?: 'default' | 'primary' | 'danger';
     size?: 'sm' | 'md' | 'lg';
     disabled?: boolean;
     children?: React.ReactNode;
     /** Event handlers */
     onChange?: (value: string) => void;
     onClose?: () => void;
   }
   ```

2. **CSS Custom Properties** — Identify which design tokens to use:
   - Colors: `var(--ui-primary)`, `var(--ui-gray-*)`, `var(--ui-danger)`, `var(--ui-success)`
   - Spacing: `var(--ui-spacing-xs)` through `var(--ui-spacing-xl)`
   - Radius: `var(--ui-radius-sm)`, `var(--ui-radius-md)`, `var(--ui-radius-lg)`
   - Typography: `var(--ui-text-sm)`, `var(--ui-text-base)`, `var(--ui-text-lg)`

3. **Keyboard Accessibility Plan** (Rule 2 §11):
   - Which keyboard events should the component handle? (Tab, Enter, Escape, Arrow keys)
   - What ARIA attributes are needed? (`aria-label`, `role`, `aria-expanded`, etc.)
   - Focus management requirements (focus trap for modals, focus return on close)

### Step 3: Implementation

#### 3a. Create the component directory:
```
bes-frontend/libs/shared-ui/src/lib/components/<ComponentName>/
├── <ComponentName>.tsx
├── <ComponentName>.module.css
├── <ComponentName>.spec.tsx
└── index.ts
```

#### 3b. Component File (`<ComponentName>.tsx`):

```typescript
import React from 'react';
import styles from './<ComponentName>.module.css';

export interface ComponentNameProps {
  id: string;
  // ... props from Step 2
}

/**
 * <ComponentName> — Brief description of what this component does.
 *
 * @example
 * <ComponentName id="my-component" variant="primary" />
 */
export const ComponentName: React.FC<ComponentNameProps> = ({
  id,
  variant = 'default',
  // ... destructure props
}) => {
  return (
    <div
      id={id}
      className={`${styles.root} ${styles[variant]}`}
      role="..."       // Appropriate ARIA role
      aria-label="..." // Descriptive label
    >
      {/* Component content */}
    </div>
  );
};
```

#### 3c. Styles (`<ComponentName>.module.css`):

> **CRITICAL**: Use vanilla CSS only. NO Tailwind. NO inline styles (Rule 2 §6).

```css
.root {
  font-family: var(--ui-font-family);
  border-radius: var(--ui-radius-md);
  transition: all 0.2s ease;
}

/* Variant styles */
.primary {
  background: var(--ui-primary);
  color: var(--ui-white);
}

.default {
  background: var(--ui-white);
  border: 1px solid var(--ui-gray-200);
  color: var(--ui-gray-900);
}

/* Interactive states */
.root:hover:not(.disabled) {
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.root:focus-visible {
  outline: 2px solid var(--ui-primary);
  outline-offset: 2px;
}

.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

#### 3d. Barrel Export (`index.ts`):
```typescript
export { ComponentName } from './<ComponentName>';
export type { ComponentNameProps } from './<ComponentName>';
```

### Step 4: Export from Library

Add to `bes-frontend/libs/shared-ui/src/index.ts`:
```typescript
export { ComponentName } from './lib/components/<ComponentName>';
export type { ComponentNameProps } from './lib/components/<ComponentName>';
```

### Step 5: Accessibility Checklist

Before marking the component complete, verify:

- [ ] **Keyboard**: Component is fully operable via keyboard (Tab, Enter, Escape)
- [ ] **ARIA**: Correct `role`, `aria-label`, `aria-expanded`, `aria-disabled` attributes
- [ ] **Focus**: `:focus-visible` styles are visible and meet 3:1 contrast ratio
- [ ] **Labels**: All interactive children have accessible names
- [ ] **Color**: Visual information is not conveyed by color alone
- [ ] **Motion**: Animations respect `prefers-reduced-motion` media query

### Step 6: Write Tests

Create `<ComponentName>.spec.tsx`:

```typescript
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { ComponentName } from './<ComponentName>';

describe('<ComponentName>', () => {
  it('renders with default props', () => {
    render(<ComponentName id="test" />);
    expect(screen.getByRole('...')).toBeInTheDocument();
  });

  it('applies variant class correctly', () => {
    const { container } = render(<ComponentName id="test" variant="primary" />);
    expect(container.firstChild).toHaveClass('primary');
  });

  it('handles keyboard events', () => {
    const onClose = vi.fn();
    render(<ComponentName id="test" onClose={onClose} />);
    fireEvent.keyDown(screen.getByRole('...'), { key: 'Escape' });
    expect(onClose).toHaveBeenCalled();
  });

  it('renders in disabled state', () => {
    render(<ComponentName id="test" disabled />);
    expect(screen.getByRole('...')).toHaveAttribute('aria-disabled', 'true');
  });
});
```

### Step 7: Register in Showcase App

Add an interactive demo to `bes-frontend/apps/showcase/`:

1. Create a demo section showing the component in all variants, sizes, and states.
2. Include interactive controls so developers can experiment with props.
3. Show code examples for common usage patterns.

### Step 8: Verify

```bash
cd bes-frontend
npx nx test shared-ui        # Tests pass
npx nx lint shared-ui         # No lint errors
npx nx build shell            # Full build succeeds (no missing exports)
```

---

## Component Quality Standards

| Aspect | Requirement |
|:---|:---|
| **Styling** | Vanilla CSS + CSS custom properties only. NO Tailwind, NO inline styles |
| **TypeScript** | Strict types, named exports, `interface` for props |
| **Testing** | Unit tests with @testing-library/react |
| **A11y** | Keyboard navigable, ARIA attributes, focus-visible styles |
| **Premium Look** | Micro-animations, hover effects, smooth transitions, modern typography |
| **Reusability** | Generic props, no module-specific logic, theming via CSS variables |
