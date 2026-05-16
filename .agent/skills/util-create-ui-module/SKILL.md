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

### Step 1: Generate the Nx Library

Run this command from the workspace root (`BES/bes-frontend`):

```bash
npx nx generate @nx/react:library [module-name] \
  --directory=libs/[module-name] \
  --importPath=@bes/[module-name] \
  --bundler=none \
  --unitTestRunner=none \
  --style=css \
  --no-interactive
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



### Step 4: Register in the Shell

Update `apps/shell/src/main.tsx`:

```typescript
// apps/shell/src/main.tsx
import { init[ModuleName]Module } from '@bes/[module-name]';

// ... other imports

// Initialize modules alongside existing ones
init[ModuleName]Module();
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

```bash
# From workspace root
cd bes-frontend
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

