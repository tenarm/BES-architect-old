---
name: 3-plan-backend-yaml
description: "PLANNING STEP 3: Create detailed YAML, required APIs, and backend implementation plan."
---

# Skill: Plan Backend YAML
**Lifecycle Position: STEP 3 of 6 — Backend Planning**
**Feeds into:** `4-plan-reviewer`

This skill translates the functionalities and UI flow into a concrete backend schema and API contract.

---

## Lifecycle Context
```
STEP 1: 1-plan-functionalities  →  1-functionalities.md
STEP 2: 2-plan-ui-ux            →  2-ui-ux-flow.md
[YOU ARE HERE]
STEP 3: 3-plan-backend-yaml     →  3-backend-plan.md
STEP 4: 4-plan-reviewer         →  proposed-plan.md & common-dependants.md
STEP 5: 5a/5b build             →  CODE
STEP 6: 6-bug-fixing            →  BUG FIXES
```

---

## About the BES Architecture
- **Foundation**: All tables inherit `BESBase` → `id` (UUID PK), `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `metadata_` (JSONB).
- **Hub-and-Spoke MDM**: Master data in `core` schema. Module tables FK to core.
- **Precision**: `Numeric(20,4)` for all financial columns.
- **API Envelope**: Standard `{ status, data, metadata, error }`.
- **RBAC Format**: `<module>:<resource>:<action>`.

---

## Instructions for the Assistant

When the user asks to plan backend:

1. Read `1-functionalities.md` and `2-ui-ux-flow.md`.
2. Generate the 7-section document below.
3. Save to `features-plan/<module-name>/<feature-name>/3-backend-plan.md`.

---

## Required 7-Section Structure

## 1. Module & Feature Name
State the parent module and feature.

## 2. Database Schema (YAML)
Provide a detailed YAML representation of the database schema including Tables, Columns, Types, and Foreign Keys. 
- *Rule Check:* Do not repeat standard audit and tracking fields (`id`, `created_at`, `updated_at`, `created_by`, `is_deleted`, `subsidiary_id`, `version_id`) as they are automatically provided by `BESBase`. Focus only on feature-specific business fields.

## 3. Hub-and-Spoke MDM Mapping
Explicitly state how this feature maps to or extends the `core` schema entities.

## 4. REST APIs
List all required API endpoints with request/response payloads.
- *Rule Check:* Do not define standard HTTP status codes, error models, or response envelopes. Identify which endpoints require transactional locking for high-contention operations.

## 5. Pub/Sub Events & Process Definitions
- **Pub/Sub Events**: Define event triggers (`UPPER_SNAKE_CASE` Pub/Sub events) to be emitted or listened to.
- **Workflow Pipeline Definition**: If the feature uses a process pipeline, map out the step dependencies, roles, and status event hooks. The detailed JSON format and validation schema are governed by [1-backend-architecture.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/1-backend-architecture.md).
- **Notification Rule Seeds**: Map out notifications to seed (associated event, target channel, recipient role, and template messages). The exact JSONPath payload resolution, Jinja2 template formatting rules, and channel licensing checks are governed by [1-backend-architecture.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/1-backend-architecture.md) and must not be repeated.

## 6. RBAC Permissions & Licensing Tiers
- **RBAC Matrix**: Define required roles and `<module>:<resource>:<action>` mappings.
- **Licensing Configurations**: Define feature packaging tier assignment (Basic, Pro, or Premium). All premium endpoints and features are automatically gated by standard licensing checks according to the rules.

## 7. Service Layer & Core Business Logic
Define the business services structure, validation policies, and core transaction boundaries in the service layer (`services.py`).
- **Domain Validation & Business Invariants Matrix**: List the precise business validation policies, preconditions, and exception triggers (e.g. unique constraints, state-transition rules, value limits) for each operational service.
- **Service Method Specifications**: For each major service operation:
  - **Method Signature**: Declare method name, parameters, and return type.
  - **Pre-conditions & Validations**: Specify checks performed before mutating the database state.
  - **Calculations & Rounding Algorithms**: Define business calculations, formulas, discount allocations, and rounding criteria (decimal-safe).
  - **Database Actions & Side-Effects**: Detail models read/updated, events emitted, notifications triggered, or updates to associated models.
  - **CTV Transactional Isolation**: Design service methods so that core transactions (e.g. creating/updating basic entities) are isolated from optional, premium side-effects (e.g. complex scorecard computations, external lookups). If advanced logic fails, ensure the core database mutation still succeeds gracefully.

---

### Execution Rules
- **Output file**: `features-plan/<module-name>/<feature-name>/3-backend-plan.md`
- **Next step**: Run `4-plan-reviewer`.
