# Database Connection & Pagination Management

This document outlines the architecture, caching strategies, and pagination mechanics used in TenArm to manage database connections and handle request-scoped paginated queries.

---

## 1. Connection Caching & Lifecycle (`database.py`)

To optimize resource allocation and prevent connection leaks, connection engines and session makers are **cached globally** by their database connection string:

```
                  +-----------------------------+
                  |  get_async_session() call   |
                  +-----------------------------+
                                 |
                                 v
                  +-----------------------------+
                  |    get_session_maker()      |
                  +-----------------------------+
                                 |
           Yes (Cached)          v           No (Cold Boot)
         +-----------------------+-----------------------+
         |                                               |
         v                                               v
+-------------------------------+             +---------------------+
| Retrieve cached sessionmaker  |             |  get_engine() call  |
| from global _session_makers   |             +---------------------+
+-------------------------------+                        |
                                           Yes (Cached)  v   No (New URL)
                                         +---------------+---------------+
                                         |                               |
                                         v                               v
                                +-------------------+         +---------------------+
                                | Retrieve engine   |         | create_async_engine |
                                | from _engines     |         | Cache engine & URL  |
                                +-------------------+         +---------------------+
```

### Critical APIs & Caching:
* **`get_engine(database_url: str | None)`**: Fetches a cached async SQLAlchemy engine. If the connection URL is unique (e.g. multi-tenant distinct databases), a new engine is established, cached, and returned. 
  * The environment variable `SQL_ECHO` controls whether raw SQL statements are output to standard logs.
* **`get_session_maker(eng: AsyncEngine | None)`**: Returns a cached `sessionmaker` bound to the active engine. Constructing a sessionmaker on every request adds overhead; caching it dramatically speeds up request startup latency.
* **`get_async_session()`**: A standard FastAPI async generator dependency that yields a scoped database transaction session (`AsyncSession`) and guarantees automatic closing/cleanup on request completion.

---

## 2. Reusable Pagination Dependency (`pagination.py`)

List endpoints utilize standard, request-scoped pagination parameters configured via **`PaginationParams`**. 

To adhere to **Rule 1 §4**, a maximum limit is enforced to prevent runaway database memory consumption.

### Dynamic Limits Configuration:
* The system enforces a default hard limit of **200 items per page** (`MAX_PAGE_SIZE`).
* This limit is dynamic and can be customized per tenant or environment using the `MAX_PAGE_SIZE` environment variable without requiring any codebase modifications.

### API Mechanics:
The dependency exposes four primary properties derived dynamically from client query parameters:
* **`page`**: 1-indexed page identifier (e.g. `?page=2`).
* **`page_size`**: Paging quantity limit (e.g. `?page_size=50`), capped automatically at `MAX_PAGE_SIZE`.
* **`offset`**: Calculated SQL query offset, matching `(page - 1) * page_size` (e.g. `.offset(50)`).
* **`limit`**: Calculated SQL query limit, matching `page_size` (e.g. `.limit(50)`).

### Controller Implementation Example:
```python
from fastapi import Depends
from sqlmodel import select
from core.database import get_async_session
from core.pagination import PaginationParams

@router.get("/items")
async def list_items(
    pagination: PaginationParams = Depends(),
    session: AsyncSession = Depends(get_async_session)
):
    stmt = select(Item).offset(pagination.offset).limit(pagination.limit)
    results = await session.execute(stmt)
    ...
```
