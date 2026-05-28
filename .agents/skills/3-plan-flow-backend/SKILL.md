---
name: 3-plan-flow-backend
description: "PLANNING STEP 3: Design the backend implementation for a TenArm flow — models, schemas, services, APIs, events, and cross-module integration."
---

# Skill: Plan Flow Backend

## Purpose
Given a completed `1-flow-definition.md` and `2-flow-ui-design.md`, produce a comprehensive `3-flow-backend-plan.md` that specifies every model, schema, service method, API endpoint, and event needed to power the flow.

## When to Use
- After `2-design-flow-ui` has produced the UI design
- Run THIRD in the planning pipeline, BEFORE `4-review-flow`

## Prerequisites
- Read the flow's `1-flow-definition.md` and `2-flow-ui-design.md`.
- Read Rule 1 (Backend Architecture) — especially §2 (layering), §3 (database), §14 (service rules), §18 (flow API standards).
- Read Rule 3 §7 (flow-to-module mapping) and §10 (flow event lifecycle).
- Check existing models in the backend modules this flow touches (`bes-backend/extensions/`).
- Check `core/core/models.py` for master data entities (Customer, Vendor, Product, UOM) the flow references.

---

## Execution Steps

### Step 1: Map Entities to Backend Modules

For every entity in the flow definition, determine which backend module owns it:

```markdown
## Module Mapping

| Entity | Backend Module | Table Name | New or Existing? |
|:-------|:--------------|:-----------|:-----------------|
| Quotation | sales | sales_quotations | New |
| Sales Order | sales | sales_orders | New |
| Sales Order Line | sales | sales_order_lines | New |
| Shipment | sales | sales_shipments | New |
| Invoice | sales | sales_invoices | New |
| Customer | core | customers | Existing |
| Product | core | products | Existing |
| Stock Movement | inventory | inventory_movements | New |
```

### Step 2: Design Database Models

For each NEW entity, write the complete model specification:

```markdown
### Model: SalesOrder

**Table**: `sales_orders`
**Module**: `extensions/sales/sales/models.py`
**Inherits**: `BESBase`

| Column | Type | Constraints | Notes |
|:-------|:-----|:-----------|:------|
| ref_number | str | unique, indexed | Auto-generated (SO-XXXX) |
| customer_id | UUID | FK → customers.id | Ghost FK via metadata_ if cross-module |
| status | str | indexed | Draft, Confirmed, Shipped, Invoiced, Paid, Cancelled |
| order_date | date | not null | |
| payment_terms | str | default "Net 30" | |
| subtotal | Decimal(20,4) | | Sum of line items |
| tax_amount | Decimal(20,4) | | Calculated from tax profile |
| total_amount | Decimal(20,4) | | subtotal + tax_amount |
| notes | str | nullable | |
| quotation_id | UUID | nullable | FK → sales_quotations.id (if converted from quote) |

**Relationships**:
- Has many: SalesOrderLine (via order_id FK)
- Belongs to: Customer (via customer_id)
- Optionally created from: Quotation (via quotation_id)
```

**Rules to follow**:
- ALL financial columns use `sa_column=Column(Numeric(precision=20, scale=4))`
- Status field MUST exist with defined valid values matching the state machine
- Use FKs within the same module. Use Ghost FKs (metadata_ or runtime validation) for cross-module references.
- Inherit from BESBase (provides id, created_at, updated_at, created_by, is_deleted, subsidiary_id, metadata_, version_id)

### Step 3: Design Pydantic Schemas

For each entity, define the Create and Read schemas:

```markdown
### Schemas: SalesOrder

#### SalesOrderCreate
| Field | Type | Required | Validation |
|:------|:-----|:---------|:-----------|
| customer_id | UUID | Yes | Must reference existing customer |
| order_date | date | Yes | Cannot be in the past |
| payment_terms | str | No | Default "Net 30" |
| line_items | list[SalesOrderLineCreate] | Yes | Min 1 item |
| notes | str | No | Max 500 chars |

#### SalesOrderRead
[All model fields + computed fields + nested relationships]
- Include: `customer_name` (denormalized for display)
- Include: `line_items` (nested SalesOrderLineRead[])
- Include: `total_amount` (Decimal as string for JSON safety)
```

### Step 4: Design Service Methods

For each flow step, define the service method:

