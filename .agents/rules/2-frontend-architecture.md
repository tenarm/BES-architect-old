---
trigger: always_on
description: Frontend architecture, Warm Professional design system, flow-centric navigation, Data Hub patterns, core component requirements, and UX laws for all React/TypeScript files in the bes-frontend/ Nx monorepo.
---

# 2. Frontend Architecture & Design Rules — TenArm

These rules govern all React/TypeScript code in the `bes-frontend/` Nx monorepo.

## 1. Monorepo Structure & Component Registry
- **Shell App**: `apps/shell/` is the entry point. It handles auth, navigation, sidebar, and route dispatch. It MUST NOT contain flow-specific or data-hub-specific business UI.
- **Library Organization**:
  - `libs/shared-ui/` (`@tenarm/shared-ui`) — Design tokens, base components, registry, utilities.
  - `libs/flows/<flow-id>/` — Flow-specific page components (landing, step views).
  - `libs/data-hub/<entity>/` — Data Hub entity pages (grid + detail panel).
  - `libs/settings/` — Settings and pipeline editor.
  - `libs/dashboard/` — Home dashboard.
- **ComponentRegistry**: Flows and Data Hub pages dynamically register their components in the Shell via `ComponentRegistry.registerLazy()` during module initialization.
  - Flow pages register as: `Flow_<FlowId>` (e.g., `Flow_Sell`, `Flow_Buy`)
  - Data Hub pages register as: `DataHub_<Entity>` (e.g., `DataHub_Customers`, `DataHub_Products`)
  - Dashboard widgets register as: `Widget_<Name>` (e.g., `Widget_SellMetrics`)

## 2. Warm Professional Design System

TenArm's visual identity is **Warm Professional** — inspired by Stripe Dashboard, Notion, and Linear. Premium feel through warmth, not coldness.

### Design Tokens (Mandatory)
All components MUST use CSS custom properties. Hard-coded hex values are FORBIDDEN in component code.

```css
/* Brand */
--wp-primary: #1a1a2e;              /* Deep ink */
--wp-accent: #e07a5f;               /* Terracotta */
--wp-accent-muted: rgba(224, 122, 95, 0.1);

/* Warm Neutrals (stone, not gray) */
--wp-stone-50 through --wp-stone-900

/* Surfaces */
--wp-surface-base: #fafaf9;         /* Warm white */
--wp-surface-card: #ffffff;
--wp-surface-sidebar: #fafaf9;

/* Typography */
--wp-font-display: 'Outfit', system-ui;      /* Headings */
--wp-font-body: 'Inter', system-ui;          /* Body */

/* Motion */
--wp-motion-enter: 200ms cubic-bezier(0.16, 1, 0.3, 1);
--wp-motion-exit: 150ms cubic-bezier(0.4, 0, 1, 1);
--wp-motion-spring: 400ms cubic-bezier(0.34, 1.56, 0.64, 1);
```

### Design Identity Rules
- **Never cold**: Use stone/warm-gray scale, not blue-gray/slate. Background is warm white `#fafaf9`, not cold `#f8fafc`.
- **Accent sparingly**: Terracotta (`--wp-accent`) for interactive elements, focus rings, active states, and primary CTAs. Not everywhere.
- **Outfit for display**: Headings, flow names, metric labels use the `Outfit` font. Body text uses `Inter`.
- **Warm shadows**: Shadow colors use warm tint `rgba(28, 25, 23, ...)`, not cold blue.
- **Focus glow**: Focus rings use terracotta glow (`--wp-shadow-glow`), not generic browser blue.
- **Motion is purposeful**: Use exactly 3 motion primitives (enter, exit, spring). No arbitrary `transition: all 0.3s`.

## 3. Flow-Centric Navigation (see Rule 3 for details)
- **Sidebar**: Home + My Tasks at top, then three sections — WORKFLOWS, DATA HUB, SYSTEM. No nested sub-items.
- **My Tasks**: Cross-flow inbox showing pending tasks for the current user. Badge count on sidebar item. Clicking a task navigates to the flow step view.
- **Flow Landing Pages**: Every flow entry point shows pending tasks for this flow + recent activity + "Start New [Flow]" CTA + KPI metrics.
- **Flow Step Views**: Horizontal FlowStepper progress bar + step content area + Previous/Next navigation.
- **Data Hub Pages**: DataTable + DetailPanel pattern. Consistent across all entities.
- **Encapsulated Layering (Z-Index)**: Stacking layers managed by `@tenarm/shared-ui`. Custom code MUST NOT hardcode z-index:
  - **Upgrade Modals / Command Palette**: `9999`
  - **Flow Pipeline Overlay**: `9990`
  - **Detail Panels / Drawers**: `9980`
  - **Toasts / Notifications**: `9970`
  - **Base UI Elements**: `<9970`

