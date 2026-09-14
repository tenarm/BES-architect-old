---
trigger: always_on
description: Post-build bug prevention guidelines and registry keys to avoid regressions during backend, frontend, database, or config development.
---

# 5. Build Bug Prevention Rules — BES

These rules are compiled post-build prevention rules that all agents MUST read and strictly adhere to during development (backend, frontend, config, database) to avoid recurring bugs and regressions.

## 1. Process Definition Files (Config/Backend)
- **[Config] Envelope Wrapping**: Always wrap process definition JSON files under `process_definitions/` in a parent dictionary mapping the process ID to the actual process definition object (e.g., `{ "my_process_id": { "processId": "my_process_id", ... } }`). The platform's dynamic process scanner (`core/core/processes.py`) loads JSON files and runs `merged_defs.update(data)` expecting a keyed dictionary, not a bare process definition object.
- **[Config] Process Step Validation**: Every step in a process definition JSON must adhere strictly to the `ProcessStep` Pydantic model. Steps must define `id` (not `stepId`), `type` (e.g. `direct`, `approval`, `sequence`), and `statusEvent`. Missing these fields causes a schema validation failure and crashes processes API routes with HTTP 500.
- **[Config] Conform to Process Definition Schemas**: Every process definition JSON file under `process_definitions/` must adhere strictly to the `ProcessDefinition` Pydantic specifications: use `"label"` (not `"name"`), define `"entity"`, and ensure all steps utilize `"label"` and `"statusEvent"` to avoid runtime validation crashes.

## 2. ComponentRegistry Route Keys (Frontend)
- **[Frontend] Display-Name Key Matching**: Always register sub-item route components using `` `Route_${RESOURCE_NAMES['your_resource_key']}` `` as the exact `ComponentRegistry.registerLazy` key — never a hand-crafted PascalCase alias (e.g., `Route_CompanySetup`). The shell builds the lookup key as `` `Route_${activeItem}` `` where `activeItem` is the sidebar display name sourced from `RESOURCE_NAMES`. Any mismatch silently falls through to the "coming soon" placeholder. Reference `ROUTE_KEYS` in `constants.ts` to cross-check registered keys.
- **[Frontend] Template Literal Formatting**: Never use Python-style formatting specifiers (such as `:02d`, `%s`, `{:.2f}`) inside JavaScript/TypeScript template literals. Use `.padStart(2, '0')` or `.toFixed()` to format strings, numbers, or dates, avoiding compiler parse failures.
- **[Frontend] localStorage Token Key**: Always read the JWT access token from `localStorage.getItem('bes_token')`. The canonical key is `'bes_token'` as set by `auth-store.ts`. Never invent or guess an alternative key (e.g., `'bes_access_token'`) — a wrong key returns `null`, producing `Authorization: Bearer ` (empty) which the backend rejects with 401.
- **[Frontend] Sidebar Module Gating & Initializers**: When introducing a new frontend module library, ensure the module key is added to `allModulesList` in `use-shell.ts` AND its dynamic initializer function (e.g., `initSupplyChainModule()`) is uncommented/added inside `initializeModules` in `auth-store.ts`. Failing to do either will cause the module to be completely invisible in the sidebar menu.

## 3. Core-Extension Service Structuring (Backend)
- **[Backend] Preserve Boilerplate Imports**: Before fully overwriting services or repositories in a boilerplate extension, verify if the router or other components import any legacy/dummy logic, and ensure those imports and functions are preserved or safely deprecated to prevent `ImportErrors` during application boot.

## 4. Flow Architecture (Frontend)
- **[Frontend] Flow vs Data Hub Routing**: Flow pages register as `Flow_<FlowId>` and Data Hub pages as `DataHub_<Entity>` in the ComponentRegistry. Never mix these naming conventions — the shell dispatches based on these prefixes.
- **[Frontend] Design Token Usage**: ALWAYS use `--wp-*` CSS custom properties from the Warm Professional design system. NEVER hardcode hex colors, font names, or shadow values in component code. The old `--ui-*` tokens are deprecated.
- **[Frontend] FlowStepper State**: The FlowStepper component tracks step states (pending, active, completed, skipped, failed). Never manipulate DOM classes directly — always update via the FlowStepper's state prop.


