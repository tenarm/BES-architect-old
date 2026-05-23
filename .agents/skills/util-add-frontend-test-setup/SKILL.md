---
name: add-frontend-test-setup
description: This skill walks you through retrofitting an existing Nx frontend UI library with Vitest test configurations and global mocks.
---

# Skill: Retrofitting an Existing Frontend Library with Vitest Setup

This skill guides you through systematically adding Vitest test configuration to any existing Nx frontend library (e.g., `settings`, `inventory`, `sales`, etc.) that currently lacks a testing setup.

---

## Prerequisites
- The target library already exists under `libs/[lib-name]` (e.g., `libs/settings`).
- The global test setup file `libs/shared-ui/src/test-setup.ts` is available to provide common browser API mocks.

---

## Steps

### Step 1: Run the Nx Vitest Generator

From the workspace root (`BES/bes-frontend`), execute the generator for the target project:

```bash
npx nx generate @nx/vitest:configuration --project=[lib-name]
```

> [!NOTE]
> This command will:
> 1. Create a `vite.config.mts` inside the library directory.
> 2. Generate a `tsconfig.spec.json` referencing Vitest types.
> 3. Register the `test` target in the project's `project.json` or update inferred configuration.

---

### Step 2: Configure the Vitest Environment

Open the newly generated `libs/[lib-name]/vite.config.mts` and update the `test` block. You must:
1. Set the environment to `jsdom` (since it is a React/UI library).
2. Point the `setupFiles` list to the shared-ui global test-setup file to reuse browser API mocks (`sessionStorage`, `matchMedia`, `localStorage`, `EventSource`).
3. Set the statement, branch, and function coverage threshold to `80%` (if strict enforcement is desired).

Update the `test` block to match this structure:

```typescript
  test: {
    name: '[lib-name]',
    watch: false,
    globals: true,
    environment: 'jsdom',
    setupFiles: ['../../libs/shared-ui/src/test-setup.ts'],
    include: ['src/**/*.{test,spec}.{js,mjs,cjs,ts,mts,cts,jsx,tsx}'],
    reporters: ['default'],
    coverage: {
      reportsDirectory: '../../coverage/libs/[lib-name]',
      provider: 'v8' as const,
      // Uncomment if you want to enforce 80% coverage on this library
      // thresholds: {
      //   statements: 80,
      //   branches: 80,
      //   functions: 80,
      //   lines: 80,
      // }
    },
  },
```

---

### Step 3: Create a Placeholder Unit Test

Create a simple placeholder test under the library's `src/lib/` folder to confirm the testing plumbing is functional and JSDOM and `@testing-library/react` can mount components.

Create `libs/[lib-name]/src/lib/[lib-name]-setup.spec.tsx` (replace `[lib-name]` with the actual folder name):

```typescript
import React from 'react';
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';

// Simple placeholder component to verify mounting
const PlaceholderComponent = () => <div>Vitest Active</div>;

describe('Vitest Setup Verification', () => {
  it('should successfully mount a component in JSDOM', () => {
    render(<PlaceholderComponent />);
    expect(screen.getByText('Vitest Active')).toBeInTheDocument();
  });
});
```

---

### Step 4: Verify the Configuration

Execute the test suite to verify the configuration is correctly wired up and passes:

```bash
# From workspace root
cd bes-frontend
npx nx test [lib-name]
```

Ensure the test passes cleanly:
```text
✓  [lib-name]  src/lib/[lib-name]-setup.spec.tsx (1 test)
Test Files  1 passed (1)
     Tests  1 passed (1)
```
