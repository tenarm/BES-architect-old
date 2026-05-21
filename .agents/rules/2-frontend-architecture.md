# 2. Frontend Architecture & Security Rules — BES

These rules govern all React/TypeScript code in the `bes-frontend/` Nx monorepo.

## 1. Monorepo Structure & Component Registry
- **Shell App**: `apps/shell/` is the entry point. It MUST NOT contain module-specific business UI.
- **Module Libraries**: Each module's UI lives in `libs/<module>/`. 
- **Shared UI**: Base components and design tokens MUST come from `libs/shared-ui/` (`@bes/shared-ui`).
- **ComponentRegistry**: Modules dynamically register their components in the Shell via `ComponentRegistry.register()` (e.g., `Route_FinanceMain`, `Widget_FinanceSummary`) during `init<Module>Module()`.

## 2. Process Transparency & Workflow
- **Process Chain**: Use the `FloatingProcessPipeline` component for dynamic process transparency. It renders as a glassmorphic vertical stepper hydrated in real time via Server-Sent Events (SSE).
- **Module-Level Pre-loading & Caching**: 
  - To ensure rapid "first paint" UI speeds, process definitions MUST be pre-loaded dynamically upon navigation (e.g., inside the shell's `toggleModule`).
  - Definitions are cached in `sessionStorage` under key `bes_module_procs_{module_name}` to allow the floating window to paint instantly, while live updates are stream-hydrated from SSE.
- **Process Workflow**: Use the `Timeline` component for micro-level vertical activity feeds.
- **Encapsulated Layering (Z-Index)**: Stacking layers are managed internally by `@bes/shared-ui` components (e.g., dynamic approval overlay modals use `9999`, `FloatingProcessPipeline` steppers use `9990`, sliding drawers use `9980`, and standard UI stays `<9980`). Custom modules MUST NOT hardcode or override `z-index` properties manually, ensuring uniform layer behavior.
- Pending/Draft actions must surface on the Home Dashboard.

## 3. UI Degradation, Subscription Controls & The Money Rule
- **Graceful Degradation**: Modules indicated as `READONLY_EXTENSIONS` from `/api/v1/bootstrap` must seamlessly degrade to read-only UI states without crashing.
- **Subscription Tier UI Control & Locks**:
  - Instead of completely hiding unlicensed premium modules or premium sub-features from users, render a premium lock indicator 🔒 next to their menu items, page tabs, or button actions to present a premium trials experience and drive package upsells.
  - Clicking locked features must open an attractive, premium-styled "Upgrade Plan" modal/overlay card listing package options, rather than rendering an empty white page or crashing.
- **Money Rule**: Currency and high-precision numbers MUST use `decimal.js` or `big.js` (4 decimal places matching backend).

## 4. State Management
- **Global State**: Use **Zustand** for global state (auth, session, navigation), stored in `apps/shell/src/store/`. NEVER use React Context for global state.
- **Local State**: Module-specific state stays within the module library.

## 5. Security, API Integration & RBAC
- **API Proxy**: Use Vite proxy. Base URL is `/api/v1`. Include auth token from `useAuthStore` in headers.
- **RBAC & License Map**:
  - All permission checking uses `checkPermission()` from `@bes/shared-ui`. Format: `<module>:<resource>:<action>`.
  - The dynamic sidebar and routing elements are populated using the bootstrap licensing map (`/api/v1/bootstrap`). RBAC constraints check permissions *after* licensing gates are evaluated.
- **Dev-Only Features**: Gated behind `import.meta.env.DEV`.
- **Authentication Flow**: Login -> Stores tokens in localStorage -> Fetches `auth/me` -> Renders shell. Use refresh token on expiry.

## 6. Styling & Component Guidelines
- **Styling**: Use **vanilla CSS** with CSS custom properties (design tokens from `@bes/shared-ui`). NEVER use inline styles for reusable components. Primary color: `#162867` (`var(--ui-primary)`).
- **Components**: Functional components only. Use unique, descriptive `id` attributes for testing. Named exports from libraries.
- **TypeScript**: Strict mode enabled. No `any` types. Use `interface` for props, `type` for unions. Use path aliases (`@bes/finance`).
