# Modular BES Factory - Master Blueprint

This document bridges the gap between the "Raw Python" setup and "Global SaaS" scale. It defines the architectural rules, security boundaries, licensing models, and operational pipelines that govern the Business Execution System (BES).

---

## 1. High-Level System Architecture & Distribution

The system follows a strict **Kernel-and-Plugin (Core + Extension)** pattern. The Core acts as the "Operating System," while Extensions act as the "Applications."

```
                     ┌──────────────────────────────┐
                     │          Shell UI            │
                     │      (Vite Nx Monorepo)      │
                     └──────────────┬───────────────┘
                                    │
                                    ▼ (Dynamic Route & Component Registry)
 ┌──────────────────────────────────┴──────────────────────────────────┐
 │                          FastAPI Backend                            │
 │                                                                     │
 │   ┌─────────────────────────────────────────────────────────────┐   │
 │   │                         Core Kernel                         │   │
 │   │   - Database Pooling  - Auth & RBAC     - Event Bus         │   │
 │   │   - Base Models       - System Context  - Storage & UOM     │   │
 │   └──────────────────────────────┬──────────────────────────────┘   │
 │                                  │                                  │
 │         ┌────────────────────────┼────────────────────────┐         │
 │         ▼                        ▼                        ▼         │
 │   ┌───────────┐            ┌───────────┐            ┌───────────┐   │
 │   │ Extension │            │ Extension │            │ Instance  │   │
 │   │  (Sales)  │            │ (Finance) │            │ (Custom)  │   │
 │   └───────────┘            └───────────┘            └───────────┘   │
 └─────────────────────────────────────────────────────────────────────┘
```

### A. Core-as-a-Package Monorepo Simulation
While built inside a monorepo for maximum developer efficiency, the codebase is structurally segmented to simulate a **Core-as-a-Package** physical firewall:
- **The Engine (`bes-core`)**: Located under `bes-backend/core/`. It contains the central fastapi framework factory, DB pooling, authentication, RBAC logic, event bus, base model definitions, standard response builders, and middlewares. It MUST NEVER import from extensions or instance directories.
- **Extensions**: Business modules located under `bes-backend/extensions/<module_name>/`. They inherit schemas, bases, and utility services from `core` and expose standard business routes.
- **Client Instances**: Client-specific deployments located under `bes-backend/instances/<client_id>/`. They depend on `core` and their enabled extensions, adding client-specific overrides and models.

### B. Monorepo Directory & Workspace Layout

| Component | Physical Path | Workspace System | Responsibility |
| :--- | :--- | :--- | :--- |
| **Core Kernel** | `bes-backend/core/` | PDM Library Workspace | Fast API Bootstrap, Auth, DB Pooling, Event Bus, Base Models, Global Middlewares |
| **Standard Extensions** | `bes-backend/extensions/*` | PDM Module Workspaces | Domain-specific logic (HR, Sales, Finance), DB tables, APIs, and Event Subscribers |
| **Client Instances** | `bes-backend/instances/*` | PDM App Workspaces | Client-specific configurations, dynamic model provisioning, custom permissions, and unique business logic |
| **Shell UI** | `bes-frontend/apps/shell/` | Nx Monorepo App | User login, routing bootstrap, primary sidebar, global state (`Zustand`), and upgrades orchestration |
| **Module UI Libs** | `bes-frontend/libs/*` | Nx Module Libraries | Isolated frontend features that dynamically register UI views, reports, and controls in the Shell |
| **Shared UI** | `bes-frontend/libs/shared-ui/` | Nx Shared Library | Base design tokens, standard components (`ProcessPipeline`, `Timeline`), and `checkPermission` utility |

---

## 2. Extension Module Layering

