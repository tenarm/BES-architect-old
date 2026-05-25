---
trigger: always_on
description: Technical architecture, Multi-Tenancy, DB schemas, transaction boundary, and security rules for all Python/FastAPI backend files in the bes-backend/ directory.
---

# 1. Backend Architecture & Security Rules — BES

These rules govern all Python/FastAPI code and architectural patterns in the `bes-backend/` directory.

## 1. Core-as-a-Package & Repository Layout
- **The Engine (`bes-core`)**: The `bes-backend/core/` package is the kernel (auth, DB, RBAC, events, base models). NEVER add domain-specific business logic here.
- **Extensions**: Business modules live in `bes-backend/extensions/<module_name>/`. They depend on `core`. Core MUST NOT import from extensions.
- **Dependency**: Extensions declare core in `pyproject.toml` (`dependencies = ["core"]`).
- **Naming**: Use `snake_case` for extensions, `<module>_<entity>s` for DB tables, and `UPPER_SNAKE_CASE` for events.

## 2. Extension Module Strict Layering
Every extension MUST follow a strict layer division (models, schemas, services, router, events, manifest). For simple extensions, these can be flat files:
- **`models.py`**: ONLY database table classes inheriting from `BESBase`.
- **`schemas.py`**: Pydantic `*Create` and `*Read` classes for API validation. NEVER use ORM models as API input.
- **`services.py`**: All business logic, transaction safety, and cross-table operations.
- **`router.py`**: Thin HTTP layer. Calls services. NO business logic.
- **`events.py`**: Event bus subscribers and emitters.
- **`manifest.py`**: MUST export a `manifest` instance of `ExtensionManifest`.

### Scaling Up: Modular Package Structure
When an extension module scales up with multiple sub-features or distinct business entities (e.g. `settings`), flat files should be modularized into directories (packages):
- **Directories**: Replace flat files with directories named `models/`, `schemas/`, `services/`, and `router/`.
- **Packaging (`__init__.py`)**: Each folder must contain an `__init__.py` file that re-exports its contents to preserve import compatibility from external packages (e.g., `from settings.models import CompanyProfile`).
- **Routing Aggregation**: In `router/__init__.py`, initialize the main `APIRouter` (declaring the common prefix and tags) and use `.include_router()` to mount the sub-routers defined under the `router/` subdirectory (e.g. `company.py`, `user.py`), keeping routers thin and modular.


