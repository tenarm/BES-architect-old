# 5. Build Bug Prevention Rules — BES

These rules are compiled post-build prevention rules that all agents MUST read and strictly adhere to during development (backend, frontend, config, database) to avoid recurring bugs and regressions.

## 1. Process Definition Files (Config/Backend)
- **[Config] Envelope Wrapping**: Always wrap process definition JSON files under `process_definitions/` in a parent dictionary mapping the process ID to the actual process definition object (e.g., `{ "my_process_id": { "processId": "my_process_id", ... } }`). The platform's dynamic process scanner (`core/core/processes.py`) loads JSON files and runs `merged_defs.update(data)` expecting a keyed dictionary, not a bare process definition object.

## 2. ComponentRegistry Route Keys (Frontend)
- **[Frontend] Display-Name Key Matching**: Always register sub-item route components using `` `Route_${RESOURCE_NAMES['your_resource_key']}` `` as the exact `ComponentRegistry.registerLazy` key — never a hand-crafted PascalCase alias (e.g., `Route_CompanySetup`). The shell builds the lookup key as `` `Route_${activeItem}` `` where `activeItem` is the sidebar display name sourced from `RESOURCE_NAMES`. Any mismatch silently falls through to the "coming soon" placeholder. Reference `ROUTE_KEYS` in `constants.ts` to cross-check registered keys.
- **[Frontend] localStorage Token Key**: Always read the JWT access token from `localStorage.getItem('bes_token')`. The canonical key is `'bes_token'` as set by `auth-store.ts`. Never invent or guess an alternative key (e.g., `'bes_access_token'`) — a wrong key returns `null`, producing `Authorization: Bearer ` (empty) which the backend rejects with 401.
