# Run Guide

This document describes the most common commands for local development, client onboarding, Docker deployment, and tests for the BES workspace.

## Setup

From the repo root, enter the backend workspace and activate the Python virtual environment:

```bash
cd /Users/bvk/BVK_Workspace/BES/bes-backend
source .venv/bin/activate
```

Use `pdm` for backend commands once the environment is active.

## 1. Add a new extension

```bash
# Scaffold a standard flat extension layout
pdm run python scripts/add_extension.py --id sales --name Sales

# Scaffold a modular directory/package layout (recommended for large modules)
pdm run python scripts/add_extension.py --id settings --name Settings --modular
```

## 2. Remove an extension

```bash
# Completely delete an extension and revert its configurations
pdm run python scripts/remove_extension.py --id sales --force
```

## 3. Onboard a client

```bash
pdm run python scripts/onboard_client.py
```

## 4. Change a client license plan

```bash
pdm run python scripts/change_license.py --client-id test_custom --plan basic
pdm run python scripts/change_license.py --client-id test_custom --plan pro
```

## 5. Run the FastAPI server for a client

```bash
pdm run uvicorn instances.test_custom.test_custom.main:app --reload
```

## 6. Docker deployment and orchestration

From the repo root:

```bash
cd /Users/bvk/BVK_Workspace/BES
docker compose up --build -d
```

To start the full stack for a specific client instance:

```bash
CLIENT_ID=acme_inc docker compose up --build -d
```

To stop the stack:

```bash
docker compose down
```

Build individual containers when needed:

```bash
cd /Users/bvk/BVK_Workspace/BES/bes-frontend
docker build -t bes-frontend-shell:latest .

cd /Users/bvk/BVK_Workspace/BES/bes-backend
docker build -f instances/acme_inc/Dockerfile -t bes-backend-acme_inc:latest .
```

## 7. Run backend tests (PDM workspace)

```bash
pdm run test
pdm run test-cov
pdm run test-core
pdm run test-extensions
pdm run test-instances
```

## 8. Run frontend tests (Nx workspace)

```bash
cd /Users/bvk/BVK_Workspace/BES/bes-frontend
npx nx run-many -t test
npx nx test shared-ui
npx nx e2e shell-e2e
npx nx affected -t test
```
