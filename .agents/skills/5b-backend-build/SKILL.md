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

1. Read the global bug registry [features-plan/build-bugs.md](file:///Users/bvk/BVK_Workspace/BES/features-plan/build-bugs.md) and the compiled prevention rules [.agents/rules/5-build-bug-prevention-rules.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/5-build-bug-prevention-rules.md). Identify any historical backend, database, event, or config issues relevant to your module, and note them as active quality constraints in your checklist.
2. Read `features-plan/<module-name>/<feature-name>/proposed-plan.md` and `3-backend-plan.md`.
3. Use Graphify (`graphify query`) to verify `bes-core` imports and existing master data schemas.
4. Execute the implementation steps defined in `proposed-plan.md` for the DB and API phases.
4. Create database models inheriting from `BESBase` and applying Hub-and-Spoke MDM rules.
5. Scaffold the FastAPI routers and endpoints. If the extension module is large or contains multiple distinct feature areas (e.g. `settings`), modularize it into directory packages (`models/`, `schemas/`, `services/`, `router/`) instead of flat files, using `__init__.py` files to re-export the contents.
6. Apply RBAC permissions and wire up Pub/Sub events.
7. Save workflow pipeline definition schemas in `extensions/<module_name>/<module_name>/process_definitions/<module_name>.json` to register them automatically with the core dynamic scanner.
8. Enforce all backend architectural and security rules from `.agents/rules/1-backend-architecture.md` (Beginner-Friendly & Self-Documenting Design, Error-Forgiving API Design, soft-deletes).
9. Keep backend code beginner-friendly: write explicit docstrings, use typed variables, avoid overly complex SQLAlchemy operations, and structure services/repositories clearly with detailed inline comments.


---

### Execution Rules
- **Output**: Code changes in backend extension paths.
- **Next step**: Testing, or `5a-frontend-build` if not complete.
