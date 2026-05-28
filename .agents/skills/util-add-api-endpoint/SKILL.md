---
name: add-api-endpoint
description: This skill walks you through adding a new API endpoint to an existing extension module, following all architecture rules.
---

# Skill: Add a New API Endpoint

This skill walks you through adding a new API endpoint to an existing extension module, following all architecture rules.

---

## Prerequisites
- The extension module already exists with the standard file structure (either flat files or modular package directories)
- You know which entity/resource the endpoint operates on

## Steps

### Step 1: Define the Schema (if new)

Add request/response schemas to `schemas.py` (or the appropriate file inside the `schemas/` package if modularized):

```python
# extensions/<module>/schemas.py (or schemas/<entity>.py)

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

Add the function to `services.py` (or the appropriate file inside the `services/` package if modularized).

> **IMPORTANT**: Per Rule 1 §14, services MUST NOT raise `HTTPException`. Raise domain-specific exceptions instead.

```python
# extensions/<module>/services.py (or services/<entity>.py)

# Define domain exceptions at top of file (or in a shared exceptions.py)
class InvalidAmountError(Exception):
    """Raised when a financial amount violates business rules."""
    pass

class EntityNotFoundError(Exception):
    """Raised when a requested entity does not exist."""
    pass

async def do_business_operation(
    session: AsyncSession,
    data: NewEntityCreate
) -> NewEntity:
    """
    Service function with:
    - Input validation via domain exceptions
    - Business rules
    - Transaction safety (service owns the commit)
    """
    # Validate business rules — raise domain exceptions, NOT HTTPException
    if data.amount < 0:
        raise InvalidAmountError("Amount must be positive")
    
    # Create the entity
    db_obj = NewEntity.model_validate(data)
    session.add(db_obj)
    await session.commit()
    await session.refresh(db_obj)
    return db_obj
```

### Step 3: Add the Route to Router

Add the endpoint to `router.py`. The router catches domain exceptions and translates them to HTTP responses:

```python
# extensions/<module>/router.py (or router/<entity>.py)

from .schemas import NewEntityCreate
from .services import do_business_operation, InvalidAmountError, EntityNotFoundError

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

# For CREATE endpoints — catch domain exceptions in router
@router.post("/new-entities")
async def create_new_entity(
    data: NewEntityCreate,
    session: AsyncSession = Depends(get_async_session)
):
    try:
        item = await do_business_operation(session, data)
    except InvalidAmountError as e:
        raise HTTPException(status_code=400, detail=str(e))
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

### Step 5: Add Feature-Level Licensing Checks

Always assert feature eligibility to enforce subscription packaging tier restrictions:

```python
from core.licensing import require_licensed_feature

@router.post("/new-entities")
async def create_new_entity(
    data: NewEntityCreate,
    session: AsyncSession = Depends(get_async_session)
):
    # Enforce granular subscription tier licensing check
    require_licensed_feature("<module>", "<subfeature>")
    
    item = await do_business_operation(session, data)
    return success_response(data=item)
```

### Step 6: Add Event Emission (if cross-module)

If this action should notify other modules:

```python
# In services.py or router.py
from .events import emit_entity_created

# After successful creation
await emit_entity_created(str(db_obj.id), db_obj.name)
```

### Step 7: Verify

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
- [ ] Granular feature-level licensing checked via `require_licensed_feature`
- [ ] Event emission if cross-module impact