```markdown
### Service: SalesOrderService

#### create_order(session, data: SalesOrderCreate) → SalesOrderRead
1. Validate customer exists (query core.Customer)
2. Validate all product IDs exist (query core.Product)
3. Calculate line subtotals (qty × unit_price, Decimal precision)
4. Calculate order subtotal, tax, total
5. Generate ref_number (next in sequence)
6. Create SalesOrder with status = "draft"
7. Create SalesOrderLine records
8. Commit transaction
9. Emit SELL_ORDER_CREATED event (post-commit)

**Domain Exceptions**:
- `CustomerNotFoundError` — if customer_id doesn't exist
- `ProductNotFoundError` — if any line item product doesn't exist
- `EmptyOrderError` — if no line items provided

#### confirm_order(session, order_id: UUID, version_id: int) → SalesOrderRead
1. Load order with optimistic lock check (version_id)
2. Validate status transition: Draft → Confirmed
3. Update status = "confirmed"
4. Commit transaction
5. Emit SELL_ORDER_CONFIRMED event (post-commit)
   - Subscribers: inventory (reserve stock), notifications (alert warehouse)

**Domain Exceptions**:
- `InvalidTransitionError` — if current status doesn't allow confirmation
- `ConcurrencyError` — if version_id mismatch (handled by BESBase)
```

**Rules to follow**:
- Services MUST NOT raise HTTPException (Rule 1 §14)
- Services own the transaction boundary — only services call commit
- Events emitted AFTER commit (Rule 1 §14)
- Financial calculations use Decimal with max precision, quantize at document boundary

### Step 5: Design API Endpoints

For each service method, define the router endpoint:

```markdown
### Router: sales/router.py

#### POST /api/v1/sales/orders
- **Permission**: `sales:orders:write`
- **Body**: SalesOrderCreate
- **Response**: StandardResponse(data=SalesOrderRead)
- **Errors**: 
  - 422 → validation errors
  - 404 → customer/product not found (catches domain exception)
  - 403 → permission denied

#### PATCH /api/v1/sales/orders/{order_id}/confirm
- **Permission**: `sales:orders:write`
- **Query**: `version_id` (required)
- **Response**: StandardResponse(data=SalesOrderRead)
- **Errors**:
  - 409 → concurrency conflict
  - 400 → invalid status transition

#### GET /api/v1/sales/orders
- **Permission**: `sales:orders:read`
- **Query**: PaginationParams + status filter + customer_id filter + date_range
- **Response**: StandardResponse(data=list[SalesOrderRead], metadata=pagination)
```

### Step 6: Design Events

List all events emitted by this flow and their expected subscribers:

```markdown
## Event Registry

| Event | Emitted By | Payload | Subscribers |
|:------|:----------|:--------|:-----------|
| SELL_QUOTE_CREATED | sales.services | { quotation_id, customer_id } | notifications |
| SELL_ORDER_CONFIRMED | sales.services | { order_id, customer_id, line_items[], total } | inventory (reserve), notifications |
| SELL_SHIPMENT_DISPATCHED | sales.services | { shipment_id, order_id, items[] } | inventory (deduct), notifications |
| SELL_INVOICE_GENERATED | sales.services | { invoice_id, order_id, total } | finance (journal entry), notifications |
| SELL_PAYMENT_COLLECTED | sales.services | { payment_id, invoice_id, amount } | finance (receipt), notifications |
```

### Step 7: Identify Shared Dependencies

What from other modules does this flow need that doesn't exist yet?

```markdown
## Dependencies

### Required (must exist before building this flow)
| Dependency | Module | Status |
|:-----------|:-------|:-------|
| Customer CRUD API | core | ✅ Exists |
| Product CRUD API | core | ✅ Exists |
| Number sequence service | settings | ⚠️ Needs building |
| Inventory reservation service | inventory | ⚠️ Needs building |

### Stubs Needed
If a dependency doesn't exist yet, define the stub:
- `inventory.reserve_stock(product_id, qty)` → no-op, returns success
- `settings.next_sequence("SO")` → returns "SO-0001" (in-memory counter)
```

---

## Output

Write the completed document to:
```
features-plan/flows/<flow_id>/3-flow-backend-plan.md
```

## Quality Checklist
- [ ] Every entity mapped to a backend module with table name
- [ ] Every model has all fields with types and constraints
- [ ] Financial fields use Decimal(20,4) — no floats
- [ ] Every service method documents: inputs, logic steps, domain exceptions, events emitted
- [ ] Services don't raise HTTPException — only domain exceptions
- [ ] Every API endpoint has permission, body/query params, response format, and error codes
- [ ] Events follow `{FLOW}_{STEP}_{ACTION}` naming convention
- [ ] Cross-module dependencies identified with existing/stub status
- [ ] State machine transitions enforced in service layer
- [ ] No direct imports between extension modules — all cross-module via events
