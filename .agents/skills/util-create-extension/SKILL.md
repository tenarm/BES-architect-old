---
name: create-extension
description: This skill walks you through creating a complete new BES backend extension module that follows all architectural rules.
---

# Skill: Create a New Backend Extension Module

This skill walks you through creating a complete new BES backend extension module that follows all architectural rules.

---

## Prerequisites
- Know the module name (e.g., `procurement`)
- Know the initial entities/tables needed (e.g., `PurchaseOrder`, `Vendor`)
- The module must be listed in the `admin_permissions.json` schema

## Steps

### Step 1: Create the Extension Directory

```bash
mkdir -p bes-backend/extensions/<module_name>/<module_name>
```

### Step 2: Create `pyproject.toml`

```toml
# bes-backend/extensions/<module_name>/pyproject.toml
[project]
name = "<module_name>"
version = "0.1.0"
description = "<Module Display Name> Extension Module"
requires-python = ">=3.10"
dependencies = [
    "core"
]
```

### Step 3: Create `__init__.py`

```python
# bes-backend/extensions/<module_name>/<module_name>/__init__.py
"""<Module Display Name> Extension"""
```

### Step 4: Create `models.py`

Define ONLY database table classes here. All tables inherit from `BESBase`.

```python
# bes-backend/extensions/<module_name>/<module_name>/models.py
import uuid
from typing import Optional
from decimal import Decimal
from sqlmodel import Field
from sqlalchemy import Column, Numeric
from core.models import BESBase


class MyEntity(BESBase, table=True):
    __tablename__ = "<module_name>_<entities>"  # e.g., "procurement_purchase_orders"
    
    name: str
    status: str = Field(default="DRAFT")
    amount: Decimal = Field(
        default=Decimal("0.0"),
        sa_column=Column(Numeric(precision=20, scale=4))
    )
    # Add foreign keys to core entities as needed:
    # customer_id: uuid.UUID = Field(foreign_key="customers.id")
```

### Step 5: Create `schemas.py`

Separate API input/output from ORM models.

```python
# bes-backend/extensions/<module_name>/<module_name>/schemas.py
import uuid
from decimal import Decimal
from typing import Optional
from sqlmodel import SQLModel


class MyEntityCreate(SQLModel):
    name: str
    amount: Decimal = Decimal("0.0")


class MyEntityRead(SQLModel):
    id: uuid.UUID
    name: str
    status: str
    amount: Decimal
```

### Step 6: Create `services.py`

All business logic lives here.

```python
# bes-backend/extensions/<module_name>/<module_name>/services.py
import logging
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import HTTPException
from .models import MyEntity
from .schemas import MyEntityCreate

logger = logging.getLogger(__name__)


async def create_entity(session: AsyncSession, data: MyEntityCreate) -> MyEntity:
    db_obj = MyEntity.model_validate(data)
    session.add(db_obj)
    await session.commit()
    await session.refresh(db_obj)
    return db_obj
```

### Step 7: Create `router.py`

Thin HTTP layer that delegates to services and enforces licensing + permissions.

```python
# bes-backend/extensions/<module_name>/<module_name>/router.py
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from sqlmodel import select, func

from .models import MyEntity
from .schemas import MyEntityCreate
from .services import create_entity
from core.database import get_async_session
from core.responses import success_response, paginated_response
from core.pagination import PaginationParams
from core.licensing import require_licensed_feature
from core.rbac import require_permission

router = APIRouter(prefix="/api/v1/<module_name>", tags=["<module_name>"])


@router.get("/entities", dependencies=[Depends(require_permission("<module_name>:entity:read"))])
async def list_entities(
    pagination: PaginationParams = Depends(),
    session: AsyncSession = Depends(get_async_session)
):
    # Enforce granular subscription tier licensing check
    require_licensed_feature("<module_name>", "entity_management")

    count_stmt = select(func.count()).select_from(MyEntity).where(MyEntity.is_deleted == False)
    total = (await session.execute(count_stmt)).scalar() or 0
    
    stmt = (
        select(MyEntity)
        .where(MyEntity.is_deleted == False)
        .offset(pagination.offset)
        .limit(pagination.limit)
    )
    result = await session.execute(stmt)
    items = result.scalars().all()
    
    return paginated_response(data=items, total=total, page=pagination.page, page_size=pagination.page_size)


@router.post("/entities", dependencies=[Depends(require_permission("<module_name>:entity:write"))])
async def create_entity_endpoint(
    data: MyEntityCreate,
    session: AsyncSession = Depends(get_async_session)
):
    # Enforce granular subscription tier licensing check
    require_licensed_feature("<module_name>", "entity_management")

    item = await create_entity(session, data)
    return success_response(data=item)
```

### Step 8: Create `events.py`

Define event subscribers and emitter functions.

```python
# bes-backend/extensions/<module_name>/<module_name>/events.py
import logging
from core.events import event_bus, BaseEventPayload

logger = logging.getLogger(__name__)


async def emit_entity_created(entity_id: str, entity_name: str):
    payload = BaseEventPayload(
        emitter_module="<module_name>",
        event_type="<MODULE>_ENTITY_CREATED",
        data={"entity_id": entity_id, "name": entity_name}
    )
    await event_bus.emit(payload)


def register_event_handlers():
    # Subscribe to events from other modules
    # event_bus.subscribe("SOME_EVENT", handler_function)
    logger.info("[<Module>] Event handlers registered")
```

### Step 9: Create `manifest.py`

```python
# bes-backend/extensions/<module_name>/<module_name>/manifest.py
from fastapi import APIRouter
from core.extension import ExtensionManifest


class <Module>Manifest(ExtensionManifest):
    @property
    def module_name(self) -> str:
        return "<module_name>"

    def get_router(self) -> APIRouter:
        from .router import router
        return router

    def get_models(self) -> list[type]:
        from .models import MyEntity
        return [MyEntity]

    def get_event_handlers(self):
        from .events import register_event_handlers
        return register_event_handlers


manifest = <Module>Manifest()
```

### Step 10: Map Module inside Packages Subscription Tiers

Register the module and its granular features in `core/core/packages.json` under the appropriate subscription tiers:
```json
{
  "packages": {
    "basic": {
      "display_name": "Basic Plan",
      "modules": {
        "<module_name>": ["entity_management"]
      }
    },
    "pro": {
      "display_name": "Professional Plan",
      "modules": {
        "<module_name>": ["entity_management", "advanced_reporting"]
      }
    }
  }
}
```

### Step 11: Add Master Permissions

Register all available resources and granular actions for the new module in `core/core/admin_permissions.json`:
```json
{
  "<module_name>": {
    "entity_management": { "read": true, "write": true, "delete": true },
    "advanced_reporting": { "read": true, "write": true, "delete": true }
  }
}
```

### Step 12: Onboard Client Instance

Run the client onboarding CLI script to generate the client workspace chassis, compile licensed permissions, and build packages:
```bash
cd bes-backend
python scripts/onboard_client.py
```
1. Input your `client_id` and `client_name`.
2. Choose your subscription plan (Basic, Pro, Premium, or Custom).
3. The script will automatically filter permissions according to the selected plan and copy boilerplate configurations to `instances/<client_id>`.

### Step 13: Verify

```bash
cd bes-backend
pdm run uvicorn instances.<client_id>.<client_id>.main:app --reload
# Check: GET /health
# Check: GET /api/v1/<module_name>/entities (should return empty list if licensed)
```

