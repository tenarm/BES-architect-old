# Flow & Module Build Checklist — BES Quick Reference

Use this checklist when building a new flow or backend module feature. Each item maps to a specific rule or skill.

---

## Planning Phase *(Rule 4)*

- [ ] Flow definition defined → `1-flow-definition.md` *(Skill: 1-plan-flow)*
- [ ] UI/UX flow designed → `2-flow-ui-design.md` *(Skill: 2-design-flow-ui)*
- [ ] Backend schema + APIs specified → `3-flow-backend-plan.md` *(Skill: 3-plan-flow-backend)*
- [ ] Architectural review passed → `proposed-plan.md` *(Skill: 4-review-flow)*
- [ ] Graphify queried to verify existing shared UI components or core APIs
- [ ] Cross-flow dependencies documented → `common-dependants.md` *(Rule 4 §6)*

---

## Backend Build

### Models *(Rule 1 §2-3)*
- [ ] All tables inherit from `BESBase`
- [ ] Money fields use `Numeric(20,4)` / `Decimal` — NEVER `float`
- [ ] Soft deletes: `is_deleted` filtering on all queries
- [ ] Multi-tenancy: `subsidiary_id` scoping
- [ ] Optimistic locking: `version_id` passed on updates

### Schemas *(Rule 1 §2)*
- [ ] Separate `*Create` and `*Read` Pydantic schemas
- [ ] NEVER use ORM model as API input

### Services *(Rule 1 §14)*
- [ ] Service owns transaction boundary (`session.commit()`)
- [ ] Domain exceptions only — NOT `HTTPException`
- [ ] Events emitted AFTER commit, not before
- [ ] `Decimal` math with `ROUND_HALF_UP` at final stage

### Router *(Rule 1 §2, §4)*
- [ ] Thin HTTP layer — NO business logic
- [ ] `StandardResponse` envelope on all responses
- [ ] Pagination on list endpoints (`PaginationParams`)
- [ ] RBAC: `require_permission()` on write endpoints
- [ ] Licensing: `require_licensed_feature()` check

### Events & Flows *(Rule 1 §6-7, §13, Rule 3 §8)*
- [ ] Event bus for cross-module communication
- [ ] Flow definition JSON structured properly in `features-plan/flows/<flow_id>/flow-definition.json`
- [ ] For legacy processes: JSON wrapped in `{ "process_id": { ... } }` dict (Rule 5 §1)
- [ ] Steps use `id` (not `stepId`), `label` (not `name`)

### Manifest *(Rule 1 §2)*
- [ ] `manifest.py` exports `manifest` instance of `ExtensionManifest`

### Tests *(Rule 1 §17)*
- [ ] `tests/` directory with `conftest.py`
- [ ] Unit tests for service methods
- [ ] Integration tests for API routes
- [ ] Financial calc tests with 4-decimal assertions

---

## Frontend Build

### Library Setup *(Rule 2 §1, Rule 3)*
- [ ] Nx library created at `libs/flows/<flow-id>/` or `libs/data-hub/<entity>/`
- [ ] `ComponentRegistry.registerLazy` with correct keys (`Flow_<FlowId>` or `DataHub_<Entity>`)

### Shell Registration *(Rule 5 §2)*
- [ ] Module initializer added to `auth-store.ts` → `initializeModules()`
- [ ] Module key added to `allModulesList` in `use-shell.ts`
- [ ] Route keys accurately map to `RESOURCE_NAMES` if using legacy sidebar routes

### Components *(Rule 2 §2, §4, §6, Rule 5 §4)*
- [ ] Functional components with named exports
- [ ] Unique `id` attributes for testing
- [ ] CSS module files with custom properties (`--wp-*`) — NO inline styles
- [ ] No Tailwind — vanilla CSS only
- [ ] `decimal.js` for financial calculations
- [ ] Use shared UI components (DataTable, DetailPanel, FlowStepper, etc.)
- [ ] FlowStepper state updated via props, never DOM manipulation

### State *(Rule 2 §8)*
- [ ] Zustand for global state in `apps/shell/src/store/`
- [ ] Flow State in Zustand store within the flow library

### UX & Flows *(Rule 2 §7, §11, Rule 3)*
- [ ] Flow-centric navigation reinforced
- [ ] Miller's Law: 5-7 items per form section
- [ ] DetailPanel stays open alongside DataTable
- [ ] FlowStepper visually tracks flow progress
- [ ] Auto-save and draft behavior implemented

### Subscription Gates *(Rule 2 §5)*
- [ ] Lock indicators (🔒) for premium features
- [ ] `UpgradeGateOverlay` opens on locked feature click
- [ ] Read-only mode for flows if applicable

### Accessibility *(Rule 2 §14)*
- [ ] Keyboard navigable (Tab, Enter, Escape)
- [ ] All inputs have labels or aria-label
- [ ] Focus management on modal/drawer open/close
- [ ] Color contrast meets WCAG 2.1 AA (4.5:1)

### Error Boundaries *(Rule 2 §13)*
- [ ] Flow library root wrapped in `ErrorBoundary`
- [ ] Graceful fallback UI (not blank screen)

### Tests
- [ ] Vitest configured with shared-ui test-setup
- [ ] Placeholder unit tests passing
- [ ] `npx nx lint <library>` — no errors
- [ ] `npx nx build shell` — full build succeeds

---

## Integration

- [ ] API endpoints return `StandardResponse` envelope
- [ ] Pagination works on list endpoints
- [ ] CRUD operations work end-to-end
- [ ] Flow pipeline loads and displays correctly
- [ ] Notifications fire on key events
- [ ] `graphify update .` run to index changes

---

## Documentation

- [ ] `common-dependants.md` updated with inbound/outbound dependencies
- [ ] `build-bugs.md` updated if any bugs found during build
- [ ] `5-build-bug-prevention-rules.md` updated if systemic issues found