## 3. Database, Models, and Isolation (Multi-Tenancy)
- **Base Inheritance**: ALL tables MUST inherit from `core.models.BESBase` (provides UUID `id`, `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `metadata_`, `version_id`).
- **Multi-Tenancy**: DB-per-tenant isolation. Within a DB, `subsidiary_id` scopes data. Queries automatically filter via `ContextAwareSecurityMiddleware` and `BaseRepository._apply_scopes()`.
- **Hub-and-Spoke MDM**: Core Master data (`customers`, `vendors`) resides in `core`. Module-specific tables use FKs to core.
- **Soft Deletes**: Physical deletion is FORBIDDEN. Use `is_deleted = True`. Queries must filter by `is_deleted == False`.
- **The Money Rule**: ALL financial amounts MUST use `sa_column=Column(Numeric(precision=20, scale=4))` in DB and `Decimal` in Python. NEVER use `float`.
- **Concurrency Control**: 
  - **Optimistic Locking**: Handled automatically via `version_id` on `BESBase`. Updates MUST pass the loaded version ID to `BaseRepository.update` to prevent edit overrides; conflicts raise `ConcurrencyError` and return `409 Conflict`.
  - **Pessimistic Locking**: Use `BaseRepository.get_with_lock(session, id)` to lock rows with `FOR UPDATE` for high-contention database modifications.


## 4. API Standards & Pagination
- **Envelope**: ALL responses (including successful results, field validation errors, concurrency collisions, and standard `HTTPException`s) MUST conform to the `StandardResponse` envelope (`{ status, data, metadata, error }`). This is enforced globally via FastAPI exception handlers configured by `setup_exception_handlers` in `core.responses`.
- **Pagination**: ALL list endpoints MUST support pagination using `core.pagination.PaginationParams`. Max page size is 200.
- **Route Prefix**: `/api/v1/<module>`.


## 5. Security & RBAC
- **Secrets**: NEVER hardcode secrets. Use `os.environ["KEY"]` (fail-fast).
- **Authentication**: JWT Access (short-lived) and Refresh (long-lived) tokens. Passwords hashed via bcrypt.
- **RBAC**: Endpoints modifying sensitive data MUST use `require_permission()`. Format: `<module>:<resource>:<action>` (e.g., `finance:coa:write`).
- **System Context**: Use `elevate_context()` to bypass checks for event-driven automated operations.
- **Audit**: `X-Request-ID` logging, and `updated_at` auto-updates on modifications.

## 6. Inter-Module & Customization Communication
- Modules communicate ONLY through the asynchronous **Event Bus** (`core.events`).
- Standard extension modules MUST NOT directly import or call another extension module's or client-specific instance's code.

## 7. Modular Process Pipeline Definitions
- **Assessment Requirement**: When specifying or planning new functionalities, you MUST check if a multi-step, approval-driven, or asynchronous process pipeline is required (e.g., employee exit clearance). Direct synchronous changes (e.g., GL entry creation updating the COA) should be distinguished from multi-layered pipelines.
- **Process Schema Files**: Every extension module that implements standard multi-step processes or pipelines MUST specify them inside `extensions/<module_name>/<module_name>/process_definitions/<module_name>.json`.
- **Dynamic Aggregation & Self-Healing Filtering**: The core engine (`core/core/processes.py`) dynamically scans all active extension directories for `process_definitions/*.json` on boot, validating their structure against strict **Pydantic** models. 
  - To handle licensing bounds, any step belonging to an unlicensed module/feature MUST define `requiredModule` or `requiredFeature` in the step schema.
  - The backend dynamically filters out these unlicensed steps and recursively rewires their dependencies (`dependsOn`) to keep the pipeline intact (Self-Healing).
- **API Exposing**: Process schemas are exposed to the frontend via the `GET /api/v1/audit/processes` router (filterable by `?module={module_name}`).

## 8. Subscription Packaging & Tier Licensing
- **Tier Configuration (`core/core/packages.json`)**: Feature scopes are defined under three tiered offerings: **Basic**, **Pro**, and **Premium**.
- **Granular License Verification**: Service layers and routing endpoints must enforce granular limits using `require_licensed_feature("module", "subfeature")`. 
- **Enforcement Mechanics**:
  - In a Web request context, checking failure raises a standard 403 Forbidden `HTTPException`.
  - In background processes or queue workers, checking failure raises `LicensingError`.
- **System Bypass**: Privileged background routines and system-initiated event listeners bypass license verification when wrapped in `with elevate_context():`.

## 9. Client Customization & Instance Extension Foundation
- **Code Separation**: Standard extension modules and the core engine are 100% tenant-agnostic. All client-specific code (custom endpoints, business logic, DB models, integrations) MUST live strictly within `instances/<client_id>/`.
- **Cli Onboarding & Merging Permissions**:
  - Clients are provisioned using `scripts/onboard_client.py`.
  - Private custom client permissions reside in `instances/<client_id>/config/custom_permissions.json` (generated using the interactive CLI `scripts/custom_feature_config.py`). These are merged dynamically into the instance's active `admin_permissions.json` on onboarding.
- **Dynamic Model Provisioning**: Client-specific models must reside in `instances/<client_id>/<client_id>/models/`. They must be eagerly registered inside the client chassis's `lifespan.py` lifespan context before executing `SQLModel.metadata.create_all` on startup.
- **Decoupled Business Rules**: Client instances subscribe dynamically to the core event bus (`core.events`) inside `lifespan.py` to trigger custom webhooks or handlers without mutating standard extension packages.

## 10. Beginner-Friendly & Self-Documenting Design
- **Clean Structure**: Code modules must follow a strict file division (models, schemas, services, router, events). This separation makes it intuitive for beginners to know exactly where logic goes.
- **Self-Documenting Code**: Document all functions, classes, and service actions with explicit docstrings, parameter types, and return types. Use inline comments to explain non-obvious business logic, avoiding complex code shortcuts.
- **Traceability**: All cross-module workflows should emit trace events with explicit `correlation_id` values, ensuring operations can be tracked sequentially in the database and audit timeline.

## 11. Error-Forgiving API Design
- **Informative Exceptions**: Do not crash on bad user input or licensing failures. APIs must catch expected errors (e.g. database constraints, validation errors, licensing limits) and translate them into clear, human-readable error messages via `StandardResponse` or `HTTPException`.
- **Validation Helpers**: Leverage Pydantic's descriptive validation errors to highlight exactly which fields failed validation, providing actionable feedback to the caller.
- **Soft Deletion**: Always enforce soft deletes via `is_deleted = True` for models inheriting `BESBase` to prevent permanent, accidental data loss.

## 12. Import Ordering
1. Standard library (`os`, `uuid`, `datetime`, `decimal`)
2. Third-party (`fastapi`, `sqlmodel`)
3. Core package (`core.database`, `core.responses`)
4. Current module (`.models`, `.schemas`, `.services`)

## 13. Core Notification Service Architecture
- **Event-Driven Dispatch**: Business logic services must NEVER instantiate or save notification records directly. They must emit high-level Pub/Sub events (e.g. `SALES_ORDER_COMPLETED`) to `core.events` via `event_bus.emit(event)`.
- **Database Rule Seeding**: Every module must seed default `NotificationRule` records for its key events. A rule defines:
  - `event_type`: The matching `UPPER_SNAKE_CASE` event payload type.
  - `channel`: The delivery medium, either `IN_APP` or `EMAIL`.
  - `recipient_type`: How recipients are identified (`USER`, `ROLE`, or `DYNAMIC_PATH`).
  - `recipient_target`: The user ID, role name, or a JSONPath selector string matching the event payload (e.g., `$.data.created_by` or `$.data.customer_email`).
  - `template_title` / `template_body`: Jinja2 formatted template strings to construct the notification content dynamically from event variables.
- **Channel Licensing & Lockouts**: The core notifications processor checks if features are licensed before triggering rules. For example, if a tenant's basic plan does not license the settings rule customization feature (e.g., `settings:notification_rules` or custom channels), premium channels like `EMAIL` are bypassed and skipped, while standard `IN_APP` notifications proceed if allowed.
- **SSE Stream**: Real-time delivery of `IN_APP` notifications uses `notification_broadcaster` to stream updates over a `/api/v1/notifications/stream` Server-Sent Events (SSE) connection.

## 14. Business Service Layer & Domain Rules
To ensure the business logic layer (`services.py`) remains decoupling-ready, maintainable, and mathematically robust, follow these strict rules:

- **Decoupled Domain Exceptions**:
  - Business services MUST NOT raise web-specific exceptions like FastAPI `HTTPException` directly.
  - Instead, services must define and raise module-specific custom Exceptions (e.g., `DuplicateTaxRegistrationError`, `InsufficientStockError`, or a base `DomainException`).
  - The API router layer (`router.py`) or a central FastAPI exception handler is responsible for catching these domain exceptions and translating them into standard HTTP status codes and responses.
  
- **Service Transaction Boundaries & Unit of Work**:
  - The service layer is the sole orchestrator of database transactions.
  - The repository layer (`BaseRepository`) and router layer (`router.py`) MUST NOT call `session.commit()` or `session.rollback()`.
  - Service functions must run their validations, mutations, and insertions under a shared session context. Call `await session.commit()` only after all business rules have been successfully validated and applied.
  - If a multi-entity orchestration flow is required, pass the same `AsyncSession` reference down through all helper service calls to ensure they share the same transaction (Unit of Work).

- **Service Hierarchy & Circular Import Prevention**:
  - Services must be logically layered to prevent circular dependencies within extension modules:
    - **Leaf Services**: Handle CRUD and basic invariants for a single specific entity (e.g., `CustomerService`). They never import or reference other services.
    - **Composite Services**: Orchestrate workflows across multiple leaf services or modules (e.g., `OrderProcessingService` calling `CustomerService` and `InventoryService`).
  - Cross-module business reactions must always be decoupled using the asynchronous **Event Bus** (`core.events`).

- **Mathematical Calculation & Rounding Safeties**:
  - All mathematical operations involving currency, prices, or ledger totals MUST use Python's `decimal` library.
  - Perform all intermediate calculations at maximum precision to prevent rounding error accumulation.
  - Quantize and round only at the final document boundary or ledger posting stage. Round values to exactly 4 decimal places (`Numeric(20,4)`) using `.quantize(Decimal("0.0001"), rounding=ROUND_HALF_UP)`.

- **Safe Post-Commit Event Dispatching**:
  - High-level Pub/Sub events MUST NOT be emitted before the database transaction commits successfully.
  - If a service publishes an event and the database commit subsequently fails, external subscribers will act on invalid data.
  - Collect events in an in-memory session/request queue during processing, and dispatch them to the event bus immediately *after* the `await session.commit()` call has resolved successfully.

- **State Machine Transition Invariants**:
  - For entities that progress through workflow statuses (e.g. `Draft -> Approved -> Fulfilled -> Invoiced`), the service layer is responsible for enforcing status transition rules.
  - Status updates must be checked against a strict, valid transition matrix. Avoid direct/arbitrary status mutations.
  - Atomic side effects associated with a transition (e.g., locking inventory stock upon moving to `Fulfilled` or posting immutable journals upon moving to `Invoiced`) must execute synchronously within the same database transaction.