Every extension module MUST strictly decouple concerns across the following structure:
- **`models.py`**: Declares database tables inheriting from `BESBase`.
- **`schemas.py`**: Declares Pydantic schemas for API payload validation (`*Create`, `*Read`). *ORM models must never be used directly as API parameters or returned unmapped.*
- **`services.py`**: Executes business rules, transaction orchestration, database queries, and inter-table transformations.
- **`router.py`**: Pure, thin HTTP layer that handles parsing parameters, checking permissions, and delegating to services. No business logic lives here.
- **`events.py`**: Subscribes to and emits events via the global asynchronous Event Bus.
- **`manifest.py`**: Exports an instance of `ExtensionManifest` to register metadata and license hooks.

---

## 3. The Data Foundation (SQLModel & PostgreSQL)

Every database table in the system is built on a unified foundation to ensure security, auditing, soft-deletes, and multi-tenant constraints.

### A. The Base Model (`BESBase`)
All database models inherit from `core.models.BESBase` which implements:
- **UUID Primary Keys**: Protects database tables from id enumeration attacks and guarantees merge safety.
- **Auditing Fields**: Standardized UTC fields: `created_at`, `updated_at`, and `created_by`.
- **Soft Delete Flag**: An `is_deleted` boolean flag. *Physical SQL deletion is strictly forbidden.* All query repositories automatically apply `is_deleted == False` filters.
- **Organizational Scoping**: A `subsidiary_id` foreign key.
- **Expansion Joint**: A `metadata_` JSONB column.
  - Used for "Ghost Foreign Keys" (referencing client-specific or third-party tables without hard SQL foreign key constraints).
  - Used for dynamic client-specific custom fields without altering the database schema.
- **Concurrency Tracking**: A `version_id` column used to track entity updates for optimistic locking.

### B. Hierarchical Multi-Tenancy
Multi-tenancy is enforced through a dual-isolation layer:
1. **Tenant Isolation**: A dedicated PostgreSQL database per client instance. No data is co-mingled at the database engine layer.
2. **Subsidiary Scoping**: A tenant-level division scoped by `subsidiary_id` on every table. A backend `ContextVar` populated by the `ContextAwareSecurityMiddleware` automatically applies the active `subsidiary_id` scope to all SELECT, UPDATE, and DELETE operations.

### C. Hub-and-Spoke Master Data Management (MDM)
Core master entities (such as `customers` and `vendors`) are maintained inside the Core database workspace. Extension modules reference these core master records using "Ghost Foreign Keys" mapped within their `metadata_` schemas rather than direct SQL level joins, preventing circular dependencies across extensions.

### D. Concurrency Control (Optimistic & Pessimistic Locking)
To handle concurrent edit conflicts ("lost updates") in collaborative multi-user environments:
1. **Optimistic Locking**:
   - `BESBase` includes a `version_id` column configured dynamically as `version_id_col` via SQLAlchemy `@declared_attr`.
   - On updates, `BaseRepository.update` checks that the client's expected version matches the DB version. A mismatch raises `ConcurrencyError`, which is handled globally to return `409 Conflict`.
2. **Pessimistic Locking**:
   - For high-contention operations (e.g. inventory deductions, ledger postings), services must use `BaseRepository.get_with_lock(session, id)` to lock rows with `FOR UPDATE`.

---


## 4. Subscription Packaging & Tier Licensing

BES implements a granular subscription and feature toggle system, enforcing plan boundaries across three preset tiers and customized modules.

### A. Tier Configuration Matrix (`packages.json`)
Subscription scopes are configured inside `core/core/packages.json`:
- **Basic Plan**: Unlocks standard `sales` actions (customer master, quotations, sales orders), `inventory` core functions, and primary `settings`.
- **Professional Plan**: Adds full financial ledgers (CoA, GL, AP, AR, taxes), advanced SCM (requisitions, purchase orders), HR (employee logs, attendance, Ess portal), CRM pipeline, and advanced settings (workflows, sequences).
- **Enterprise Premium Plan**: Grants unlimited access (`*`) to all available core and extension modules.
- **Custom Plan**: Allows provisioning a selected subset of modules tailored to unique client contracts.

