---
name: 5-build-flow
description: "BUILD STEP: Execute the approved proposed-plan.md — build both backend and frontend for a TenArm flow end-to-end."
---

# Skill: Build Flow

## Purpose
Given an approved `proposed-plan.md`, build the complete flow — backend models, services, APIs AND frontend pages, components, styles — in one vertical pass.

## When to Use
- After `4-review-flow` has produced an approved `proposed-plan.md`
- The user has reviewed and approved the plan
- Run FIFTH and FINAL in the planning pipeline

## Prerequisites
- The flow's `proposed-plan.md` must be approved by the user.
- Read Rule 5 (Build Bug Prevention) before writing any code.
- Read the flow's complete planning docs (1-flow-definition, 2-flow-ui-design, 3-flow-backend-plan).
- Check that all build prerequisites from the proposed plan are satisfied.

---

## Execution Steps

### Step 0: Pre-Build Checklist

Before writing any code, verify:

- [ ] Read Rule 5 (Build Bug Prevention) — every item
- [ ] All prerequisites from proposed-plan.md are built or stubbed
- [ ] The backend extension directory exists (or create it via scaffold-module skill)
- [ ] The frontend flow library exists (or create it)
- [ ] Design tokens are available in shared-ui (`--wp-*` tokens)

### Step 1: Build Backend — Models

Create/update model files in `bes-backend/extensions/<module>/<module>/models.py` (or `models/` directory):

**Rules**:
- Inherit from `BESBase`
- Financial fields: `sa_column=Column(Numeric(precision=20, scale=4))`
- Status field with defined valid values
- `__tablename__` follows `<module>_<entity>s` convention
- Import ordering: stdlib → third-party → core → current module

### Step 2: Build Backend — Schemas

Create `schemas.py` with `*Create` and `*Read` Pydantic models:

**Rules**:
- Never use ORM models as API input
- `*Read` includes all display fields + computed fields
- Financial amounts serialized as string for JSON safety
- Use `Field(description=...)` for documentation

### Step 3: Build Backend — Services

Create `services.py` with all business logic:

**Rules**:
- Define domain exceptions (e.g., `InvalidTransitionError`), NOT `HTTPException`
- Services own transaction boundaries — only services call `session.commit()`
- Financial calculations use Python `Decimal` with max precision
- Quantize at document boundary with `ROUND_HALF_UP`
- State machine transitions checked against valid transition matrix
- Events collected in memory, dispatched AFTER `session.commit()`
- Service hierarchy: Leaf Services (single entity CRUD) → Composite Services (orchestration)

### Step 4: Build Backend — Router

Create `router.py` as a thin HTTP layer:

**Rules**:
- Catch domain exceptions → translate to HTTP status codes
- Use `require_permission()` for sensitive endpoints
- All responses wrapped in `StandardResponse`
- List endpoints use `PaginationParams`
- Route prefix: `/api/v1/<module>`

### Step 5: Build Backend — Events

Create/update `events.py`:

**Rules**:
- Event names follow `{FLOW}_{STEP}_{ACTION}` format
- Subscribers handle events idempotently
- Failed subscribers don't crash the emitter

### Step 6: Build Backend — Tests

Create test files in `tests/`:

**Rules**:
- Test file: `test_<entity>.py`
- Test function: `test_<action>_<scenario>`
- Financial tests assert 4-decimal precision
- Use in-memory SQLite for isolation
- Test happy path + error paths + state machine transitions

### Step 7: Build Frontend — Flow Library

Create the flow library in `bes-frontend/libs/flows/<flow-id>/`:

**Directory structure**:
```
libs/flows/<flow-id>/
├── src/
│   ├── index.ts                  ← Exports + ComponentRegistry registration
│   ├── lib/
│   │   ├── <flow>-landing.tsx    ← Flow landing page
│   │   ├── <flow>-landing.module.css
│   │   ├── steps/
│   │   │   ├── <step-id>.tsx     ← One component per step
│   │   │   ├── <step-id>.module.css
│   │   │   └── ...
│   │   ├── hooks/
│   │   │   └── use-<flow>.ts     ← Flow state management (Zustand)
│   │   └── types.ts              ← TypeScript interfaces
│   └── test-setup.ts
├── project.json
├── tsconfig.json
├── tsconfig.lib.json
└── vite.config.ts
```