## 4. Core Component Requirements

Every TenArm build MUST use these shared components. Building one-off alternatives is FORBIDDEN.

| Component | Location | Purpose |
|:---|:---|:---|
| **DataTable** | `@tenarm/shared-ui` | Primary data grid. Sort, filter, search, paginate, row selection, keyboard nav. Every Data Hub page uses this. |
| **DetailPanel** | `@tenarm/shared-ui` | Slide-over panel for entity details. Tabs: Overview, Transactions, Timeline. Every row-click uses this. |
| **FlowStepper** | `@tenarm/shared-ui` | Horizontal progress bar showing flow steps. Visual states: pending, active, completed, skipped, failed. |
| **FormSection** | `@tenarm/shared-ui` | Declarative form groups with labels, validation, error messages, and progressive disclosure. Max 5-7 fields per section. |
| **CommandPalette** | `@tenarm/shared-ui` | Cmd+K global search. Searches flows, entities, settings. Keyboard-driven. |
| **StatusChip** | `@tenarm/shared-ui` | Visual status badge. Predefined states: Draft, Active, Confirmed, Shipped, Completed, Overdue, On Hold, Cancelled. |
| **MetricCard** | `@tenarm/shared-ui` | Dashboard KPI card: value, label, trend indicator, optional sparkline. |
| **Toast** | `@tenarm/shared-ui` | Notification toasts. Success, error, warning, info. Auto-dismiss. Stacks. |
| **EmptyState** | `@tenarm/shared-ui` | Illustrated empty state for pages with no data. Includes a CTA to create the first record. |
| **TaskCard** | `@tenarm/shared-ui` | Task inbox card showing: flow name, step, entity reference, assigned time, priority badge. Click navigates to step view. |
| **CommentThread** | `@tenarm/shared-ui` | Threaded comments with @mentions, internal/external toggle, edit history. Used in DetailPanel and step views. |
| **FileUpload** | `@tenarm/shared-ui` | Drag-and-drop file upload zone with progress, preview, and category tagging. Used in flow steps and DetailPanel. |
| **BulkActionBar** | `@tenarm/shared-ui` | Floating action bar shown when DataTable rows are selected. Shows count + available bulk actions. |
| **UpgradeGateOverlay** | `@tenarm/shared-ui` | Premium upsell overlay for locked flows/features. Tier comparison + upgrade CTA. |
| **PipelineEditor** | `@tenarm/shared-ui` | Drag-and-drop visual editor for flow pipeline customization. Used in Settings → Pipelines. |

## 5. UI Degradation & Subscription Controls
- **Graceful Degradation**: Unlicensed flows degrade to locked state with 🔒 indicator, not hidden.
- **Upgrade Gate**: Clicking a locked flow opens `UpgradeGateOverlay` — never a blank page or crash.
- **Read-Only Mode**: Flows in read-only mode disable form inputs and hide action buttons while preserving data visibility.
- **Money Rule**: Currency and high-precision numbers MUST use `decimal.js` (4 decimal places matching backend). NEVER native floats.

## 6. Print, Export & Document Generation
- **Print Action**: Every entity detail view MUST have a "Print" action that generates a clean, print-optimized layout via `@media print` CSS or a server-rendered PDF.
- **PDF Export**: Business documents (invoices, POs, quotes, delivery notes) MUST support PDF export. PDFs are generated server-side via a template engine and returned as downloadable attachments.
- **Email Action**: Documents that are sent externally (POs to suppliers, invoices to customers) MUST have a "Send via Email" action that attaches the generated PDF.
- **Batch Export**: When bulk rows are selected in a DataTable, an "Export" action generates a combined PDF or CSV download.
- **Template System**: PDF templates are per-entity-type and customizable in Settings. Templates use the tenant's logo, colors, and address from Company Profile.

