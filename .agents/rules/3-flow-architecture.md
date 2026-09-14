---
trigger: always_on
description: Flow-first architecture, pipeline customization, step types, Data Hub patterns, and flow-to-module mapping governing BES's core business model.
---

# 3. Flow Architecture — BES

These rules govern how business flows are defined, structured, customized, and rendered across the BES ERP platform.

---

## 1. Flow-First Business Model

BES is a **flow-centric** ERP. Users navigate by **what they want to do** (Sell, Buy, Track Stock), not by **which module contains the data** (Sales, Inventory, Finance).

- **A Flow** is a guided multi-step business process that creates or modifies entities across one or more backend modules.
- **The Data Hub** is a secondary browsing layer for flat entity views (Customers, Suppliers, Products).
- **Backend modules** (extensions) still own their data and APIs. The flow is a **frontend orchestration concept** that calls APIs from multiple modules in sequence.

### The Navigation Hierarchy

```
PRIMARY:   Flows       → task-centric (what I DO)
SECONDARY: Data Hub    → entity-centric (what I HAVE)
SYSTEM:    Settings    → configuration (how it WORKS)
```

---

## 2. Standard Flows

Every flow ships with a **default pipeline** (standard steps) that the business can customize.

| # | Flow ID | Display Name | Default Steps | Backend Modules | Tier |
|:--|:--------|:-------------|:--------------|:----------------|:-----|
| 1 | `sell` | Sell | Quote → Confirm Order → Ship → Invoice → Collect Payment | sales, inventory, finance | Basic |
| 2 | `buy` | Buy | Purchase Request → Send PO → Receive Goods → Match Invoice → Pay | supply_chain, inventory, finance | Basic |
| 3 | `stock` | Stock | Count → Adjust → Transfer → Reorder | inventory | Basic |
| 4 | `money` | Money | Record Transaction → Categorize → Reconcile → Close Period | finance | Pro |
| 5 | `people` | People | Post Opening → Onboard → Attendance → Leave → Payroll | hr | Pro |
| 6 | `customers` | Customers | Capture Lead → Qualify → Propose → Negotiate → Close Deal | crm | Pro |
| 7 | `manufacture` | Manufacture | Plan BOM → Production Order → Execute → QC → Complete | manufacturing, quality | Premium |
| 8 | `projects` | Projects | Plan → Allocate → Track Time → Bill Milestones | project_management | Premium |
| 9 | `assets` | Assets | Register → Schedule Maintenance → Execute → Log | asset_management | Premium |
| 10 | `support` | Support | Receive Ticket → Assign → Resolve → Close → SLA Report | service_desk | Premium |

> New flows can be added without modifying existing ones. Each flow is independently defined, independently customizable.

---

## 3. Flow Step Model

Every step in a flow pipeline has a defined type and structure.

### Step Types

| Type | Description | Example |
|:-----|:-----------|:--------|
| `action` | A user-driven screen where data is entered or modified. The core building block. | "Create Quote", "Enter Line Items" |
| `approval` | A gate requiring one or more users/roles to approve before proceeding. The flow pauses here. | "Manager Approval for PO > $5K" |
| `automation` | A system-triggered step that runs without user input (API call, calculation, auto-posting). | "Auto-post journal entry", "Send email" |
| `notification` | An alert sent to specific users or roles when this step is reached. Does not block the flow. | "Notify warehouse team on ship confirmation" |
| `decision` | A conditional branch where the next step depends on data (if/else routing). | "If credit limit exceeded → route to credit review" |

### Step Schema

```json
{
  "id": "confirm_order",
  "label": "Confirm Order",
  "type": "action",
  "description": "Review and confirm the sales order",
  "statusEvent": "ORDER_CONFIRMED",
  "entity": "sales_order",
  "ownedBy": { "type": "role", "target": "sales_rep" },
  "requiredFields": ["customer_id", "line_items", "payment_terms"],
  "validations": [
    { "rule": "line_items.length > 0", "message": "At least one line item required" },
    { "rule": "total_amount > 0", "message": "Order total must be positive" }
  ],
  "sideEffects": [
    { "type": "automation", "action": "reserve_inventory" },
    { "type": "notification", "target": "warehouse_team", "template": "new_order_confirmed" }
  ],
  "permissions": ["sales:orders:write"],
  "dependsOn": ["create_quote"]
}
```

### Multi-Role Flow Execution

A single flow is **NOT executed by a single person**. Different steps are owned by different roles. This is fundamental to how real businesses work:

```
Sell Flow — Role Ownership:
─────────────────────────────────────────────────────
Step              │ Owned By        │ Permission
─────────────────────────────────────────────────────
Create Quote      │ Sales Rep       │ sales:quotes:write
Confirm Order     │ Sales Rep       │ sales:orders:write
Ship              │ Warehouse Staff │ inventory:shipments:write
Generate Invoice  │ Accountant      │ finance:invoices:write
Collect Payment   │ Accountant      │ finance:payments:write
─────────────────────────────────────────────────────
```

