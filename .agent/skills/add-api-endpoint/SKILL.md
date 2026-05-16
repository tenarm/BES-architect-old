---
name: add-api-endpoint
description: This skill walks you through adding a new API endpoint to an existing extension module, following all architecture rules.
---

# Skill: Add a New API Endpoint

This skill walks you through adding a new API endpoint to an existing extension module, following all architecture rules.

---

## Prerequisites
- The extension module already exists with the standard file structure
- You know which entity/resource the endpoint operates on

## Steps

### Step 1: Define the Schema (if new)

Add request/response schemas to `schemas.py`:

```python
# extensions/<module>/schemas.py

class NewEntityCreate(SQLModel):
    """Input schema — only fields the user should provide."""
    name: str
    amount: Decimal = Decimal("0.0")
    # DO NOT include: id, created_at, is_deleted, subsidiary_id

class NewEntityRead(SQLModel):
    """Output schema — what the API returns."""
    id: uuid.UUID
    name: str
    amount: Decimal
    status: str
```

### Step 2: Add Business Logic to Service Layer

Add the function to `services.py`:

```python
# extensions/<module>/services.py

async def do_business_operation(
    session: AsyncSession,
    data: NewEntityCreate
) -> NewEntity:
    """
    Service function with:
    - Input validation
    - Business rules
    - Transaction safety
    """
    # Validate business rules
    if data.amount < 0:
        raise HTTPException(status_code=400, detail="Amount must be positive")
    
    # Create the entity
    db_obj = NewEntity.model_validate(data)
    session.add(db_obj)
    await session.commit()
    await session.refresh(db_obj)
    return db_obj
```

### Step 3: Add the Route to Router

Add the endpoint to `router.py`:

```python
# extensions/<module>/router.py

from .schemas import NewEntityCreate
from .services import do_business_operation

# For LIST endpoints — always include pagination
@router.get("/new-entities")
async def list_new_entities(
    pagination: PaginationParams = Depends(),
    session: AsyncSession = Depends(get_async_session)
):
    count_stmt = select(func.count()).select_from(NewEntity).where(NewEntity.is_deleted == False)
    total = (await session.execute(count_stmt)).scalar() or 0
    
    stmt = (
        select(NewEntity)
        .where(NewEntity.is_deleted == False)
        .offset(pagination.offset)
        .limit(pagination.limit)
    )
    result = await session.execute(stmt)
    items = result.scalars().all()
    
    return paginated_response(data=items, total=total, page=pagination.page, page_size=pagination.page_size)

# For CREATE endpoints — use Create schema
@router.post("/new-entities")
async def create_new_entity(
    data: NewEntityCreate,
    session: AsyncSession = Depends(get_async_session)
):
    item = await do_business_operation(session, data)
    return success_response(data=item)

# For DETAIL endpoints — validate existence
@router.get("/new-entities/{entity_id}")
async def get_new_entity(
    entity_id: uuid.UUID,
    session: AsyncSession = Depends(get_async_session)
):
    entity = await session.get(NewEntity, entity_id)
    if not entity or entity.is_deleted:
        raise HTTPException(status_code=404, detail="Entity not found")
    return success_response(data=entity)
```

### Step 4: Add RBAC (if needed)

Protect the endpoint with `require_permission`:

```python
from core.rbac import require_permission

@router.post("/new-entities")
async def create_new_entity(
    data: NewEntityCreate,
    session: AsyncSession = Depends(get_async_session),
    _: bool = Depends(require_permission("<module>:<resource>:write"))
):
    ...
```

### Step 5: Add Event Emission (if cross-module)

If this action should notify other modules:

```python
# In services.py or router.py
from .events import emit_entity_created

# After successful creation
await emit_entity_created(str(db_obj.id), db_obj.name)
```

### Step 6: Verify

```bash
# Test with curl or the Swagger docs at http://localhost:8000/docs
curl -X GET http://localhost:8000/api/v1/<module>/new-entities \
  -H "Authorization: Bearer <token>"
```

## Checklist

- [ ] Schema defined in `schemas.py` (not using ORM model as input)
- [ ] Business logic in `services.py` (not in router)
- [ ] Pagination on list endpoints
- [ ] `StandardResponse` envelope (`success_response` / `paginated_response`)
- [ ] Soft-delete filtering (`.where(Model.is_deleted == False)`)
- [ ] RBAC protection if needed
- [ ] Event emission if cross-module impact
