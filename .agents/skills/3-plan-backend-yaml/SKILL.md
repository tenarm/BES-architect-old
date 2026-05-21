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
2. Generate the 6-section document below.
3. Save to `features-plan/<module-name>/<feature-name>/3-backend-plan.md`.

---

## Required 6-Section Structure

## 1. Module & Feature Name
State the parent module and feature.

## 2. Database Schema (YAML)
Provide a detailed YAML representation of the database schema including Tables, Columns, Types, and Foreign Keys (inheriting `BESBase`).

## 3. Hub-and-Spoke MDM Mapping
Explicitly state how this feature maps to or extends the `core` schema entities.

## 4. REST APIs
List all required APIs with request/response payloads conforming to the standard API envelope.

## 5. Pub/Sub Events & Process Definitions
- **Pub/Sub Events**: Define Event triggers (`UPPER_SNAKE_CASE` Pub/Sub events) to be emitted or listened to.
- **Workflow Pipeline Definition JSON**: Specify the step-by-step process definition structure matching the schema fields (`processId`, `module`, `label`, `entity`, `steps`, `statusEvent`, `dependsOn`, `requiredRole`, `requiredModule`, `requiredFeature`, `action`).
  - **Dynamic Step Licensing**: For any steps that cross into separate modules or require specific packages, explicitly specify `requiredModule` and `requiredFeature` on the step level to enable self-healing, license-aware pipeline construction.
  - State that this file must be saved in `extensions/<module_name>/<module_name>/process_definitions/<module_name>.json` so it can be dynamically loaded, filtered, and validated.

## 6. RBAC Permissions & Licensing Guards
- **RBAC Matrix**: Define required roles and `<module>:<resource>:<action>` mappings.
- **Licensing Configurations**: Define changes required under `core/core/packages.json` to assign this sub-feature to a specific tier (Basic, Pro, or Premium).
- **API Guard Placement**: Specify precisely which endpoints or services will invoke `require_licensed_feature("<module>", "<subfeature>")`.

---

### Execution Rules
- **Output file**: `features-plan/<module-name>/<feature-name>/3-backend-plan.md`
- **Next step**: Run `4-plan-reviewer`.

