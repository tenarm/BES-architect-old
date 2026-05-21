# Backend Architecture Process Flows

This document contains Mermaid diagrams illustrating the workflows parallel to the commands in [run.txt](file:///Users/bvk/BVK_Workspace/BES/bvk-items/commands/backend/run.txt).

---

## 1. Extension Creation Workflow (`add_extension.py`)

```mermaid
flowchart TD
    A([Start: add_extension.py]) --> B[Prompt or Read id & name]
    B --> C[Create folders: extensions/id/id/]
    C --> D[Generate Boilerplate: manifest.py, models.py, router.py, events.py, pyproject.toml]
    D --> E[Register package in core/core/packages.json & admin_permissions.json]
    E --> F[Add editable dependency to root pyproject.toml dev group]
    F --> G[Run pdm lock -d & pdm install -d]
    G --> H([End: Extension Registered & Synced])
```

---

## 2. Client Onboarding Workflow (`onboard_client.py`)

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

## 3. License Upgrade/Degrade Workflow (`change_license.py`)

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

## 4. Client Server Lifespan & Boot Workflow (`uvicorn`)

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
