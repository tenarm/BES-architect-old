---
name: util-add-backend-test
description: "UTILITY STEP: Guide developers and AI assistants through systematically adding unit, integration, and E2E fusion tests inside the PDM monorepo workspaces."
---

# Skill: Adding Backend Tests inside PDM Monorepo

This utility skill guides developers and AI assistants on how to initiate and add tests to the **Core Kernel**, **Extensions**, and **Client Instances** in the `bes-backend` repository.

---

## 1. Test Directory Structure

Backend tests are co-located in their respective packages to preserve clean boundary isolation:

| Package Category | Source Code Directory | Test Target Directory | Purpose |
| :--- | :--- | :--- | :--- |
| **Core Kernel** | `core/core/` | [core/tests/](file:///Users/bvk/BVK_Workspace/BES/bes-backend/core/tests/) | Core units, Auth flows, RBAC tables, event persistence, and Notification SSE queues. |
| **Extensions** | `extensions/<name>/<name>/` | `extensions/<name>/tests/` | Isolated router handlers, calculations, models, and custom schema constraints. |
| **Client Instances** | `instances/<client_id>/<client_id>/` | `instances/<client_id>/tests/` | E2E dynamic module loading, fusion integrity, tenant database seeding, and licensing gates. |

---

## 2. Test Execution Commands

PDM script shortcuts are pre-defined in the root [pyproject.toml](file:///Users/bvk/BVK_Workspace/BES/bes-backend/pyproject.toml). You must always run tests using these wrappers:

*   **Run all tests**: `pdm run test`
*   **Run only core tests**: `pdm run test-core`
*   **Run only extensions tests**: `pdm run test-extensions`
*   **Run only client fusion tests**: `pdm run test-instances`
*   **Verify code coverage**: `pdm run test-cov`

---

## 3. How to Initiate Tests for a Component

### 3.1 Adding Tests to a New / Existing Core Module
1. Navigate to the [core/tests/](file:///Users/bvk/BVK_Workspace/BES/bes-backend/core/tests/) directory.
2. Create your test file following `test_<feature_name>.py`.
3. Import the shared fixtures (`client`, `db_session`, `test_app`, `test_engine`) dynamically from the local `conftest.py`.
4. Implement your test function as an async test using `@pytest.mark.asyncio`:
   ```python
   import pytest
   
   @pytest.mark.asyncio
   async def test_my_core_flow(client):
       response = await client.get("/api/v1/some-endpoint")
       assert response.status_code == 200
   ```

### 3.2 Adding Tests to a New / Existing Extension
1. Create a `tests/` folder in the extension package (e.g. `extensions/my_extension/tests/`).
2. Create a local `conftest.py` inside that folder, importing all core fixtures dynamically:
   ```python
   import sys
   from pathlib import Path
   
   # Resolve the core path to inherit standard database/client overrides
   core_path = str(Path(__file__).resolve().parents[3] / "core")
   if core_path not in sys.path:
       sys.path.append(core_path)
   
   from tests.conftest import *
   from fastapi import FastAPI
   from core.database import get_async_session
   
   @pytest.fixture
   def test_app(db_session) -> FastAPI:
       """App fixture containing core routers + the extension under test."""
       from my_extension.manifest import MyExtensionManifest
       app = FastAPI()
       
       # Register extension router dynamically
       manifest = MyExtensionManifest()
       app.include_router(manifest.get_router())
       
       # Override DB dependencies
       async def override_get_async_session():
           yield db_session
       app.dependency_overrides[get_async_session] = override_get_async_session
       return app
   ```
3. Implement your route/service tests (e.g. `test_router.py`) referencing this specialized `client`.

### 3.3 Adding Tests to a Client Instance (E2E Fusion)
1. Create a `tests/` folder in the instance directory (e.g. `instances/acme_corp/tests/`).
2. Create a local `conftest.py` importing the actual client app:
   ```python
   import sys
   from pathlib import Path
   
   core_path = str(Path(__file__).resolve().parents[3] / "core")
   if core_path not in sys.path:
       sys.path.append(core_path)
   
   from tests.conftest import *
   from fastapi import FastAPI
   from core.database import get_async_session
   
   @pytest.fixture
   def test_app(db_session) -> FastAPI:
       """Imports the client's actual main FastAPI app with all licensed modules fused."""
       from acme_corp.main import app as main_app
       
       # Apply database isolation
       async def override_get_async_session():
           yield db_session
       main_app.dependency_overrides[get_async_session] = override_get_async_session
       return main_app
   ```
3. Create `test_fusing.py` verifying license locking (require_licensed_feature returns `403 Forbidden` response for unlicensed features).

---

## 4. Coding & Architecture Best Practices for Testing

> [!IMPORTANT]
> **The Money Rule**: Always assert calculations return a `Decimal` numeric type (never floats) when writing finance-related assertion routines.

> [!WARNING]
> **Database Isolation**: Never allow test cases to read or write to local file databases (`bes.db`). All fixtures MUST override the `DATABASE_URL` environment variable to `sqlite+aiosqlite:///:memory:` before test bootstrap.

> [!TIP]
> **Mocking the Event Bus**: If you do not want an operation to emit background tasks, mock the Event Bus's `emit` command inside your test case:
> ```python
> from unittest.mock import AsyncMock
> from core.events import event_bus
> 
> @pytest.fixture
> def mock_event_bus(mocker):
>     return mocker.patch.object(event_bus, 'emit', new_callable=AsyncMock)
> ```

---

## 5. Maintenance
After introducing new testing configurations or placeholder suites, you MUST rebuild the knowledge graph by running:
```bash
graphify update .
```
This ensures that AST relations and communities reflect the changes.
