---
trigger: always_on
description: Flow-based planning pipeline, dependency management, build staging, and cross-flow reference registry governing how TenArm features are planned and documented.
---

# 4. Planning & Dependency Management Rules — TenArm

These rules govern how flows are planned, documented, and tracked before implementation.

---

## 1. Flow-First Planning Principle
All feature planning is organized around **Flows**, not modules. A flow spans multiple backend modules and produces a single, cohesive user experience.

- **Planning unit**: A Flow (e.g., "Sell", "Buy"), not a module (e.g., "Sales", "Inventory").
- **Planning artifacts**: Live under `features-plan/flows/<flow_id>/`.
- **Module-level concerns** (data models, API schemas) are addressed WITHIN the flow plan — they are not separate planning exercises.

---

## 2. Planning Directory Structure

```
features-plan/
├── flows/
│   ├── sell/
│   │   ├── 1-flow-definition.md      ← Steps, entities, business rules
│   │   ├── 2-flow-ui-design.md       ← Screens, layouts, component mapping
│   │   ├── 3-flow-backend-plan.md    ← Models, APIs, services, events
│   │   ├── proposed-plan.md          ← Reviewed & approved implementation plan
│   │   └── flow-definition.json      ← Machine-readable flow pipeline definition
│   ├── buy/
│   ├── stock/
│   └── ...
├── data-hub/
│   └── <entity>/                     ← Entity-specific Data Hub plans if needed
├── common-dependants.md              ← Cross-flow dependency registry
└── deferred-architecture.md          ← Future features parked for later
```

---

## 3. Sequential Planning Pipeline (Skills)

Every flow MUST go through this 5-step pipeline before implementation:

| Step | Skill | Input | Output |
|:-----|:------|:------|:-------|
| 1 | `1-plan-flow` | Flow name + description | `1-flow-definition.md` — steps, entities, business rules, validations |
| 2 | `2-design-flow-ui` | Flow definition | `2-flow-ui-design.md` — screens, layouts, component specs |
| 3 | `3-plan-flow-backend` | Flow definition + UI design | `3-flow-backend-plan.md` — models, schemas, APIs, services, events |
| 4 | `4-review-flow` | All 3 docs + rules | `proposed-plan.md` — reviewed implementation plan ready for approval |
| 5 | `5-build-flow` | Approved proposed-plan | Working code — backend + frontend built together |

> Each skill can be run independently. Run them in order for a complete flow from planning to code.

---

## 4. Flow Definition Document (`1-flow-definition.md`)

Every flow definition MUST contain:

1. **Flow Identity**: ID, display name, description, icon, tier, primary module, supporting modules.
2. **Default Pipeline**: Ordered list of steps with type, entity, status event, dependencies, and skippability.
3. **Entities Involved**: List of database entities this flow creates/modifies, with their key fields.
4. **Business Rules**: Validation rules, constraints, calculations, and invariants specific to this flow.
5. **State Machine**: The valid status transitions for the flow's primary entity (e.g., Draft → Confirmed → Shipped → Invoiced).
6. **Cross-Module Events**: Events emitted at each step and the expected subscribers.
7. **Data Hub Entities**: Which Data Hub views this flow feeds data into.

---

## 5. Staged Build Order

Flows are built in dependency order. A flow that needs data from another flow must wait for that flow to be built first.

### Phase 1: Foundation
- Settings (Company, Users, RBAC)
- Design System & Core Components

### Phase 2: Core Transactions
- **Sell Flow** — requires: Settings, Customers (Data Hub), Products (Data Hub)
- **Buy Flow** — requires: Settings, Suppliers (Data Hub), Products (Data Hub)
- **Stock Flow** — requires: Products, Warehouses

### Phase 3: Financial
- **Money Flow** — requires: Sell Flow (invoices), Buy Flow (bills)

### Phase 4: People & CRM
- **People Flow** — independent (HR module)
- **Customers Flow** — independent (CRM module)

### Phase 5: Advanced
- **Manufacture Flow** — requires: Products, Stock, Buy
- **Projects Flow** — requires: People
- **Assets Flow** — independent
- **Support Flow** — independent

---

## 6. Cross-Flow Dependencies (`common-dependants.md`)

This file is the global registry of what each flow **needs** (inbound) and **produces** (outbound). It is **auto-maintained** by the skills pipeline — do NOT edit manually.

### Lifecycle
- **Created/Drafted** by Skill 1 (`1-plan-flow`), Step 9 — when a flow is first planned, its expected dependencies and outbound assets are registered.
- **Finalized** by Skill 5 (`5-build-flow`), Step 14 — after the code is built, entries are updated with verified event names, table names, and API endpoints from the actual code.

### Structure
Organize under `## Flow: <Flow Name>` with two tables:

#### A. Required External Dependencies (Inbound)
| Source Flow/Module | Dependency | Description | Phase |
| :--- | :--- | :--- | :--- |

#### B. Exposed Assets (Outbound)
| Asset | Consumers | Description | Notes |
| :--- | :--- | :--- | :--- |

---

## 7. Domain-Only Planning Content
Flow plans MUST focus on domain-specific business logic, not framework mechanics:

**Include**: Entity fields, business validations, status transitions, calculation formulas, approval conditions, user-facing labels and descriptions.

**Exclude**: CSS z-index values, JSON envelope format, soft-delete query patterns, BESBase fields, import ordering — these are governed by the Rules files.

---

## 8. Deferred Architecture (`deferred-architecture.md`)
Features that are planned but not in the current build phase are tracked here. Each entry has:
- Feature description
- Which flow it belongs to
- Prerequisites / dependencies
- Target tier (Basic/Pro/Premium)
- Reason for deferral

This document prevents scope creep while ensuring nothing is forgotten.

---

## 9. Value-Driven Flow Planning
- **Core Transactional Value (CTV)**: Define the minimum viable steps for a flow to be useful. This ships in the Basic tier.
- **Advanced Governance (AEG)**: Approval gates, automation hooks, advanced validations — these are Pro/Premium additions to the same flow, not separate features.
- **Progressive Complexity**: A flow starts simple (3-5 steps) and grows via the Pipeline Editor. Never ship a 10-step flow when a 5-step flow covers 90% of users.

---

## 10. SME-less Validation Rules
When building without domain experts:
- Benchmark flows against industry standards (GAAP/IFRS for finance, APICS for supply chain).
- Reference open-source ERP patterns (Odoo, ERPNext) for step sequences and entity schemas.
- Default to manual user input over automated heuristics when uncertain.
- Document the reference source for every complex business rule or status machine.