## 7. Auto-Save & Draft Behavior
- **Auto-Save**: All form views in flow steps auto-save to the backend every 30 seconds (debounced). Visual indicator: small "Saved ✓" text near the form header that fades after 2 seconds.
- **Draft Recovery**: On page load, check for unsaved drafts. If found, show a subtle banner: "You have an unsaved draft from [date]. [Resume] [Discard]"
- **Offline Resilience**: If auto-save fails (network error), queue saves locally in `localStorage` and retry on reconnection. Show a warning Toast: "Changes saved locally — will sync when online."

## 8. State Management
- **Global State**: Use **Zustand** for global state (auth, session, active flow, navigation), stored in `apps/shell/src/store/`. NEVER use React Context for global state.
- **Flow State**: Each active flow instance maintains its own state (current step, form data, entity IDs) in a Zustand store within the flow library.
- **Data Hub State**: Entity list filters, pagination, and selected entity are local state within the Data Hub library.

## 9. Security, API Integration & RBAC
- **API Proxy**: Use Vite proxy. Base URL is `/api/v1`. Include auth token from `useAuthStore` in headers.
- **Token Key**: ALWAYS read JWT from `localStorage.getItem('bes_token')`. The canonical key is `'bes_token'`.
- **RBAC**: All permission checking uses `checkPermission()` from `@tenarm/shared-ui`. Format: `<module>:<resource>:<action>`.
- **Flow Permissions**: A flow step checks permissions against its `permissions` array in the flow definition. If any permission fails, the step renders as disabled with an explanation.
- **Dev-Only Features**: Gated behind `import.meta.env.DEV`.

## 10. Styling & TypeScript Standards
- **Styling**: Use **vanilla CSS modules** (`*.module.css`) with CSS custom properties from the design system. NEVER use inline styles for reusable components. NEVER use Tailwind.
- **Components**: Functional components only. Named exports. Unique `id` attributes for testing.
- **TypeScript**: Strict mode. No `any`. Use `interface` for props, `type` for unions. Path aliases (`@tenarm/shared-ui`, `@tenarm/flows-sell`).

## 11. UX Laws & Cognitive Load
- **Flow-Centric Mental Model**: Users think in workflows ("I want to sell"), not in modules ("I need the Sales module"). The sidebar, navigation, and page structure must reinforce this mental model.
- **Miller's Law**: Max 5-7 fields per form section. Use progressive disclosure for advanced options.
- **Fitts's Law**: Primary CTAs are large, prominent, and close to the user's focus area. "Next Step →" is always bottom-right.
- **Error-Forgiving Design**: Inline validation on blur. Friendly error messages. Undo support where possible.
- **Density Over Whitespace**: Data tables are compact. Minimal padding. Tag chips for status. Enterprise users need to see data, not empty space.
- **Persistent Context**: DetailPanel stays open alongside DataTable. Flow progress bar stays visible while working on steps. Never force context switches.

## 12. Notification Integration
- **Real-Time Bell**: Shell header notification bell subscribed to `/api/v1/notifications/stream` SSE.
- **Flow-Aware Routing**: Notification payloads include `flow_id` and `entity_id`. Clicking a notification navigates to the flow step or Data Hub entity.
- **Micro-Interactions**: Bell vibration on new notification. Badge pulse on unread count increment.

## 13. Error Boundaries
- **Flow-Level Boundaries**: Every flow library MUST wrap its root in an `ErrorBoundary`.
- **Graceful Fallback**: User-friendly fallback with "Retry" option. NEVER blank screen or raw stack trace.
- **Error Reporting**: Log error with flow ID, step ID, and route context.

## 14. Accessibility (a11y)
- **Keyboard Navigation**: All interactive elements operable via keyboard (Tab, Enter, Escape, Arrow keys).
- **Labels**: All form inputs MUST have `<label>` or `aria-label`.
- **Color Contrast**: WCAG 2.1 AA (4.5:1 normal text, 3:1 large text).
- **Focus Management**: Manage focus on panel/modal open/close and flow step transitions.
- **Live Regions**: Use `aria-live` for dynamic status changes (flow transitions, validation, loading).

## 15. Beginner-Friendly Development
- **Clear File Layout**: Group components within flow/data-hub subdirectories (components/, hooks/, stores/).
- **Simple Abstractions**: No over-complex generics or deep component nesting. Readable code with inline comments.
- **Showcase App**: Build shared UI components in isolation in `apps/showcase/` for visual documentation.
