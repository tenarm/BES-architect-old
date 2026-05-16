# Backend Coding Rules — BES Factory

These rules govern all Python/FastAPI code in the `bes-backend/` directory.

---

## 1. The Money Rule (Decimal Precision)

> **NEVER use `float` for any financial or monetary value.**

- Backend: Use `Decimal` type with `sa_column=Column(Numeric(precision=20, scale=4))`.
- All prices, amounts, balances, quantities, and rates MUST use `Decimal`.
- Import pattern:
  ```python
  from decimal import Decimal
  from sqlalchemy import Column, Numeric
  
  amount: Decimal = Field(
      default=Decimal("0.0"),
      sa_column=Column(Numeric(precision=20, scale=4))
  )
  ```

## 2. Base Model Inheritance

- ALL database tables MUST inherit from `core.models.BESBase`.
- This provides: UUID primary key, audit trail (`created_at`, `updated_at`), `subsidiary_id`, `is_deleted`, and `metadata_`.
- The `updated_at` field auto-updates via SQLAlchemy `before_update` event.
- `BESBase` gives you soft-delete capability via `BaseRepository.delete()`.

## 3. Soft Delete

- **Physical deletion is FORBIDDEN** for any business data.
- Use `is_deleted = True` flag via `BaseRepository.delete()`.
- All queries MUST filter by `is_deleted == False` (auto-applied by `BaseRepository._apply_scopes()`).
- If building custom queries, always include `.where(Model.is_deleted == False)`.

## 4. Layer Separation

Every extension module MUST separate concerns into layers:

| Layer | File | Responsibility |
|:------|:-----|:---------------|
| **Models** | `models.py` | Database table definitions ONLY |
| **Schemas** | `schemas.py` | API input/output validation (Pydantic) |
| **Services** | `services.py` | Business logic, validation, transactions |
| **Router** | `router.py` | HTTP handling (thin — calls services) |
| **Events** | `events.py` | Event bus subscribers and emitters |

### Router Rules
- Routers MUST NOT contain business logic.
- Routers call service functions and return the result.
- Never use ORM models as API input — always use `*Create` schemas.
- All list endpoints MUST use `PaginationParams` dependency.

### Service Rules
- Services receive a `session: AsyncSession` and schema/data objects.
- Services handle validation, cross-table operations, and transaction safety.
- Services raise `HTTPException` for business rule violations.

## 5. API Response Envelope

ALL API responses MUST use the `StandardResponse` envelope:

```python
from core.responses import success_response, paginated_response, error_response

# Single item
return success_response(data=item)

# Paginated list
return paginated_response(data=items, total=count, page=page, page_size=size)

# Error
return error_response("Something went wrong")
```

## 6. Pagination

- ALL list endpoints MUST support pagination.
- Use `core.pagination.PaginationParams` as a FastAPI dependency.
- Maximum page size is capped at 200 items.
- Return total count in metadata for UI pagination.

```python
from core.pagination import PaginationParams

@router.get("/items")
async def list_items(
    pagination: PaginationParams = Depends(),
    session: AsyncSession = Depends(get_async_session)
):
    # Count + paginated query
```

## 7. Import Ordering

Follow this import order in all Python files:

```python
# 1. Standard library
import os
import uuid
from datetime import datetime
from decimal import Decimal

# 2. Third-party
from fastapi import APIRouter, Depends
from sqlmodel import select
from sqlalchemy.ext.asyncio import AsyncSession

# 3. Core package
from core.database import get_async_session
from core.responses import success_response
from core.pagination import PaginationParams

# 4. Current module
from .models import MyModel
from .schemas import MyCreate
from .services import my_service_function
```

## 8. Error Handling

- Use `HTTPException` for client-facing errors (400, 404, etc.).
- Use the global exception handler for unexpected server errors.
- Never expose internal error details to the client in production.
- Log all errors with proper context using `logger.error(..., exc_info=True)`.

## 9. Environment Variables

- **NEVER hardcode secrets** (JWT keys, database passwords, API keys).
- Use `os.environ["KEY"]` (fail-fast) for required secrets.
- Use `os.getenv("KEY", "default")` only for non-sensitive configuration.
- Document all required env vars in the instance's README.
