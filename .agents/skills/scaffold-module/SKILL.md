---
name: scaffold-module
description: "End-to-end bootstrap of a new BES module — backend extension + frontend library + shell registration + planning directory."
---

# Skill: Scaffold a New BES Module

This skill creates the complete boilerplate for a new BES module across both backend and frontend, with proper shell registration and planning structure. It prevents the recurring build bugs documented in Rule 5.

---

## Prerequisites
- Core kernel (`bes-backend/core/`) is built and functional
- Shell app (`bes-frontend/apps/shell/`) is built and functional
- `@bes/shared-ui` library is available

---

## Naming Conventions

| Input Name | Backend Directory | Frontend Library | Module Key | Display Name |
|:---|:---|:---|:---|:---|
| `finance` | `extensions/finance/finance/` | `libs/finance/` | `finance` | `Finance` |
| `supply_chain` | `extensions/supply_chain/supply_chain/` | `libs/supply-chain/` | `supply-chain` | `Supply Chain` |
| `hr` | `extensions/hr/hr/` | `libs/hr/` | `hr` | `HR` |
| `crm` | `extensions/crm/crm/` | `libs/crm/` | `crm` | `CRM` |

> **Rule**: Backend uses `snake_case`. Frontend uses `kebab-case`. Display names use Title Case.

---

## Steps

### Step 1: Pre-flight Check

1. Use Graphify to verify the module doesn't already exist:
   ```
   graphify query "Does extension <module_name> exist?"
   ```
2. Check `features-plan/planning1.json`, `planning2.json`, `planning3.json` to identify which development phase the module belongs to.
3. Confirm the module isn't already scaffolded in `bes-backend/extensions/` or `bes-frontend/libs/`.

### Step 2: Backend Extension Scaffold

Create the full directory structure at `bes-backend/extensions/<module_name>/<module_name>/`:

#### 2a. Directory Structure
```
bes-backend/extensions/<module_name>/
├── <module_name>/
│   ├── __init__.py
│   ├── models.py
│   ├── schemas.py
│   ├── services.py
│   ├── router.py
│   ├── events.py
│   ├── manifest.py
│   └── process_definitions/     (empty directory)
├── tests/
│   ├── __init__.py
│   └── conftest.py
└── pyproject.toml
```

#### 2b. File Templates

**`__init__.py`** — Empty file.

**`models.py`**:
```python
"""
Database models for the <MODULE_NAME> module.
All models inherit from BESBase (provides id, created_at, updated_at,
created_by, is_deleted, subsidiary_id, metadata_, version_id).
"""
from typing import Optional
from decimal import Decimal
from sqlmodel import SQLModel, Field
from core.models import BESBase
```

**`schemas.py`**:
```python
"""
Pydantic schemas for API request/response validation.
NEVER use ORM models as API input — always define explicit Create/Read schemas.
"""
import uuid
from typing import Optional
from decimal import Decimal
from pydantic import BaseModel
```

**`services.py`**:
```python
"""
Business logic and transaction orchestration for the <MODULE_NAME> module.

Rules:
- Services own the transaction boundary (commit/rollback).
- NEVER raise HTTPException — raise domain exceptions instead (Rule 1 §14).
- Emit events AFTER successful commit, not before.
"""
from sqlmodel.ext.asyncio.session import AsyncSession
from core.repository import BaseRepository
```

**`router.py`**:
```python
"""
FastAPI router — thin HTTP layer that calls services.
NO business logic belongs here. Catch domain exceptions and translate to HTTP responses.
"""
from fastapi import APIRouter, Depends, HTTPException
from core.responses import success_response, paginated_response
from core.auth import get_current_user
from core.database import get_async_session

router = APIRouter(prefix="/api/v1/<module_name>", tags=["<Module Name>"])
```

**`events.py`**:
```python
"""
Event Bus subscribers and emitters for cross-module communication.
"""
from core.events import event_bus
```

**`manifest.py`**:
```python
"""
Extension manifest — registers this module with the core engine.
"""
from core.extension import ExtensionManifest
from .router import router

manifest = ExtensionManifest(
    name="<module_name>",
    description="<Module Display Name> management module",
    routers=[router],
)
```

**`tests/conftest.py`**:
```python
"""
Test fixtures for the <MODULE_NAME> extension.
Inherits shared fixtures from core test configuration.
"""
import pytest
from pathlib import Path
import sys

# Add core to path for shared fixtures
core_path = Path(__file__).resolve().parents[3] / "core"
sys.path.insert(0, str(core_path))

from tests.conftest import *  # noqa: Import shared fixtures
```

**`pyproject.toml`**:
```toml
[project]
name = "<module_name>"
version = "0.1.0"
dependencies = ["core"]

[build-system]
requires = ["pdm-backend"]
build-backend = "pdm.backend"
```

### Step 3: Frontend Library Scaffold

Follow the `util-create-ui-module` skill for detailed steps, but in summary:

1. Generate the Nx library:
   ```bash
   cd bes-frontend
   npx nx generate @nx/react:library <module-name> \
     --directory=libs/<module-name> \
     --importPath=@bes/<module-name> \
     --bundler=none --unitTestRunner=none --style=css --no-interactive
   ```
2. Add Vitest configuration (see `util-create-ui-module` Step 1).
3. Create module entry point with `init<ModuleName>Module()` and `ComponentRegistry.registerLazy`.
4. Create the home page component with `TabGroup` layout.
5. Create a placeholder unit test.

### Step 4: Shell Registration

> [!CAUTION]
> Both of these steps are MANDATORY per Rule 5 (Build Bug Prevention). Missing either makes the module invisible.

1. **Add module initializer** to `apps/shell/src/store/auth-store.ts` → `initializeModules()`:
   ```typescript
   import { init<ModuleName>Module } from '@bes/<module-name>';
   // Inside initializeModules():
   init<ModuleName>Module();
   ```

2. **Add module key** to `allModulesList` in `apps/shell/src/hooks/use-shell.ts`:
   ```typescript
   const allModulesList = [
     // ... existing modules
     '<Module Display Name>',
   ];
   ```

3. **Add module icon** to `apps/shell/src/app/app-config.tsx`:
   ```typescript
   export const MODULE_ICONS: Record<string, React.ReactNode> = {
     // ... existing
     '<module-name>': <IconComponent size={20} />,
   };
   ```

4. **Add resource names and route keys** to `libs/shared-ui/src/lib/constants.ts`.

### Step 5: Planning Directory

Create the feature planning directory:
```bash
mkdir -p features-plan/<module-name>/
```

### Step 6: Verification

1. Run `graphify update .` to index the new files.
2. Verify backend: `cd bes-backend && python -c "from extensions.<module_name>.<module_name>.manifest import manifest; print(manifest.name)"`
3. Verify frontend: `cd bes-frontend && npx nx test <module-name>`
4. Verify shell build: `cd bes-frontend && npx nx build shell`

---

## Post-Scaffold: What's Next?

After scaffolding, the module is an empty shell. Use the planning pipeline to design features:

1. **Plan**: Use `plan-orchestrator` skill (or run `plan-1-functionalities` through `plan-4-review`)
2. **Build**: Use `build-backend` and `build-frontend` skills
3. **Seed**: Use `seed-data` skill for demo data
4. **Verify**: Use `verify-build` skill for post-build checks