#### Step Ownership (`ownedBy`)
Every step defines WHO is responsible for executing it:

```json
"ownedBy": {
  "type": "role",          // "role" | "user" | "initiator" | "dynamic"
  "target": "warehouse_staff"
}
```

- `role` — assigned to anyone with the named role (e.g., `warehouse_staff`, `accountant`)
- `user` — assigned to a specific user ID
- `initiator` — the person who started this flow instance
- `dynamic` — determined at runtime by a field on the entity (e.g., the assigned sales rep on the order)

#### Handoff Mechanics
When a step completes and the **next step has a different owner**:
1. The current step's status event fires (e.g., `SELL_ORDER_CONFIRMED`)
2. A **task is created** in the `flow_tasks` table for the next step's owner
3. A **notification** is sent to the next owner: "Sales Order SO-0042 is ready for shipping"
4. The flow instance's `current_step` advances, but the next step shows as **"Awaiting [Role]"** until the owner acts
5. The new owner sees this task in their **My Tasks** inbox

#### Flow Task Data Model
```
flow_tasks
├── id (UUID)
├── flow_id (string, e.g., "sell")
├── flow_instance_id (UUID — the specific order/quote being processed)
├── step_id (string, e.g., "ship")
├── entity_type (string, e.g., "sales_order")
├── entity_id (UUID — the specific order)
├── assigned_to_role (string, nullable — e.g., "warehouse_staff")
├── assigned_to_user (UUID, nullable — specific user)
├── status ("pending" | "in_progress" | "completed" | "skipped")
├── priority ("normal" | "high" | "urgent")
├── due_date (datetime, nullable)
├── completed_by (UUID, nullable)
├── completed_at (datetime, nullable)
├── created_at, updated_at
```

#### My Tasks (Inbox)
Every user has a **My Tasks** view accessible from the Home dashboard:

```
┌──────────────────────────────────────────────────┐
│  My Tasks                              [3 pending]│
│──────────────────────────────────────────────────│
│  🔴 URGENT  Ship Order SO-0042                    │
│     Sell Flow → Ship step                         │
│     Customer: Apex Technologies                   │
│     Assigned: 2 hours ago                         │
│                                                    │
│  ○  Review Invoice INV-0038                       │
│     Sell Flow → Invoice step                      │
│     Customer: GreenLeaf Co                        │
│     Assigned: 1 day ago                           │
│                                                    │
│  ○  Approve PO PO-0015 ($12,400)                  │
│     Buy Flow → Approval step                     │
│     Supplier: NexGen Supply                       │
│     Requires: Manager approval (amount > $10K)    │
└──────────────────────────────────────────────────┘
```

Clicking a task navigates directly to the flow step view for that entity, pre-loaded with the entity data.

