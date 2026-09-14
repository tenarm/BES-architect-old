# Response Envelope & Exception Management

This document details the architecture and mechanisms used by Bes to unify API response formats and bridge business logic (Domain layer) with HTTP presentation boundaries (Web layer).

---

## 1. The Decoupled Exception Architecture

Following **Rule 14 (Business Service Layer & Domain Rules)**, business logic inside our services layer MUST NOT raise web-specific exceptions like FastAPI `HTTPException` directly. Instead, services raise specialized, decoupled **Domain Exceptions**. 

A global exception mapper catches these domain errors at the boundary of the API router and translates them into appropriate HTTP status codes and response messages.

```
+-------------------------------------------------------------+
|                     Business Service Layer                  |
|  - Validates business rules, calculations, and invariants.  |
|  - Raises: DomainException (ValidationError, etc.)          |
+-------------------------------------------------------------+
                              | (Raises Exception)
                              v
+-------------------------------------------------------------+
|                  FastAPI Global Exception Mapper            |
|  - Catches: DomainException subclasses                      |
|  - Logs details structured (warnings vs errors)             |
|  - Translates: HTTP status codes (e.g. 409, 403, 404)       |
+-------------------------------------------------------------+
                              | (Serializes HTTP Envelope)
                              v
+-------------------------------------------------------------+
|                        REST Client                          |
|  - Receives: StandardResponse JSON ({status, error, ...})   |
+-------------------------------------------------------------+
```

---

## 2. Core Domain Exceptions (`exceptions.py`)

All domain exceptions inherit from `DomainException` and are isolated in [`exceptions.py`](file:///Users/bvk/BVK_Workspace/BES/Bes/backend/core/core/exceptions.py):

| Exception Class | Purpose | Transformed HTTP Status | Code Key |
| :--- | :--- | :--- | :--- |
| **`DomainException`** | Base class for all business validation errors. | `500 Internal Server` (fallback) | `DOMAIN_ERROR` |
| **`ConcurrencyError`** | Thrown when an optimistic lock version conflict or pessimistic update collision is detected. | `409 Conflict` | `CONCURRENCY_CONFLICT` |
| **`ValidationError`** | Thrown when functional/domain data validation fails. | `400 Bad Request` | `VALIDATION_ERROR` |
| **`StateTransitionError`** | Thrown when an illegal status progression is attempted in the state machine matrix. | `400 Bad Request` | `INVALID_TRANSITION` |
| **`EntityNotFoundError`** | Thrown when a record does not exist or has been soft-deleted (`is_deleted = True`). | `404 Not Found` | `NOT_FOUND` |
| **`InsufficientPermissionError`**| Thrown when an active user lacks the required RBAC module permission. | `403 Forbidden` | `PERMISSION_DENIED` |
| **`LicensingError`** | Thrown when a sub-feature or flow is accessed without active package tier licensing. | `403 Forbidden` | `LICENSE_EXPIRED` |

---

## 3. Global Response Envelope (`responses.py`)

To ensure standard format uniformity across the platform (Rule 1 §4), every single API endpoint output—including successful operations, validation failures, and database crashes—is enclosed in the **`StandardResponse`** Pydantic model:

```json
{
  "status": "success",
  "data": { "id": "uuid..." },
  "metadata": { "page": 1 },
  "error": null
}
```

### Utility Wrapper Helpers:
* **`success_response(data, metadata)`**: Packs successful operational returns with optional metadata parameters.
* **`error_response(error_msg)`**: Packs system failures and caught exceptions with clear message signatures.
* **`paginated_response(data, total, page, page_size)`**: Auto-calculates standard paging offsets and page limits.

---

## 4. Custom Starlette/FastAPI Exception Mappings

FastAPI registers unified handlers to map exceptions in [`responses.py`](file:///Users/bvk/BVK_Workspace/BES/Bes/backend/core/core/responses.py). 

### Schema Validation Handling (`RequestValidationError`):
When a client sends structured JSON that violates a Pydantic schema validation boundary (e.g. sending a string instead of an integer), Starlette raises a `RequestValidationError`. 
Bes overrides the default unformatted crash, maps it to a unified `422 Unprocessable Entity` status, and flattens validation trace points for front-end processing:

```json
{
  "status": "error",
  "data": null,
  "metadata": null,
  "error": "Validation Error: [body -> customer_id]: value is not a valid uuid"
}
```
