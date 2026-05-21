---
name: 5b-backend-build
description: "IMPLEMENTATION STEP 2b: Work on backend design and development based on the reviewed plan."
---

# Skill: Backend Build
**Lifecycle Position: STEP 5b of 6 — API Implementation**

This skill executes the backend development by strictly following the `proposed-plan.md` implementation steps.

---

## Lifecycle Context
```
STEP 1: 1-plan-functionalities  →  1-functionalities.md
STEP 2: 2-plan-ui-ux            →  2-ui-ux-flow.md
STEP 3: 3-plan-backend-yaml     →  3-backend-plan.md
STEP 4: 4-plan-reviewer         →  proposed-plan.md & common-dependants.md
[YOU ARE HERE]
STEP 5: 5b-backend-build        →  CODE
STEP 6: 6-bug-fixing            →  BUG FIXES
```

---

## About the BES Architecture
- **Stack**: FastAPI (PDM Monorepo) backend, PostgreSQL.
- **Core as a Package**: The `bes-core` is the engine. Extensions import it.
- **Models**: Must inherit `BESBase`.
- **Standard**: Soft deletes (`is_deleted = True`), 4-decimal precision (`Numeric(20,4)`), `{status, data, metadata, error}` envelope.

---

## Instructions for the Assistant

When asked to build the backend:

1. Read `features-plan/<module-name>/<feature-name>/proposed-plan.md` and `3-backend-plan.md`.
2. Use Graphify (`graphify query`) to verify `bes-core` imports and existing master data schemas.
3. Execute the implementation steps defined in `proposed-plan.md` for the DB and API phases.
4. Create database models inheriting from `BESBase` and applying Hub-and-Spoke MDM rules.
5. Scaffold the FastAPI routers and endpoints.
6. Apply RBAC permissions and wire up Pub/Sub events.
7. Save workflow pipeline definition schemas in `extensions/<module_name>/<module_name>/process_definitions/<module_name>.json` to register them automatically with the core dynamic scanner.

---

### Execution Rules
- **Output**: Code changes in backend extension paths.
- **Next step**: Testing, or `5a-frontend-build` if not complete.
