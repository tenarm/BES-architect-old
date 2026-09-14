---
name: 1-plan-flow
description: "PLANNING STEP 1: Define a BES business flow — its steps, entities, business rules, state machine, and cross-module events."
---

# Skill: Plan Flow Definition

## Purpose
Given a flow name and description, produce a comprehensive `1-flow-definition.md` document that defines everything about the flow before any UI or backend work begins.

## When to Use
- Starting a new flow (e.g., "Sell", "Buy", "Stock")
- Adding a new workflow to BES
- Run this FIRST in the planning pipeline, BEFORE `2-design-flow-ui`

## Prerequisites
- Read Rule 3 (Flow Architecture) for step types, pipeline model, and flow-to-module mapping.
- Read Rule 4 §4 for what a flow definition document must contain.
- Check `features-plan/common-dependants.md` for existing cross-flow dependencies.

---

## Execution Steps

### Step 1: Understand the Flow
Ask yourself (or the user):
1. **What business problem does this flow solve?** (e.g., "A user wants to sell goods to a customer and get paid")
2. **Who uses it?** (e.g., Sales rep, warehouse staff, accountant)
3. **What's the happy path?** (e.g., Quote → Order → Ship → Invoice → Collect)
4. **What can go wrong?** (e.g., credit limit exceeded, item out of stock, payment overdue)

### Step 2: Define the Default Pipeline
For each step in the flow, define:

```markdown
### Step: [Step Label]
- **ID**: `snake_case_id`
- **Type**: action | approval | automation | notification | decision
- **Entity**: The primary database entity this step creates/modifies
- **Status Event**: `{FLOW}_{STEP}_{ACTION}` (e.g., `SELL_ORDER_CONFIRMED`)
- **Required Fields**: List of fields the user must fill
- **Validations**: Business rules checked before transition
- **Side Effects**: What happens in other modules when this step completes
- **Skippable**: Can the user skip this step? (yes/no)
- **Depends On**: Which step(s) must complete first
```

### Step 3: Define Entities
For each database entity involved in the flow:

```markdown
### Entity: [Entity Name]
- **Table**: `<module>_<entity>s`
- **Module**: Which backend extension owns this entity
- **Key Fields**: List columns with types (mark financial fields as Decimal(20,4))
- **Relationships**: FKs to other entities (core.Customer, core.Product, etc.)
- **Status Field**: The status column with valid values
- **Computed Fields**: Any calculated values (subtotal, tax, total)
```

### Step 4: Define the State Machine
Draw the valid transitions for the primary entity:

```markdown
### State Machine: [Primary Entity]
Draft → Confirmed → Shipped → Invoiced → Paid → Closed
Draft → Cancelled
Confirmed → Cancelled (with reversal)
```

Include:
- Which transitions require approval
- Which transitions trigger automation (inventory reservation, ledger posting)
- Which transitions are irreversible

### Step 5: Map Cross-Module Events
For each event emitted by this flow:

```markdown
### Event: SELL_ORDER_CONFIRMED
- **Emitted by**: sales.services (after commit)
- **Payload**: { order_id, customer_id, line_items[], total_amount }
- **Subscribers**:
  - inventory: reserve stock for line items
  - notifications: alert warehouse team
```

### Step 6: Identify Data Hub Feeds
Which Data Hub entities does this flow populate?
- Customers (new customer created during quote)
- Products (referenced in line items)
- etc.

### Step 7: Business Rules & Calculations
Document domain-specific rules:
- Calculation formulas (subtotal, tax, discount, total)
- Validation constraints (credit limit, stock availability)
- Rounding rules (per Rule 1 §14 — Decimal, quantize at document boundary)

### Step 8: Tier Mapping
- Which steps are Basic tier?
- Which steps are Pro/Premium additions?
- What features are available as Pipeline Editor customizations?

### Step 9: Update Cross-Flow Dependency Registry

After completing the flow definition, update `features-plan/common-dependants.md`.

Append a new `## Flow: <Flow Name>` section (or update the existing one) with two tables:

#### A. Required External Dependencies (Inbound)
List everything this flow needs from other flows or core:

```markdown
| Source Flow/Module | Dependency | Description | Phase |
| :--- | :--- | :--- | :--- |
| Core | core.Customer | Base customer profile for order creation | Phase 1 |
| Stock Flow | ITEM_OUT_OF_STOCK event | Blocks order confirmation if stock is zero | Phase 2 |
```

#### B. Exposed Assets (Outbound)
List everything this flow produces that other flows may consume:

```markdown
| Asset | Consumers | Description | Notes |
| :--- | :--- | :--- | :--- |
| SELL_ORDER_CONFIRMED event | Stock (reserve), Notifications | Emitted after order commit | Payload: order_id, line_items[] |
| sales_orders table | Money Flow (invoicing) | Orders feed into invoice generation | FK: customer_id |
```

**Rules**:
- Source the dependencies and assets from Step 5 (Cross-Module Events) and Step 3 (Entities)
- If the flow depends on another flow that isn't built yet, note the Phase
- Keep entries concise — one row per dependency/asset, not one per field

---

## Output

Write the completed document to:
```
features-plan/flows/<flow_id>/1-flow-definition.md
```

Also update:
```
features-plan/common-dependants.md
```

## Output Format

```markdown
# Flow: [Display Name]

## Overview
- **Flow ID**: `<flow_id>`
- **Display Name**: [Name]
- **Description**: [What this flow does]
- **Icon**: [emoji]
- **Tier**: Basic | Pro | Premium
- **Primary Module**: [e.g., sales]
- **Supporting Modules**: [e.g., inventory, finance]

## Default Pipeline

### Step 1: [Label]
[Step definition per Step 2 above]

### Step 2: [Label]
...

## Entities

### Entity: [Name]
[Entity definition per Step 3 above]

## State Machine
[Per Step 4]

## Cross-Module Events
[Per Step 5]

## Data Hub Feeds
[Per Step 6]

## Business Rules & Calculations
[Per Step 7]

## Tier Mapping
[Per Step 8]

## Open Questions
[Any unresolved design decisions for user review]
```

---

## Quality Checklist
- [ ] Every step has an ID, type, entity, and status event
- [ ] Every entity has a table name, key fields, and status field
- [ ] State machine covers happy path AND error/cancellation paths
- [ ] Cross-module events list both emitter and subscribers
- [ ] Financial calculations use Decimal with 4-decimal precision
- [ ] Business rules reference industry standards where possible (Rule 4 §10)
- [ ] Tier mapping is explicit — no ambiguity on what's Basic vs Pro vs Premium
- [ ] `common-dependants.md` updated with this flow's inbound dependencies and outbound assets
