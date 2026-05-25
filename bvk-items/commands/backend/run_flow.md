# Backend Architecture Process Flows

This document contains Mermaid diagrams illustrating the workflows parallel to the commands in [run.md](file:///Users/bvk/BVK_Workspace/BES/bvk-items/commands/backend/run.md).

---

## 1. Extension Creation Workflow (`add_extension.py`)

```mermaid
flowchart TD
    A([Start: add_extension.py]) --> B[Prompt or Read id, name, and --modular flag]
    B --> C{Is --modular active?}
    C -- Yes --> D1[Create package directories: models/, schemas/, services/, router/]
    C -- No --> D2[Create flat files: models.py, schemas.py, services.py, router.py]
    D1 --> E[Generate package __init__.py files and boilerplate templates]
    D2 --> F[Generate standard flat boilerplate files]
    E --> G[Register package in core/core/packages.json & admin_permissions.json]
    F --> G
    G --> H[Add editable dependency to root pyproject.toml dev group]
    H --> I[Run pdm lock -d & pdm install -d]
    I --> J([End: Extension Registered & Synced])
```

---

## 2. Extension Deletion Workflow (`remove_extension.py`)

```mermaid
flowchart TD
    A([Start: remove_extension.py]) --> B[Read id & --force flag]
    B --> C{Force delete or confirm interactive prompt?}
    C -- Confirmed / Force --> D[Delete physical extensions/id/ directory]
    C -- Aborted --> E([Cancel: Exit script])
    D --> F[Remove editable dependency from root pyproject.toml dev group]
    F --> G[Unregister package in core/core/packages.json & admin_permissions.json]
    G --> H[Run pdm lock -d & pdm install -d in root workspace]
    H --> I([End: Extension Deleted & Synced])
```

---

## 3. Client Onboarding Workflow (`onboard_client.py`)

```mermaid
flowchart TD
    A([Start: onboard_client.py]) --> B[Prompt: client_id, client_name, DB URL]
    B --> C[Prompt: Select Subscription Plan]
    C --> D[Filter master admin_permissions.json based on plan]
    D --> E[Check & Merge instances/id/config/custom_permissions.json]
    E --> F[Write instances/id/config/onboard_config.json with status & timestamps]
    F --> G[Write instances/id/config/admin_permissions.json]
    G --> H[Generate instances/id/pyproject.toml linking physical extensions]
    H --> I[Copy template app to instances/id/id/ and replace client_id placeholders]
    I --> J[Run pdm build inside instance directory]
    J --> K[Register client entry in root pyproject.toml dev group]
    K --> L[Run pdm lock -d & pdm install -d]
    L --> M([End: Client Onboarded & Active])
```

---

## 4. License Upgrade/Degrade Workflow (`change_license.py`)

```mermaid
flowchart TD
    A([Start: change_license.py]) --> B[Verify client folder exists & load current onboard_config.json]
    B --> C[Prompt/Read: Select New Subscription Plan]
    C --> D[Filter permissions from master admin_permissions.json]
    D --> E[Check & Merge custom_permissions.json if present]
    E --> F[Update onboard_config.json: set plan, set custom_config flag, keep registered_at, update updated_at]
    F --> G[Write updated admin_permissions.json]
    G --> H[Re-generate instances/id/pyproject.toml dependencies]
    H --> I[Run pdm build inside instance directory]
    I --> J[Run pdm lock -d & pdm install -d in root workspace]
    J --> K([End: Client License Modified & Synced])
```

---

## 5. Client Server Lifespan & Boot Workflow (`uvicorn`)

```mermaid
flowchart TD
    A([Start: uvicorn Main App]) --> B[Load test_custom.bootstrap: client config & licensed modules list]
    B --> C[Import and load core exceptions, routes & middleware]
    C --> D[Dynamic Route Discovery: loop over licensed modules]
    D --> E{Physical extension exists?}
    E -- Yes --> F[Import manifest.py and register Router]
    E -- No --> G[Print Warning, continue startup]
    F --> H[SQLModel Schema Registration: register models from active manifests]
    G --> H
    H --> I[FastAPI Lifespan Startup Hook]
    I --> J[Run migrations: create database tables for registered schemas]
    J --> K[Seed default admin roles and default admin user if missing]
    K --> L([Uvicorn Ready & Listening on Port 8000])
```

---

## 6. Testing Pipeline Workflow (`pdm run test`)

```mermaid
flowchart TD
    A([Start: pdm run test]) --> B[Load tool.pytest.ini_options: ignore scratch/build directories]
    B --> C[Pytest Collection: locate tests/ folders under core, extensions, instances]
    C --> D[Initialize Session-Scoped In-Memory SQLite: sqlite+aiosqlite:///:memory:]
    D --> E[Build Schema: run SQLModel.metadata.create_all]
    E --> F[Test Run Execution: yield isolated db_session per test case]
    F --> G[FastAPI Override: inject test db_session into APIRoutes]
    G --> H[Execution Ends: tear down in-memory SQLite tables]
    H --> I([End: Test Results Output])
```

---

## 7. Frontend Unit & Component Testing Pipeline (`npx nx test [lib]`)

```mermaid
flowchart TD
    A([Start: npx nx test lib]) --> B[Parse project.json & load vite.config.mts]
    B --> C[Vitest Initialization: setup jsdom & load global mocks from shared-ui test-setup.ts]
    C --> D[Scan specifications: locate *.spec.ts or *.spec.tsx]
    D --> E[Execute tests: run component render / store updates / RBAC mapping]
    E --> F[Verify 80% coverage requirements if configured]
    F --> G([End: Test Results & Coverage output])
```

---

## 8. Frontend E2E Testing Pipeline (`npx nx e2e shell-e2e`)

```mermaid
flowchart TD
    A([Start: npx nx e2e shell-e2e]) --> B[Load playwright.config.ts]
    B --> C[Launch Vite Dev Server: serve shell locally in background]
    C --> D[Browser Initialization: spin up headless Chromium on port 4200]
    D --> E[Sequence Execution: run Playwright tests sequentially to prevent port conflict]
    E --> F[Mock Network Layer: intercept backend endpoints & rate-limit SSE stream]
    F --> G[Assess assertions: verify sign-in, authentication persistence, and subscription gate behavior]
    G --> H[Shutdown server & close browser context]
    H --> I([End: E2E Test Report & Artifacts])
```

