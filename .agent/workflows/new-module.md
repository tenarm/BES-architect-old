# Workflow: Add a Complete New BES Module

This workflow covers the end-to-end process of adding a new BES module to the platform, from backend to frontend.

---

## Overview

Adding a new module (e.g., "Procurement") requires changes across 4 areas:
1. **Core Config** — Permissions schema, client license
2. **Backend Extension** — Python module with models, schemas, services, router, events, manifest
3. **Frontend Library** — React Nx lib with pages, components, ComponentRegistry registration
4. **Verification** — End-to-end testing

## Phase 1: Planning

### 1.1 Define the Module Scope
- List all entities (database tables) the module needs
- List all API endpoints
- Identify cross-module events (what this module emits/subscribes to)
- Map out the RBAC resources and actions

### 1.2 Update Core Permissions
Add the module to `bes-backend/core/core/admin_permissions.json`:
```json
{
  "<module_name>": {
    "<resource_1>": { "read": true, "write": true, "delete": true },
    "<resource_2>": { "read": true, "write": true, "delete": true }
  }
}
```

### 1.3 Update Client License
Add to `bes-backend/onboarded/<client>.json`:
```json
{
  "licensed_modules": [..., "<module_name>"]
}
```

## Phase 2: Backend Implementation

Follow the **[Create Extension](../skills/create-extension/SKILL.md)** skill:

1. Create directory structure
2. Create `pyproject.toml`
3. Create `models.py` (DB tables only)
4. Create `schemas.py` (API input/output)
5. Create `services.py` (business logic)
6. Create `router.py` (HTTP handlers with pagination)
7. Create `events.py` (subscribers and emitters)
8. Create `manifest.py` (formal contract)

### Backend Verification
```bash
cd bes-backend
pdm run uvicorn instances.acme_corp.acme_corp.main:app --reload --port 8000
# Verify: GET /health → module count increased
# Verify: GET /api/v1/<module>/entities → empty list (200 OK)
# Verify: POST /api/v1/<module>/entities → creates record
```

## Phase 3: Frontend Implementation

Follow the **[Create UI Module](../skills/create-ui-module/SKILL.md)** skill:

1. Generate Nx library
2. Update `tsconfig.base.json` with path alias
3. Create module entry point with `ComponentRegistry.register()`
4. Create main page component
5. Register in Shell's `main.tsx`
6. Add icons and display names to `app-config.tsx`

### Frontend Verification
```bash
cd bes-frontend
npm run dev
# Verify: Module appears in sidebar after login
# Verify: Clicking module renders the page
# Verify: Data loads from backend API
```

## Phase 4: Integration Testing

### Event Bus Wiring
If this module emits or subscribes to events:
1. Verify emitter fires on the appropriate action
2. Verify subscriber receives and processes the event
3. Test with the subscriber module both loaded and not loaded

### RBAC Verification
1. Login as `admin` → full access to new module
2. Login as `manager` → read/write, no delete
3. Login as `staff` → read-only

### Pagination Verification
1. Create >50 records
2. Verify list endpoint returns paginated results
3. Verify `metadata.total`, `metadata.page`, `metadata.total_pages`

## Checklist

### Backend
- [ ] `pyproject.toml` with `core` dependency
- [ ] `models.py` with BESBase inheritance and Decimal for money
- [ ] `schemas.py` with separate Create/Read classes
- [ ] `services.py` with business logic
- [ ] `router.py` with pagination on all list endpoints
- [ ] `events.py` with subscriber/emitter functions
- [ ] `manifest.py` implementing ExtensionManifest
- [ ] Module added to `admin_permissions.json`
- [ ] Module added to client's `licensed_modules`

### Frontend
- [ ] Nx library generated with `@bes/<module>` import path
- [ ] `tsconfig.base.json` path alias added
- [ ] Components registered in ComponentRegistry
- [ ] Module initialized in Shell's `main.tsx`
- [ ] Icons and names added to `app-config.tsx`
- [ ] `RESOURCE_NAMES` updated for sidebar sub-items

### Cross-cutting
- [ ] RBAC permissions defined and enforced
- [ ] Events wired for cross-module integration
- [ ] Pagination working on all list endpoints
- [ ] API responses use StandardResponse envelope
