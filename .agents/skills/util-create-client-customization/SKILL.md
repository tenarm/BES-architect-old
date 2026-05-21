---
name: util-create-client-customization
description: This skill walks you through structuring client-specific custom code inside instances/<client_id>/, merging custom permissions, dynamic model discovery, and subscribing to core events.
---

# Skill: Create Client-Specific Customizations (Instance Extensions)

This skill guides you through the technical steps required to implement custom features, database tables, API routes, or event handlers for a specific client without polluting the standard extension modules.

---

## Architecture Golden Rules
1. **Perfect Isolation**: Core and standard extension modules (`bes-backend/extensions/`) MUST remain 100% client-agnostic. They must NEVER import from or depend on any file within `instances/<client_id>/`.
2. **Modular Custom Code**: All client-specific logic lives strictly inside `instances/<client_id>/<client_id>/`.
3. **Decoupled Extensions**: When custom behavior is needed in response to core system actions, subscribe dynamically to core events using the **Event Bus** instead of modifying core or standard extension code.

---

## Steps

### Step 1: Add Custom Permissions using the Interactive CLI

Run the interactive python builder to define your custom module and its granular actions (read, write, delete) for your specific client:

```bash
cd bes-backend
python scripts/custom_feature_config.py
```
- Select your client instance (e.g. `acme_corp`).
- Input the name of your custom module (e.g. `sap_integration`).
- Input a comma-separated list of sub-features (e.g. `sync, logs, settings`).
- This will automatically compile and save permissions to:
  `instances/<client_id>/config/custom_permissions.json`

---

### Step 2: Define Client-Specific DB Models

Create a standard model file inside the client package:
`instances/<client_id>/<client_id>/models/custom_sync.py`

All models must inherit from `BESBase` to automatically inherit tenant controls (`subsidiary_id`, `is_deleted` soft-deletes, and UUID keys):

```python
# instances/<client_id>/<client_id>/models/custom_sync.py
import uuid
from typing import Optional
from decimal import Decimal
from sqlmodel import Field
from sqlalchemy import Column, Numeric
from core.models import BESBase

class SAPSyncLog(BESBase, table=True):
    __tablename__ = "custom_sap_sync_logs"  # Prefix custom tables with "custom_"
    
    integration_type: str
    status: str = Field(default="PENDING")
    record_count: int = Field(default=0)
    synced_amount: Decimal = Field(
        default=Decimal("0.0"),
        sa_column=Column(Numeric(precision=20, scale=4))
    )
```

---

### Step 3: Register Custom Models in the Lifespan Context

To ensure the SQLModel migrations automatically discover and provision your client-specific DB tables on boot, eagerly import your custom models inside `instances/<client_id>/<client_id>/lifespan.py` before the database metadata executes:

```python
# instances/<client_id>/<client_id>/lifespan.py
# (Inside lifespan context before conn.run_sync(SQLModel.metadata.create_all))

# Force model registration:
from {{client_id}}.models.custom_sync import SAPSyncLog  # noqa: F401
```

---

### Step 4: Implement Decoupled Event Listeners

If your custom feature needs to react when core events occur (e.g., triggering SAP synchronization when a Sales Order is completed), subscribe to the Event Bus inside the startup lifespan block inside `lifespan.py`:

```python
# instances/<client_id>/<client_id>/lifespan.py
from core.events import event_bus, BaseEventPayload

async def handle_sales_order_completed(payload: BaseEventPayload):
    # Retrieve details from event payload
    sales_order_id = payload.data.get("sales_order_id")
    # Initiate your custom client-specific background sync
    print(f"Triggering SAP synchronization for sales order: {sales_order_id}")

def register_custom_event_handlers():
    # Dynamically bind custom listener to the core event bus
    event_bus.subscribe("SALES_ORDER_COMPLETED", handle_sales_order_completed)
```
Call `register_custom_event_handlers()` inside `lifespan(app: FastAPI)` during client boot.

---

### Step 5: Implement Client-Specific API Router

Create custom REST endpoints for the client's integration interfaces under `instances/<client_id>/<client_id>/api/`:

```python
# instances/<client_id>/<client_id>/api/endpoints.py
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from core.database import get_async_session
from core.responses import success_response
from core.rbac import require_permission

router = APIRouter(prefix="/api/v1/custom/sap", tags=["Custom SAP Integration"])

@router.post("/trigger-sync", dependencies=[Depends(require_permission("sap_integration:sync:write"))])
async def trigger_sap_sync(
    session: AsyncSession = Depends(get_async_session)
):
    # Integration logic...
    return success_response(data={"message": "SAP sync initiated successfully"})
```

Register this router in the client's `main.py` entry point:
```python
# instances/<client_id>/<client_id>/main.py
from {{client_id}}.api.endpoints import router as sap_router
app.include_router(sap_router)
```

---

### Step 6: Onboard/Rebuild Client Instance

Once your custom files and permissions configurations are structured, run the onboarding CLI to merge your custom permissions and update PDM dependencies:

```bash
cd bes-backend
python scripts/onboard_client.py
```
1. Input your active client ID.
2. Select your subscription package or select "Custom".
3. The script will dynamically read your `custom_permissions.json`, merge it into the active `admin_permissions.json`, and rebuild your dependency profile.

---

### Step 7: Verify

```bash
pdm run uvicorn instances.<client_id>.<client_id>.main:app --reload
# Verify custom DB tables are provisioned in the database schema.
# Verify custom API triggers under Swagger /docs page.
```
