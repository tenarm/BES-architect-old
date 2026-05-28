---
name: create-ui-module
description: This skill walks you through creating a new Nx library for a BES module's UI components and registering them with the Shell.
---

# Skill: Create a New Frontend UI Module Library

This skill walks you through creating a new Nx library for a BES module's UI components and registering them with the Shell.

---

## Prerequisites
- The backend extension for this module already exists.
- Module name matches the backend extension (e.g., `inventory`).
- `@bes/shared-ui` library is available.

## Steps

### Step 1: Generate the Nx Library & Test Configuration

Run these commands from the workspace root (`BES/bes-frontend`):

#### 1. Generate the library skeleton:
```bash
npx nx generate @nx/react:library [module-name] \
  --directory=libs/[module-name] \
  --importPath=@bes/[module-name] \
  --bundler=none \
  --unitTestRunner=none \
  --style=css \
  --no-interactive
```

#### 2. Systematically add Vitest test configurations (inferred test target):
```bash
npx nx generate @nx/vitest:configuration --project=[module-name]
```

#### 3. Update the generated `libs/[module-name]/vite.config.mts` to reference the shared UI global test setup:
Modify the `test` block inside `libs/[module-name]/vite.config.mts` to configure the `jsdom` environment and point `setupFiles` to the shared UI test-setup file to avoid duplicating global mocks:
```typescript
  test: {
    name: '[module-name]',
    watch: false,
    globals: true,
    environment: 'jsdom',
    setupFiles: ['../../libs/shared-ui/src/test-setup.ts'],
    include: ['src/**/*.{test,spec}.{js,mjs,cjs,ts,mts,cts,jsx,tsx}'],
    reporters: ['default'],
    coverage: {
      reportsDirectory: '../../coverage/libs/[module-name]',
      provider: 'v8' as const,
    },
  },
```

> [!NOTE]
> Replace `[module-name]` with your actual module identifier (e.g., `inventory`, `hr`).

### Step 2: Create the Module Entry Point

Replace the content of `libs/[module-name]/src/index.ts` with the following:

```typescript
// libs/[module-name]/src/index.ts
import { ComponentRegistry } from '@bes/shared-ui';

/**
 * Initialize the module and register its components.
 * This is called by the Shell during application startup.
 */
export function init[ModuleName]Module() {
  console.log('Initializing [ModuleName] Module...');

  // Use registerLazy for better code-splitting and performance
  ComponentRegistry.registerLazy('Route_[ModuleName]', () =>
    import('./lib/[module-name]-home').then(m => ({ default: m.[ModuleName]HomePage }))
  );
}

// Re-export for direct usage if needed
export { [ModuleName]HomePage } from './lib/[module-name]-home';
```



### Step 3: Create the Main Page Component

Create `libs/[module-name]/src/lib/[module-name]-home.tsx`:

```typescript
// libs/[module-name]/src/lib/[module-name]-home.tsx
import React, { useState, useEffect } from 'react';
import { Layout } from 'lucide-react'; // Use appropriate icon
import { Button, Skeleton } from '@bes/shared-ui';

export const [ModuleName]HomePage: React.FC = () => {
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Simulate initial data fetch
    const timer = setTimeout(() => setLoading(false), 1000);
    return () => clearTimeout(timer);
  }, []);

  return (
    <div style={{ padding: 'var(--ui-spacing-lg)' }}>
      <header style={{ 
        display: 'flex', 
        justifyContent: 'space-between', 
        alignItems: 'center', 
        marginBottom: 'var(--ui-spacing-lg)' 
      }}>
        <div>
          <h2 style={{ 
            fontSize: 'var(--ui-text-xl)', 
            fontWeight: '700', 
            color: 'var(--ui-gray-900)', 
            margin: 0 
          }}>[Module Display Name]</h2>
          <p style={{ 
            fontSize: 'var(--ui-text-sm)', 
            color: 'var(--ui-gray-500)', 
            marginTop: '4px' 
          }}>Module description or subtitle goes here.</p>
        </div>
        <Button variant="primary" size="sm">
          Action Button
        </Button>
      </header>

      <div style={{ 
        background: 'var(--ui-white)', 
        borderRadius: 'var(--ui-radius-lg)', 
        border: '1px solid var(--ui-gray-200)', 
        padding: 'var(--ui-spacing-xl)',
        minHeight: '400px'
      }}>
        {loading ? (
          <Skeleton height="300px" />
        ) : (
          <div>
            {/* Main Module Content Goes Here */}
            <p>Welcome to the [ModuleName] module.</p>
          </div>
        )}
      </div>
    </div>
  );
};
```

### Step 3b: Create the Unit Test

Create a placeholder unit test to verify that the page renders correctly under Vitest:

`libs/[module-name]/src/lib/[module-name]-home.spec.tsx`:

```typescript
// libs/[module-name]/src/lib/[module-name]-home.spec.tsx
import React from 'react';
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { [ModuleName]HomePage } from './[module-name]-home';

describe('[ModuleName]HomePage', () => {
  it('renders the page header and title successfully', () => {
    render(<[ModuleName]HomePage />);
    expect(screen.getByText('[Module Display Name]')).toBeInTheDocument();
  });
});
```

### Step 4: Register in the Shell

> [!CAUTION]
> Per Rule 5 (Build Bug Prevention), you MUST do both of the following or the module will be invisible in the sidebar.

#### 4a. Add the module initializer to `apps/shell/src/store/auth-store.ts`:

Find the `initializeModules` function and add your module's initializer:

```typescript
// apps/shell/src/store/auth-store.ts
import { init[ModuleName]Module } from '@bes/[module-name]';

// Inside initializeModules():
init[ModuleName]Module();
```

#### 4b. Add the module key to `allModulesList` in `apps/shell/src/hooks/use-shell.ts`:

```typescript
// apps/shell/src/hooks/use-shell.ts
const allModulesList = [
  // ... existing modules
  '[Module Display Name]',  // Must match RESOURCE_NAMES key
];
```

### Step 5: Add Icons and Configuration

Update `apps/shell/src/app/app-config.tsx` to ensure the module appears in the sidebar:

```typescript
// apps/shell/src/app/app-config.tsx
import { [IconName] } from 'lucide-react';

// ...

// 1. Add to MODULE_ICONS
export const MODULE_ICONS: Record<string, React.ReactNode> = {
  // ...
  [module-name]: <[IconName] size={20} />,
};

// 2. Register display names in @bes/shared-ui/src/lib/constants.ts
// Add the [module-name] key to MODULE_NAMES and RESOURCE_NAMES maps.
```

> [!TIP]
> Ensure the key used in `MODULE_ICONS` matches the `[module-name]` identifier used throughout.



### Step 6: Verify

#### 1. Run the Unit Tests:
Run the Vitest suite to verify the test setup and placeholder test pass successfully:
```bash
# From workspace root
cd bes-frontend
npx nx test [module-name]
```

#### 2. Run the Development Server:
Verify that the module functions normally in the local UI:
```bash
npm run dev
```

1. Login to the BES application.
2. Verify that your new module appears in the sidebar with the assigned icon.
3. Click the module and verify that the placeholder page renders correctly.

## Component Naming Conventions

| Registry Key | Purpose | Example |
|:-------------|:--------|:--------|
| `Route_[ModuleName]` | Main module page | `Route_Inventory` |
| `Route_[Resource]` | Sub-resource page | `Route_PurchaseOrder` |
| `Widget_[ModuleName][Name]` | Dashboard widget | `Widget_InventorySummary` |
| `Panel_[ModuleName][Name]` | Side panel | `Panel_InventoryItemDetail` |