**Registration in index.ts**:
```typescript
import { ComponentRegistry } from '@tenarm/shared-ui';

export function initSellFlow() {
  ComponentRegistry.registerLazy('Flow_Sell', () =>
    import('./lib/sell-landing').then(m => ({ default: m.SellLanding }))
  );
}
```

### Step 8: Build Frontend — Flow Landing Page

Build the landing page with:
- MetricCards row (KPIs from proposed plan)
- DataTable for recent activity
- "Start New [Flow]" CTA button
- Quick status filters

**Rules**:
- Use `--wp-*` design tokens — no hardcoded colors
- CSS modules for styling
- Components from `@tenarm/shared-ui`
- Zustand store for flow state

### Step 9: Build Frontend — Step Views

Build each step's view component:
- FlowStepper at the top showing pipeline progress
- FormSection components for data entry steps
- DataTable for line items
- Previous/Next navigation buttons
- Inline validation on blur
- Loading states with skeleton components

### Step 10: Build Frontend — Data Hub Pages

Build Data Hub pages in `libs/data-hub/<entity>/`:
- DataTable with sortable/filterable columns
- Row click opens DetailPanel
- DetailPanel with tabs (Overview, Transactions, Timeline)
- Empty state with CTA to start the parent flow

### Step 11: Register in Shell

Update the shell to recognize the new flow:
1. Add flow to sidebar navigation (WORKFLOWS section)
2. Add Data Hub entities to sidebar (DATA HUB section)
3. Wire up the ComponentRegistry keys
4. Add to `allModulesList` in `use-shell.ts`
5. Add initializer call in `auth-store.ts`

### Step 12: Create Flow Definition JSON

Write the machine-readable flow definition to:
```
features-plan/flows/<flow_id>/flow-definition.json
```

Following the schema from Rule 3 §8.

### Step 13: Post-Build Verification

Run the verify-build skill or manually check:

**Backend**:
```bash
cd bes-backend
pdm run pytest extensions/<module>/tests/ -v
```

**Frontend**:
```bash
cd bes-frontend
npx nx lint flows-<flow-id>
npx nx build shell
```

**Integration**:
- Start backend + frontend
- Navigate to the flow in sidebar
- Complete the full flow happy path
- Check Data Hub views populated with flow data
- Verify error paths (invalid data, missing fields)

### Step 14: Finalize Cross-Flow Dependency Registry

Update `features-plan/common-dependants.md` with verified, concrete values from the built code:

1. **Review** the flow's existing section (created during Skill 1).
2. **Verify** that all listed dependencies are actually imported/used in the code.
3. **Update** outbound assets with:
   - Actual event names (from `events.py`)
   - Actual table names (from `models.py`)
   - Actual API endpoints exposed (from `router.py`)
4. **Add** any new dependencies discovered during the build that weren't in the original plan.
5. **Remove** any planned dependencies that turned out to be unnecessary.

This ensures the registry reflects the built reality, not just the plan.

---

## Output

Working code across:
- `bes-backend/extensions/<module>/` — models, schemas, services, router, events, tests
- `bes-frontend/libs/flows/<flow-id>/` — landing page, step views, hooks
- `bes-frontend/libs/data-hub/<entity>/` — Data Hub entity pages
- `bes-frontend/apps/shell/` — sidebar + registration updates
- `features-plan/flows/<flow-id>/flow-definition.json` — pipeline config
- `features-plan/common-dependants.md` — updated with verified dependencies

## Quality Checklist
- [ ] Rule 5 (Build Bug Prevention) read and followed
- [ ] All backend models inherit BESBase with proper fields
- [ ] All services use domain exceptions, not HTTPException
- [ ] All financial calculations use Decimal
- [ ] All frontend components use `--wp-*` tokens via CSS modules
- [ ] ComponentRegistry keys follow `Flow_*` / `DataHub_*` convention
- [ ] Shell sidebar updated with new flow + Data Hub entries
- [ ] Flow landing page has metrics, activity feed, CTA
- [ ] Every step view has FlowStepper, form, and navigation
- [ ] Data Hub pages have DataTable + DetailPanel
- [ ] Backend tests pass
- [ ] Frontend lints and builds without errors
- [ ] Happy path works end-to-end in browser
- [ ] `common-dependants.md` finalized with verified event names, table names, and API endpoints