#### Rules
- Every `action` and `approval` step MUST define an `ownedBy` field.
- `automation` and `notification` steps do NOT have owners (they're system-triggered).
- When step ownership changes between consecutive steps, a handoff notification MUST be sent.
- Unassigned tasks (role-based) are visible to ALL users with that role — first to claim it owns it.
- The My Tasks view is role-scoped: users only see tasks assigned to their role or directly to them.
- Tasks have an optional `due_date` calculated from SLA rules or flow configuration.
- Overdue tasks surface with `urgent` priority and trigger escalation notifications.

### Pipeline Customization

- **Default Pipeline**: Shipped with BES. Defined in `flow_definitions/<flow_id>.json`.
- **Tenant Pipeline**: Per-tenant overrides stored in the database (`flow_pipeline_overrides` table). The user's customization is a delta on top of the default.
- **Pipeline Editor**: A visual drag-and-drop editor in Settings → Pipelines where business owners can:
  - **Add** approval gates, notification steps, automation hooks
  - **Skip** steps they don't use (marked as `skippable: true` in the default)
  - **Reorder** steps within business-logic constraints (`dependsOn` enforced)
  - **Set conditions** for decision steps ("if amount > X, require approval")

### Conditional Step Logic (`decision` type)

Decision steps evaluate conditions at runtime to determine the next step:

```json
{
  "id": "credit_check",
  "type": "decision",
  "label": "Credit Check",
  "conditions": [
    {
      "if": { "field": "total_amount", "operator": ">", "value": 10000 },
      "then": "manager_approval"
    },
    {
      "if": { "field": "customer.credit_status", "operator": "==", "value": "flagged" },
      "then": "credit_review"
    }
  ],
  "default": "confirm_order"
}
```

**Supported operators**: `==`, `!=`, `>`, `<`, `>=`, `<=`, `in`, `not_in`, `is_empty`, `is_not_empty`.
**Field paths** support dot notation to traverse entity relationships (e.g., `customer.credit_limit`).
Conditions are evaluated top-to-bottom; first match wins. `default` is the fallback.

### Dynamic Approval Chains

Approval steps support threshold-based routing via conditions:

```json
{
  "id": "po_approval",
  "type": "approval",
  "label": "PO Approval",
  "approvalRules": [
    { "condition": { "field": "total_amount", "operator": "<=", "value": 5000 }, "approvedBy": { "type": "role", "target": "team_lead" } },
    { "condition": { "field": "total_amount", "operator": "<=", "value": 50000 }, "approvedBy": { "type": "role", "target": "manager" } },
    { "condition": { "field": "total_amount", "operator": ">", "value": 50000 }, "approvedBy": { "type": "role", "target": "director" } }
  ]
}
```

The backend evaluates the entity's data against `approvalRules` to determine the correct approver. This is configurable per tenant via the Pipeline Editor.

### Dynamic Step Rendering (Frontend)

Not all steps have hardcoded React components. Custom steps added via the Pipeline Editor need to render dynamically:

1. **Standard steps** (shipped with BES) have dedicated components in `libs/flows/<flow-id>/steps/`.
2. **Custom steps** (tenant-added via Pipeline Editor) are rendered by a **GenericStepRenderer** component in `@BES/shared-ui`.
3. The GenericStepRenderer reads the step's schema (`requiredFields`, `validations`, field types) and builds a form using `FormSection` components dynamically.
4. **Component override**: If a tenant's client instance needs a fully custom UI for a step, they can register a component in `ComponentRegistry` with key `Step_<flow_id>_<step_id>`. The flow engine checks for a registered override before falling back to GenericStepRenderer.

**Resolution order for step rendering**:
```
1. ComponentRegistry.get(`Step_sell_custom_approval`)  → client-specific override
2. ComponentRegistry.get(`Step_sell_confirm_order`)     → standard flow component
3. <GenericStepRenderer step={stepSchema} />             → schema-driven fallback
```

---

## 4. Flow Page Architecture (Frontend)

### Flow Landing Page

Every flow has a **landing page** as its entry point. The landing page shows:

1. **My Pending Tasks** — tasks in THIS flow assigned to the current user's role. This is the FIRST thing they see — "what do I need to act on?"
2. **Recent Activity Feed** — combined timeline of all entities in this flow (recent quotes, orders, invoices) sorted by date, showing status progression.
3. **Start New [Flow]** — prominent CTA button in the top-right that launches the flow's first step.
4. **Summary Metrics** — 3-4 KPI cards relevant to the flow (e.g., Sell: Open Quotes, Pending Orders, This Month's Revenue, Overdue Invoices).
5. **Quick Filters** — tab-like filters to view entities by status (All, Draft, Active, Completed, Overdue).

### Flow Step View

When a user starts a flow or continues one, they enter the **Step View**:

```
┌──────────────────────────────────────────────┐
│  FlowStepper (horizontal progress bar)        │
│  [Quote ●] → [Confirm ○] → [Ship ○] → ...    │
│──────────────────────────────────────────────│
│                                                │
│  Step Content Area                             │
│  ┌────────────────────────────────────────┐   │
│  │  FormSection / DataTable / Custom View  │   │
│  │  (step-specific UI)                     │   │
│  └────────────────────────────────────────┘   │
│                                                │
│  ┌─────────────────────────────────────────┐  │
│  │  [← Previous]          [Next Step →]     │  │
│  └─────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

### Flow Step Transitions

- Moving between steps validates required fields and fires `statusEvent`.
- Steps with `type: approval` show an "Awaiting Approval" state and block the Next button until approved.
- Steps with `type: automation` auto-execute and show a loading state with the result.
- Each transition emits an event on the backend event bus for cross-module side effects.

---

## 5. Data Hub Architecture

The Data Hub provides flat, browsable views of master data entities. Every Data Hub page follows the same pattern:

### Standard Data Hub Layout

```
┌──────────────────────────────────────────────┐
│  [Search...]  [Filters ▾]  [+ New Entity]     │
│──────────────────────────────────────────────│
│  DataTable                                    │
│  ┌──────────────────────────────────────┐    │
│  │ Name      │ Status │ Balance │ Date ▾│    │
│  │ ──────────┼────────┼─────────┼───────│    │
│  │ Apex Tech │ Active │ $12,400 │ May 2 │    │  ← Row click opens DetailPanel
│  │ GreenLeaf │ Active │  $3,200 │ Apr 1 │    │
│  └──────────────────────────────────────┘    │
│                                               │
│  [← Prev]  Page 1 of 5  [Next →]             │
└──────────────────────────────────────────────┘
```

Clicking a row opens the **DetailPanel** (slide-over from right):

```
┌───────────────────────────┐
│  × Close                  │
│  ─────────────────────────│
│  Entity Name              │
│  Status: [Active ●]       │
│  ─────────────────────────│
│  [Overview] [Txns] [Log]  │  ← Tabs
│  ─────────────────────────│
│  Field: Value             │
│  Field: Value             │
│  ...                      │
│  ─────────────────────────│
│  [Edit]  [Archive]        │
└───────────────────────────┘
```

### Data Hub Entities

| Entity | Source Module(s) | Key Columns | Detail Tabs |
|:-------|:----------------|:------------|:------------|
| Customers | core.Customer + sales.SalesCustomerDetails | Name, credit limit, balance, last order | Overview, Orders, Invoices, Timeline |
| Suppliers | core.Vendor + supply_chain details | Name, payment terms, open POs | Overview, POs, Bills, Timeline |
| Products | core.Product + inventory | Name, SKU, on-hand, cost, sale price | Overview, Stock Levels, Movements, Timeline |
| Accounts | finance.Account | Code, name, type, balance | Overview, Journal Entries, Timeline |
| Employees | hr.Employee | Name, department, status | Overview, Attendance, Leave, Timeline |
| Contacts | crm.Contact | Name, company, last interaction | Overview, Activities, Deals, Timeline |

### Data Hub Rules
- Every Data Hub page MUST use the shared `DataTable` component.
- Every row click MUST open the shared `DetailPanel` component.
- Every Data Hub page MUST support pagination, search, and at least one filter.
- Detail panels MUST have at minimum two tabs: Overview and Timeline.
- The "Timeline" tab shows an audit log of all changes to the entity across all flows.
- Data Hub pages MUST NOT contain flow logic. They are read/edit views only. Creating new entities from the Data Hub is allowed (simple form), but multi-step flows are accessed from WORKFLOWS.

---

## 6. Sidebar Navigation Structure

```
━━━━━━━━━━━━━━━━━━━━━━━
  BES
━━━━━━━━━━━━━━━━━━━━━━━
 🏠 Home
 📋 My Tasks              ← Cross-flow inbox (pending tasks for current user)
━━━━━━━━━━━━━━━━━━━━━━━
 WORKFLOWS
   💰 Sell
   🛒 Buy
   📦 Stock
   💳 Money
   👥 People           🔒
   🤝 Customers        🔒
   🏭 Manufacture      🔒
   📋 Projects         🔒
━━━━━━━━━━━━━━━━━━━━━━━
 DATA HUB
   Customers
   Suppliers
   Products
   Accounts
   Employees           🔒
━━━━━━━━━━━━━━━━━━━━━━━
 SYSTEM
   ⚙️ Settings
   🔧 Pipelines
━━━━━━━━━━━━━━━━━━━━━━━
```

### Sidebar Rules
- Section headers (WORKFLOWS, DATA HUB, SYSTEM) are non-clickable labels.
- Each item is a direct link — no nested sub-items, no expandable trees.
- **My Tasks** is always visible and shows a badge count of pending tasks.
- Locked items show a 🔒 indicator. Clicking them opens the Upgrade overlay.
- Active item is visually highlighted with the accent color (`--wp-accent`).
- The sidebar is collapsible to icon-only mode.
- DATA HUB items are always visible (not hidden behind flows).
- The sidebar order is fixed: Home + My Tasks, then Workflows, Data Hub, System.

---

## 7. Flow-to-Module Mapping

A single flow touches multiple backend modules. This mapping is critical for understanding which APIs a flow needs:

| Flow | Backend Modules Used | Primary Entity | Supporting Entities |
|:-----|:--------------------|:---------------|:-------------------|
| Sell | `sales`, `inventory`, `finance` | Sales Order | Customer, Product, Invoice, Payment |
| Buy | `supply_chain`, `inventory`, `finance` | Purchase Order | Supplier, Product, GRN, Bill |
| Stock | `inventory` | Stock Movement | Product, Warehouse, Adjustment |
| Money | `finance` | Journal Entry | Account, Bank, Reconciliation |
| People | `hr` | Employee | Department, Attendance, Leave, Payroll |
| Customers | `crm` | Lead/Deal | Contact, Account, Activity |
| Manufacture | `manufacturing`, `quality` | Production Order | BOM, Work Center, QC Inspection |
| Projects | `project_management` | Project | WBS, Resource, Timesheet, Milestone |
| Assets | `asset_management` | Asset | Maintenance Schedule, Work Order |
| Support | `service_desk` | Ticket | SLA, Assignment, Resolution |

### Cross-Module Rules
- A Flow's frontend page MAY call APIs from multiple backend modules.
- Backend modules still MUST NOT import from each other (Rule 1 §6). Cross-module side effects go through the Event Bus.
- When a flow step triggers a side effect in another module (e.g., "Confirm Order" reserves inventory), it happens via event emission after the primary step's DB commit.

---

## 8. Flow Definition Files

### Location
Standard flow definitions live at:
```
features-plan/flows/<flow_id>/flow-definition.json
```

### Schema
```json
{
  "flowId": "sell",
  "displayName": "Sell",
  "description": "End-to-end sales workflow from quotation to payment collection",
  "icon": "💰",
  "tier": "basic",
  "primaryModule": "sales",
  "supportingModules": ["inventory", "finance"],
  "defaultPipeline": [
    {
      "id": "create_quote",
      "label": "Create Quote",
      "type": "action",
      "entity": "quotation",
      "statusEvent": "QUOTE_CREATED",
      "skippable": false,
      "dependsOn": []
    },
    {
      "id": "confirm_order",
      "label": "Confirm Order",
      "type": "action",
      "entity": "sales_order",
      "statusEvent": "ORDER_CONFIRMED",
      "skippable": false,
      "dependsOn": ["create_quote"]
    }
  ],
  "dataHubEntities": ["customers", "products"],
  "landingMetrics": [
    { "label": "Open Quotes", "query": "quotations.status=draft", "format": "count" },
    { "label": "Pending Orders", "query": "sales_orders.status=confirmed", "format": "count" },
    { "label": "This Month Revenue", "query": "invoices.month=current", "format": "currency" }
  ]
}
```

---

## 9. Pipeline Customization Data Model

### Database Table: `flow_pipeline_overrides`
```
flow_pipeline_overrides
├── id (UUID)
├── flow_id (string, e.g., "sell")
├── tenant_id / subsidiary_id
├── custom_steps (JSON — array of step overrides)
├── skipped_steps (JSON — array of step IDs to skip)
├── step_order (JSON — ordered array of step IDs)
├── created_at, updated_at, created_by
```

### Resolution Order
1. Load default pipeline from `flow-definition.json`
2. Apply tenant override: insert custom steps, remove skipped steps, reorder
3. Evaluate `decision` step conditions against the current entity data
4. Resolve `approval` step approvers against threshold rules
5. Validate: ensure `dependsOn` constraints are not violated after customization
6. Return the fully resolved pipeline to the frontend (including which step component to render)

---

## 10. Flow Event Lifecycle

Every flow step transition emits a standardized event:

```
Event Format: {FLOW}_{STEP}_{ACTION}

Examples:
  SELL_QUOTE_CREATED
  SELL_ORDER_CONFIRMED
  SELL_SHIPMENT_DISPATCHED
  SELL_INVOICE_GENERATED
  SELL_PAYMENT_COLLECTED
  BUY_PO_SENT
  BUY_GOODS_RECEIVED
```

These events:
- Trigger cross-module side effects (inventory reservation, ledger posting)
- Feed the notification system (email/in-app alerts)
- Update the flow's timeline for audit transparency
- Can be subscribed to by client-instance custom handlers

---

## 11. Documents & Attachments

Every flow step may produce or require documents. Attachments are a first-class concept.

### Attachment Data Model
```
entity_attachments
├── id (UUID)
├── entity_type (string — e.g., "sales_order", "purchase_order")
├── entity_id (UUID — the specific record)
├── step_id (string, nullable — which flow step uploaded this)
├── file_name (string)
├── file_type (string — MIME type)
├── file_size (integer — bytes)
├── storage_path (string — relative path in tenant's storage)
├── uploaded_by (UUID)
├── category (string — "document" | "photo" | "contract" | "receipt" | "other")
├── created_at
```

### Rules
- Every entity detail view (DetailPanel) MUST have a "Documents" tab showing all attachments.
- Flow step views SHOULD have an attachment dropzone when the step produces/requires documents.
- Supported formats: PDF, images (PNG/JPG/WEBP), spreadsheets (XLSX/CSV). Max size: 10MB per file.
- Storage is tenant-isolated. Files are stored under `storage/<tenant_id>/<entity_type>/<entity_id>/`.
- Attachments are soft-deleted with the parent entity — never orphaned.
- Common attachment categories per flow:

| Flow | Step | Typical Attachments |
|:-----|:-----|:-------------------|
| Sell | Quote | Signed quotation PDF |
| Sell | Ship | Delivery note, packing slip, proof of delivery photo |
| Sell | Invoice | Generated invoice PDF |
| Buy | Send PO | Purchase order PDF sent to supplier |
| Buy | Receive Goods | GRN document, inspection photos |
| Buy | Match Invoice | Supplier invoice scan |

---

## 12. Comments & Internal Notes

Multi-role flows need a conversation thread on entities. A warehouse person needs to ask "which dock?" An accountant needs to flag "payment terms dispute — hold invoice."

### Comment Data Model
```
entity_comments
├── id (UUID)
├── entity_type (string)
├── entity_id (UUID)
├── step_id (string, nullable — context of which flow step)
├── author_id (UUID)
├── content (text)
├── is_internal (boolean, default true — internal notes vs customer-visible)
├── mentions (JSON array of user IDs — for @mentions)
├── created_at, updated_at
├── is_deleted (boolean)
```

### Rules
- Every entity detail view MUST have a "Notes" section (either in Overview tab or a dedicated tab).
- Comments support @mentions — mentioning a user sends them a notification.
- Flow step views show a collapsible comment thread relevant to that step.
- Comments are append-only for audit safety. Users can edit their own comments (with edit history) but not delete others'.
- During handoffs, the completing user SHOULD be prompted to leave a note for the next owner: "Any notes for the warehouse team?"
- `is_internal = true` for team-only notes. `is_internal = false` for notes that may be included in customer-facing documents.

---

## 13. Draft Persistence & Auto-Save

Users get interrupted. A 50-line-item order shouldn't be lost because someone closed their browser.

### Rules
- Every `action` step MUST support saving the current form state as a draft.
- Drafts are auto-saved to the backend every 30 seconds (debounced on input) or on tab blur.
- Draft entities have `status = "draft"` and are excluded from metrics/reports until confirmed.
- The flow landing page shows drafts in the "Draft" quick filter tab.
- Drafts belong to the user who created them. Other users with the same role can see but not edit drafts they didn't create (prevents conflicts).
- Drafts older than 30 days surface a "Stale draft" warning. Drafts older than 90 days can be auto-archived.
- When a user returns to a draft, the flow step view resumes exactly where they left off (pre-populated form, scroll position, active section).

### Backend
- Draft auto-save hits `PUT /api/v1/<module>/<entity>/{id}` with `status: "draft"`.
- Auto-save responses are lightweight — return only `{ id, updated_at, version_id }` to minimize payload.
- Version conflicts on drafts are resolved by last-write-wins (drafts are single-user).

---

## 14. Partial Operations

Real business is not all-or-nothing. Partial shipments, partial payments, and partial deliveries are normal.

### Partial Fulfillment Model
```
Entity: sales_order_lines (or equivalent per-flow entity)
├── ordered_qty (Decimal)
├── fulfilled_qty (Decimal, default 0)     ← tracks cumulative fulfillment
├── remaining_qty (computed: ordered - fulfilled)
├── fulfillment_status ("unfulfilled" | "partial" | "fulfilled")
```

### Rules
- Line items track `ordered_qty`, `fulfilled_qty`, and compute `remaining_qty`.
- A step can be completed **partially** — e.g., ship 80 of 100 items. The flow does NOT advance fully; the step stays "in progress" with a partial completion marker.
- When all line items reach `fulfilled_qty == ordered_qty`, the step auto-completes.
- Partial operations create linked child documents:
  - Order (100 items) → Shipment 1 (80 items) → Shipment 2 (20 items)
  - Invoice ($10,000) → Payment 1 ($5,000) → Payment 2 ($5,000)
- The parent entity's status reflects the aggregate: "Partially Shipped", "Partially Paid".
- The FlowStepper shows partial steps with a half-filled visual indicator (not just pending/complete).

### Partial Payment Tracking
```
Entity: payment_allocations
├── payment_id (UUID)
├── invoice_id (UUID)
├── amount (Decimal 20,4)
├── allocated_at (datetime)
```

An invoice's `balance_due = total_amount - SUM(payment_allocations.amount)`. When `balance_due == 0`, status → "Paid".

---

## 15. Reversals & Corrective Flows

Flows go forward. But real business also goes backward — returns, cancellations, credit notes, payment reversals.

### Reversal Model

Every forward flow step has a potential reverse action:

| Forward Step | Reverse Action | Creates |
|:-------------|:--------------|:--------|
| Confirm Order | Cancel Order | Cancellation record + stock un-reservation |
| Ship | Return / RMA | Return Merchandise Authorization |
| Invoice | Credit Note | Credit note linked to original invoice |
| Collect Payment | Refund | Refund record linked to original payment |
| Receive Goods (Buy) | Return to Supplier | Supplier return + stock adjustment |

### Rules
- Reversals create NEW entities — they do NOT delete or modify the original. The original document remains immutable for audit.
- Every reversal entity has a `reversal_of` field (FK to the original entity) for traceability.
- Reversals follow the same permission model — a warehouse person can initiate an RMA, but a manager might need to approve a refund.
- Reversal events follow the naming convention: `{FLOW}_{ENTITY}_REVERSED` (e.g., `SELL_INVOICE_REVERSED`, `SELL_PAYMENT_REFUNDED`).
- The original entity's status updates to reflect the reversal: "Invoiced" → "Credit Noted", "Paid" → "Refunded".
- Partial reversals are supported: return 3 of 10 items, credit $500 of $1,000.

### State Machine Extension
```
Standard:     Draft → Confirmed → Shipped → Invoiced → Paid → Closed
Reversals:    Shipped → Partially Returned → Fully Returned
              Invoiced → Credit Noted
              Paid → Partially Refunded → Fully Refunded
Cancellation: Draft → Cancelled
              Confirmed → Cancelled (with reversal of reservations)
```

---

## 16. Cross-Flow Linking

Flows don't exist in isolation. A sale triggers a purchase. A return triggers a stock adjustment. These connections must be visible and navigable.

### Link Data Model
```
flow_entity_links
├── id (UUID)
├── source_flow (string — e.g., "sell")
├── source_entity_type (string — e.g., "sales_order")
├── source_entity_id (UUID)
├── target_flow (string — e.g., "buy")
├── target_entity_type (string — e.g., "purchase_order")
├── target_entity_id (UUID)
├── link_type ("triggered" | "converted" | "referenced" | "reversed")
├── created_at
```

### Link Types
- `triggered` — This entity caused that entity to be created (e.g., low stock from Sell → auto-created Purchase Order in Buy)
- `converted` — This entity was transformed into that entity (e.g., Quotation → Sales Order)
- `referenced` — These entities are related but neither caused the other (e.g., Invoice references Shipment)
- `reversed` — This entity is a reversal of that entity (e.g., Credit Note reverses Invoice)

### Frontend Display
- DetailPanel shows a "Related" tab listing all linked entities across flows.
- Each link is clickable — navigates to the target entity's DetailPanel or flow step view.
- The flow Timeline tab shows cross-flow events: "Purchase Order PO-0023 was triggered by this order."

### Rules
- Links are created automatically by event subscribers when cross-module side effects occur.
- Links are bidirectional in display (shown on both source and target) but stored once.
- Manual linking is allowed — a user can link any two entities for reference.

---

## 17. Recurring & Repeat Flows

Businesses have recurring transactions — monthly orders from the same customer, weekly stock counts, quarterly payroll.

### Rules
- **Duplicate**: Any completed flow instance can be duplicated as a new draft. "Repeat this order" copies customer, line items, and terms into a new Sales Order with `status: draft`.
- **Templates**: Users can save a flow instance as a named template (e.g., "Monthly Apex Technologies Order"). Templates are stored per-tenant and appear in the "Start New [Flow]" dialog.
- **Scheduled Recurrence** (Pro tier): A flow instance can be set to auto-repeat on a schedule (daily/weekly/monthly/custom cron). The system creates a draft at the scheduled time and assigns it to the initiator's My Tasks.
- **Template Data Model**:
```
flow_templates
├── id (UUID)
├── flow_id (string)
├── name (string — "Monthly Apex Order")
├── template_data (JSON — snapshot of entity fields, line items, etc.)
├── recurrence (JSON, nullable — { frequency, interval, next_run, end_date })
├── created_by (UUID)
├── created_at, updated_at
```

---

## 18. Number Sequences

Every business document needs a unique, human-readable reference number.

### Rules
- Every entity that represents a business document MUST have a `ref_number` field (string, unique, indexed).
- Reference numbers follow a configurable pattern: `{PREFIX}-{YYYY}{SEQ}` (e.g., `SO-202600001`, `INV-202600042`).
- Sequences are per-entity-type, per-tenant, and reset annually (configurable).
- The sequence service is in core: `core.sequences.next_number(entity_type: str) → str`.
- Sequences are gap-free in normal operation. Cancelled documents retain their number (no reuse).
- The prefix and pattern are configurable per entity type in Settings.

### Sequence Data Model
```
number_sequences
├── id (UUID)
├── entity_type (string — e.g., "sales_order", "invoice")
├── prefix (string — e.g., "SO", "INV")
├── pattern (string — e.g., "{PREFIX}-{YYYY}{SEQ:05d}")
├── current_value (integer)
├── reset_frequency ("never" | "yearly" | "monthly")
├── last_reset_at (datetime)
├── subsidiary_id (UUID)
```

---

## 19. Bulk Operations

Enterprise users process volume. One-at-a-time is painful when you have 50 invoices to send or 20 POs to approve.

### Rules
- DataTable MUST support row selection (checkbox column). Select all / select page / select individual.
- When rows are selected, a **Bulk Action Bar** appears above the table with available actions.
- Available bulk actions depend on the entity type and selected entities' statuses:

| Entity | Bulk Actions |
|:-------|:-------------|
| Invoices | Send All, Mark as Paid, Export PDF |
| Purchase Orders | Approve Selected, Reject Selected, Send to Suppliers |
| Tasks | Reassign, Change Priority, Mark Complete |
| Products | Update Prices, Change Category, Archive |

### Backend Pattern
- Bulk operations hit `POST /api/v1/<module>/<entity>/bulk` with `{ action: "approve", ids: [uuid, uuid, ...] }`.
- Max 100 items per bulk request. Larger batches are split client-side.
- Bulk operations return per-item results: `{ succeeded: [...], failed: [{ id, error }] }`.
- Failed items don't block successful ones — partial success is expected.
- Bulk operations emit one aggregate event (e.g., `SELL_INVOICES_BULK_SENT`) not per-item events, to prevent notification storms.

### Frontend Pattern
- Bulk Action Bar shows: selected count, available actions, "Clear selection" link.
- Long-running bulk operations show a progress indicator.
- On completion, a Toast shows summary: "15 of 18 invoices sent. 3 failed — [View Details]."

---

## 20. Tenant Extensibility Framework

The architecture MUST support tenant customization without modifying core code. These are the extension points that every flow and entity exposes:

### Custom Fields

Tenants can define additional fields on any entity without schema migrations:

```
custom_field_definitions
├── id (UUID)
├── entity_type (string — e.g., "sales_order")
├── field_key (string — e.g., "priority_level")
├── field_label (string — "Priority Level")
├── field_type ("text" | "number" | "date" | "select" | "boolean" | "currency")
├── options (JSON, nullable — for select: ["Low", "Medium", "High", "Critical"])
├── is_required (boolean)
├── default_value (string, nullable)
├── display_order (integer)
├── show_in_table (boolean — show as a column in DataTable)
├── show_in_detail (boolean — show in DetailPanel)
├── subsidiary_id (UUID)
├── created_at, updated_at
```

**Storage**: Custom field values are stored in the entity's `metadata_` JSON column (already on BESBase). The key in `metadata_` matches `field_key`.

**Frontend rendering**: The `FormSection` and `DetailPanel` components check for `custom_field_definitions` for the current entity type and render additional fields dynamically below the standard fields. Custom fields are visually distinguished with a subtle indicator.

**Backend validation**: Custom field values submitted via API are validated against their `field_type` and `is_required` constraints. Invalid custom fields return the same Pydantic-style validation errors as standard fields.

**Search & Filter**: Custom fields marked `show_in_table = true` appear as filterable columns in DataTable. The backend supports filtering on `metadata_` JSON keys.

### Webhook Automation Steps

Tenants can add `automation` steps that call external HTTP endpoints:

```json
{
  "id": "notify_warehouse_api",
  "type": "automation",
  "label": "Notify Warehouse System",
  "automationConfig": {
    "method": "POST",
    "url": "https://warehouse.acme.com/api/orders",
    "headers": { "Authorization": "Bearer {{secrets.WAREHOUSE_API_KEY}}" },
    "body": {
      "order_id": "{{entity.ref_number}}",
      "items": "{{entity.line_items}}",
      "ship_to": "{{entity.shipping_address}}"
    },
    "retryPolicy": { "maxRetries": 3, "backoffMs": 1000 },
    "onFailure": "continue"
  }
}
```

**Rules**:
- Webhook URLs are stored encrypted. Secrets use `{{secrets.KEY}}` template syntax, resolved from the tenant's secret store.
- `body` supports `{{entity.field}}` and `{{user.field}}` template variables resolved at runtime.
- `onFailure`: `"continue"` (log error, proceed to next step), `"pause"` (halt flow, create a task for admin), `"retry"` (use retry policy).
- Webhook execution is async — the flow step shows a loading state while the HTTP call is in progress.
- All webhook calls are logged in the entity's Timeline for audit.
- Webhook steps are a **Pro tier** feature.

### Build-Time Implications

These extensibility points affect how we build from day one:

| What we build now | What it enables later |
|:-----------------|:---------------------|
| `metadata_` JSON column on BESBase | Custom fields stored without migrations |
| `FormSection` accepts a `customFields` prop | Dynamic form rendering for tenant-defined fields |
| `DataTable` column config accepts dynamic columns | Custom field columns in entity lists |
| `DetailPanel` renders custom fields section | Custom fields visible in entity detail |
| Step rendering uses ComponentRegistry lookup chain | Client instances can override any step's UI |
| `GenericStepRenderer` reads step schema | Tenant-added pipeline steps render without code |
| `automationConfig` in step schema | Webhook/HTTP callout steps work via config |
| `approvalRules` with conditions | Threshold-based approval chains via Pipeline Editor |
| `decision` step with condition evaluator | Conditional branching without custom code |
| `flow_pipeline_overrides` in DB | All pipeline customization persisted per-tenant |

> **Build principle**: Build the standard flow with hardcoded components. But every component MUST accept extensibility props (`customFields`, `extraColumns`, `additionalActions`) even if they're empty arrays today. This prevents rewrites when customization is activated.

