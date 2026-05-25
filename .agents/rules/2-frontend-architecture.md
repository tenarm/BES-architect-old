---
trigger: always_on
description: Frontend architecture, modular component registration, z-index matrices, CSS/TS standards, UI graceful degradation, and cognitive UX laws for all React/TypeScript files in the bes-frontend/ Nx monorepo.
---

# 2. Frontend Architecture & Security Rules — BES

These rules govern all React/TypeScript code in the `bes-frontend/` Nx monorepo.

## 1. Monorepo Structure & Component Registry
- **Shell App**: `apps/shell/` is the entry point. It MUST NOT contain module-specific business UI.
- **Module Libraries**: Each module's UI lives in `libs/<module>/`. 
- **Shared UI**: Base components and design tokens MUST come from `libs/shared-ui/` (`@bes/shared-ui`).
- **ComponentRegistry**: Modules dynamically register their components in the Shell via `ComponentRegistry.register()` (e.g., `Route_FinanceMain`, `Widget_FinanceSummary`) during `init<Module>Module()`.

## 2. Process Transparency & Workflow
- **Encapsulated Layering (Z-Index)**: Stacking layers are managed internally by `@bes/shared-ui` components. Custom modules MUST NOT hardcode or override `z-index` properties manually, ensuring uniform layer behavior. Standard stacking indices:
  - **Dynamic Approval / Upgrade Modals**: `9999`
  - **Floating Process Pipeline**: `9990`
  - **AI Floating Window**: `9985`
  - **AI Capsule Button**: `9984`
  - **Sliding Drawers**: `9980`
  - **Base UI Elements**: `<9980`
- **Macro Process Pipeline (`FloatingProcessPipeline`)**: Glassmorphic vertical stepper overlay loaded dynamically on demand. It is cached in `sessionStorage` (keyed by process ID) to enable immediate first paint and avoid layout shifting. It is hydrated live via Server-Sent Events (SSE) `/api/v1/notifications/stream` connection. Visual states must progress through: *Pending*, *Active*, *Waiting Approval*, *Complete*, and *Failed*.
- **Micro Historical Logs (`Timeline`)**: A detailed chronological history feed rendered under a secondary "History" tab inside the side drawers or panels to preserve context.
- **Approval Registry Modals**: Custom verification, override, or approval forms must be registered dynamically in the central `ComponentRegistry` to bind onto specific process pipeline steps.
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

## 7. UX Laws & Cognitive Load Reductions
All frontend designs and layouts must apply these behavioral psychology principles:
- **System-Directed Mental Models**: Standardize navigation, status badges, forms, and workflows across all modules. Keep layouts uniform so a user who learns one module (e.g. Finance) instantly understands how to operate others (e.g. Sales).
- **Miller’s Law (Information Chunking)**: Do not overwhelm the user. Chunk forms and metadata into logically clustered groupings or progressive wizard steps of 5-7 elements maximum. Use clean section separators, tabs, or headers.
- **Fitts's Law for Data Entry**: Primary buttons, input toggles, and dropdown fields must have generous click/tap targets. Position primary action controls (e.g., Save, Submit) predictably and in close physical proximity to the final input fields to minimize cursor travel distance.
- **Error-Forgiving Design**: Protect users from mistakes. Provide real-time inline input validation, explicit helpers, warning indicators, and undo operations. When actions fail, present friendly explanations showing how to fix it rather than showing a generic error code.
- **Aesthetic-Usability Effect**: Deliver visually premium, polished interfaces (e.g. cohesive dark/light palettes, Outfit/Inter typography, subtle glassmorphism cards, and smooth micro-animations like bell vibration or badge pulsing). Users perceive beautiful interfaces as more usable and trustworthy.
- **Density Over Whitespace**: Enterprise software requires data density. Maximize grid visibility and tabular presentation with compact cell paddings, short row heights, and tag chips. Avoid excessive empty space that forces unnecessary scrolling.
- **Persistent Context**: Never force users to memorize information or jump screens to view details. Use split-screen side drawers, flyout panels, or side-by-side preview panes to show record details, activity logs, or approval steppers alongside the main data grid.

## 8. Beginner-Friendly Frontend Development
- **Clear File Layout**: Group components logically within module subdirectories (e.g. components, hooks, stores).
- **Simple Abstractions**: Avoid over-complex TypeScript generics, custom hooks wrapper chains, or deep nesting of components. Write straightforward functional components with readable variables and inline comments explaining state flows.
- **Storybook / Isolated Testing**: Build shared UI components in isolation to let beginner developer peers view and understand usage examples without launching the full backend server.

## 9. Notification Integration
- **Real-Time Notification Bell**: The global shell header must include an active Notification Bell component. It subscribes to `/api/v1/notifications/stream` over Server-Sent Events (SSE) using the user's active JWT bearer token. It manages a local list of recent notifications and an unread count.
- **Micro-interactions & UX Feedback**: Apply the Aesthetic-Usability Effect with subtle animations. When a new notification arrives via SSE:
  - The bell icon must perform a subtle vibration/shake animation.
  - The unread badge counter must pulse and increment dynamically.
- **Metadata-Driven Redirection (Dynamic Routing)**: Notification payloads should include routing metadata (e.g. `{"target_route": "/sales/invoices", "entity_id": "123-456"}`). Clicking a notification must redirect the user to the route and automatically open the detail drawer or flyout panel for the specific entity, maintaining persistent context.
- **Settings & Channel Licensing Lock**: In the Notification Rules configuration page:
  - Basic-tier tenants cannot activate premium channels (e.g., `EMAIL`).
  - Next to premium channel toggle options, render a lock icon `🔒`.
  - Clicking a locked notification channel option must launch the central `UpgradeGateOverlay` listing packages instead of failing or displaying a blank screen.