### B. Backend Granular Verification
Feature limits are programmatically enforced at the endpoint and service level:
- **Routing Checks**: Route endpoints assert access using the `require_licensed_feature("module", "subfeature")` decorator. If unlicensed, the route raises a standard `403 Forbidden` response.
- **Service Verification**: Background queues, event handlers, and scheduled crons execute manual licensing checks against the active context. Unlicensed execution raises a `LicensingError`.
- **System Elevation**: System-triggered events, db migrations, and high-privilege automated crons can bypass checks using the system-elevation context context-manager:
  ```python
  with elevate_context():
      # Internal operations run without license validation
      await coa_service.generate_system_ledgers(db)
  ```

---

## 5. Client Customization & Instance Isolation

Standard extension packages and the Core engine are 100% client-agnostic. All client-specific business rules, custom reports, API endpoints, integrations, and database overrides MUST reside strictly within `instances/<client_id>/`.

```
bes-backend/instances/<client_id>/
├── pyproject.toml              # Declares the client chassis app and dependencies
├── Dockerfile                  # "Blender" fusing core, standard extensions, and overrides
├── config/
│   ├── onboard_config.json      # Holds metadata of licensed modules
│   ├── admin_permissions.json   # Computed active permissions mapping for the client
│   └── custom_permissions.json  # Client-specific override permissions
└── <client_id>/
    ├── __init__.py
    ├── main.py                 # Startup file importing core and mounting custom routers
    ├── lifespan.py             # Client lifespan hooks (dynamic model and event registration)
    └── models/                 # Eagerly registered database overrides
```

### A. Operations & Setup Automation

#### 1. Client Onboarding Pipeline (`scripts/onboard_client.py`)
Clients are provisioned using a robust interactive setup pipeline:
1. **Interactive CLI Prompts**: Collects `client_id`, `client_name`, custom database URLs, and subscription tiers.
2. **Config Generation**: Generates `onboard_config.json` detailing licensed scopes.
3. **Editable Local Workspace Linking**: Compiles the client’s private `pyproject.toml`, resolving standard extensions as workspace PDM dependencies.
4. **Boilerplate Instantiation**: Copies standard client chassis boilerplates, injecting localized imports and system paths.
5. **Auto-Build Hook**: Executes `pdm build` to generate localized packages and `.egg-info` directories.

#### 2. Dynamic Permission Merging (`scripts/custom_feature_config.py`)
Client-specific modules can register custom RBAC actions using the permission builder:
- Private custom configurations are defined in `instances/<client_id>/config/custom_permissions.json`.
- The onboarding script automatically merges these custom rules into the computed `admin_permissions.json` file.

### B. Dynamic Bootstrap & Lifespan Binding
To load client-specific components without modifying the core, the client chassis utilizes Python lifespan hooks inside `instances/<client_id>/<client_id>/lifespan.py`:
- **Dynamic Model Provisioning**: Client-specific SQLModel overrides are imported and eagerly loaded before the database engine initiates `SQLModel.metadata.create_all()`.
- **Decoupled Business Rules**: Client custom routines hook dynamically into core events (e.g., `core.events.subscribe("ORDER_CONFIRMED", acme_callback)`) inside the application lifespan, enabling isolated client logic without polluting standard modules.

---

## 6. The Event-Based "Nervous System"

To keep standard extension modules, core operations, and custom client instances strictly decoupled, the Core provides an asynchronous internal Event Bus.
- **Asynchronous Emit & Subscribe**: Modules communicate exclusively through the `core.events` package using an asynchronous Pub/Sub pattern.
- **Decoupled Design**: The Sales module emits an `ORDER_CONFIRMED` event. It does not know if the Finance module or a custom client webhook is listening.
- **Graceful Fault Tolerance**: If a module subscribing to an event is unlicensed or inactive, the event bus swallows the payload without raising errors.
- **Growth Path**: Starts as an asynchronous in-memory asyncio event loop; designed to transition to a persistent Database Outbox or Redis queue for enterprise-grade transactional resilience.

---

