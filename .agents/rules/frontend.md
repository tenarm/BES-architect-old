# Frontend Coding Rules — BES Factory

These rules govern all React/TypeScript code in the `bes-frontend/` Nx monorepo.

---

## 1. Monorepo Structure

```
bes-frontend/
├── apps/shell/          # Main shell application (entry point)
├── libs/shared-ui/      # Design tokens, base components, ComponentRegistry
├── libs/finance/         # Finance module UI library
├── libs/sales/           # Sales module UI library
└── libs/<module>/        # One library per BES module
```

- The Shell app MUST NOT contain module-specific business UI.
- Each module's UI lives in its own Nx library under `libs/`.
- All modules register their components via `ComponentRegistry` in their `index.ts`.

## 2. ComponentRegistry Pattern

Modules self-register their components in the Shell using the global registry:

```typescript
// libs/finance/src/index.ts
import { ComponentRegistry } from '@bes/shared-ui';
import { ChartOfAccountsPage } from './lib/coa/coa-page';

export function initFinanceModule() {
  ComponentRegistry.register('Route_FinanceMain', ChartOfAccountsPage);
  ComponentRegistry.register('Widget_FinanceSummary', FinanceDashWidget);
}
```

### Naming Conventions:
- Route components: `Route_<ModuleName>Main`
- Dashboard widgets: `Widget_<ModuleName><Widget>`

## 3. State Management

- Use **Zustand** for global state (auth, user session, navigation).
- Each store MUST be in `apps/shell/src/store/` directory.
- Module-specific state stays within the module library (local React state or module-level Zustand store).
- NEVER use React Context for global state.

## 4. RBAC in the UI

- All permission checking uses the `checkPermission()` utility from `@bes/shared-ui`.
- Permission format: `"module:resource:action"` (e.g., `"finance:coa:write"`).
- The sidebar dynamically renders based on `currentUser.permissions`.
- Dev-only features MUST be gated behind `import.meta.env.DEV`.

```typescript
import { checkPermission } from '@bes/shared-ui';

const canEdit = checkPermission(user.permissions, 'finance:coa:write');
```

## 5. API Integration

- Use the Vite proxy for API calls: the shell's `vite.config.mts` proxies `/api` to the backend.
- API base URL: `/api/v1` (relative — proxied to backend).
- Always handle loading, error, and empty states.
- Use the auth store's token for authenticated requests:
  ```typescript
  const token = useAuthStore.getState().token;
  fetch('/api/v1/endpoint', {
    headers: { Authorization: `Bearer ${token}` }
  });
  ```

## 6. Styling

- Use **vanilla CSS** with CSS custom properties (design tokens).
- Design tokens are defined in `libs/shared-ui/src/lib/styles/design-tokens.css`.
- Primary color: `#162867` (`var(--ui-primary)`).
- NEVER use inline styles for reusable components — use CSS modules or shared stylesheets.
- Inline styles are acceptable ONLY for one-off layout adjustments.

## 7. Component Guidelines

- Use functional components exclusively (no class components).
- Prefer composition over prop drilling.
- All interactive elements MUST have unique, descriptive `id` attributes for testing.
- Use `React.useMemo` and `React.useCallback` for expensive computations.
- Export components as named exports (not default) from libraries.

## 8. TypeScript

- Strict mode is enabled — no `any` types in production code.
- Use `interface` for component props and object shapes.
- Use `type` for unions, intersections, and utility types.
- Path aliases: `@bes/shared-ui`, `@bes/finance`, `@bes/sales` (defined in `tsconfig.base.json`).

## 9. Authentication Flow

1. User submits credentials via the login form.
2. `auth-store.login()` calls `POST /api/v1/auth/login`.
3. On success, stores `access_token` and `refresh_token` in localStorage.
4. Calls `GET /api/v1/auth/me` to fetch user profile and permissions.
5. Sets `isAuthenticated = true`, renders the Shell.
6. On token expiry, use `POST /api/v1/auth/refresh` with the refresh token.