## 7. Frontend Monorepo Architecture (Vite & React)

The frontend is built inside an Nx monorepo utilizing strict workspace boundaries.

### A. Directory Segmentation
- **The Shell (`apps/shell/`)**: Acts as the portal skeleton. It handles user authentication, session state, global navigation bar, sidebar menus, theme toggling, and routing entry points. It contains no module-specific business views.
- **Feature Libraries (`libs/<module>/`)**: Module libraries contain the functional UI screens, data grid tables, customized drawers, and local services.
- **Shared Library (`libs/shared-ui/`)**: Houses primary design tokens, base layout styles, core utilities, and standard UI elements.

### B. Dynamic Component Registry
To maintain zero direct routing imports within the Shell, modules dynamically register their custom UI views inside the shell registry during module initialization (`init<Module>Module()`):
```typescript
import { ComponentRegistry } from '@bes/shared-ui';
import FinanceMainView from './views/FinanceMainView';

export function initFinanceModule() {
  ComponentRegistry.register('Route_FinanceMain', FinanceMainView);
}
```

---

## 8. Subscription Controls & UI Graceful Degradation

The React UI adapts fluidly to the licensing configuration sent by the backend `/api/v1/bootstrap` endpoint during startup.

### A. UI Graceful Degradation (Read-Only State)
If a module is marked as `READONLY_EXTENSIONS` within the bootstrap config, the frontend automatically degrades the interface:
- Input forms, checkboxes, select menus, and action buttons are disabled.
- Interactive data tables lock edits while preserving search and detail views.
- Visual warnings (e.g., "Review Mode - Write Disabled") display in active headers.

### B. Upgrade Lockouts & Upsell Overlay Experience
Instead of hiding unlicensed premium modules or sub-features entirely (which hurts upsell discoverability), the UI implements **Subscription Gates**:
1. **Lock Indicators 🔒**: Unlicensed modules or high-tier sub-features display a premium lock symbol (e.g., `Finance 🔒` or `Cycle Counts 🔒`) in menu items, tabs, and action triggers.
2. **Upgrade Plan Card Overlay**: Clicking a locked element intercepts the route and presents an attractive, premium-styled **Upgrade Plan** modal or full-screen card overlay listing pricing options and feature highlights, encouraging users to tier up.

---

## 9. Core System-wide Guiding Principles

### A. The Money Rule (Decimal Precision)
Floating-point errors are unacceptable for business-critical calculations:
- **Backend Enforcements**: Financial fields in DB models must use `sa_column=Column(Numeric(precision=20, scale=4))`. Python variables MUST use Python's native `Decimal` class.
- **Frontend Enforcements**: High-precision math and currency formatting must use `decimal.js` or `big.js`.

### B. Standardized API Responses
All endpoints return an envelope built from the Core's standard response system:
```json
{
  "status": "success",
  "data": { ... },
  "metadata": {
    "count": 100,
    "page": 1,
    "limit": 50
  },
  "error": null
}
```
- **Global Error Formatting**: All unhandled exceptions, FastAPI validation errors, optimistic locking `ConcurrencyError`s, and standard `HTTPException`s (including 404, 403, 401) are wrapped globally via exception handlers in `core.responses` to ensure they always conform to the standard error envelope:
  ```json
  {
    "status": "error",
    "data": null,
    "metadata": null,
    "error": "Error message description"
  }
  ```


### C. Audit Trails & Soft Deletes
Data integrity is paramount. Physical deletions are disabled across all business tables. Inactive records are flagged with `is_deleted = true`. System events, schema updates, and REST requests automatically record transactional `Request-ID` tags, and standard updates update the `updated_at` timestamps instantly.

### D. Centralized Storage & UOM Registry
- **Storage Service**: Core provides unified `upload()` / `download()` functions wrapping local or S3 compatible storage, preventing isolated implementation in individual extensions.
- **Unit of Measure (UOM) Registry**: Master translation grids are maintained inside a centralized `uoms` database table in Core to prevent data fragmentation.
