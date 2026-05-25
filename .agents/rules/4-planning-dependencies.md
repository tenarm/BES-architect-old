# 4. Planning & Dependency Management Rules — BES

These rules govern how feature plans, cross-module dependencies, and shareable references are reviewed, documented, and stored, especially for massive applications built in stages.

---

## 1. Scope and Purpose of proposed-plan.md & common-dependants.md
- **`proposed-plan.md`**: Generated for each feature under `features-plan/<module>/<feature>/`. It acts as the final architectural road-map before build step implementation.
- **`features-plan/common-dependants.md`**: The global, shared reference registry for all cross-module interactions, configuration mappings, and reusable assets. It serves as the single source of truth to ensure the application build progress remains on track.

---

## 2. Staged Building of Massive Applications
To prevent development bottlenecks and maintain pipeline integrity in a multi-stage enterprise build, features must align with the system's topological order:

1. **Stage 1 (Core Foundations & Global Settings)**: Auth, Subsidiaries, Users, Roles, currencies, base UOM, and master configuration entities.
2. **Stage 2 (Master Data Management - MDM)**: Customer Master, Supplier Master, Item Master.
3. **Stage 3 (Domain Transaction Modules)**: Inventory counts, Sales Orders, Purchase Orders, General Ledger.
4. **Stage 4 (Advanced Pipelines & Cross-Module Features)**: Process approval pipelines, automated ledger reconciliations, cross-extension triggers.

---

## 3. Inbound vs. Outbound Dependencies
Every feature review must identify and document the following dependency categories:

*   **Required External Dependencies (Inbound / Prerequisites)**:
    *   **Domain & Schema-Level References**: Specific database entities, foreign keys, or configuration parameters defined in previous modules (e.g., a Sales Order requiring `subsidiary_id` from Settings and `item_id` from Inventory).
    *   **External APIs & Services**: Backend service methods, endpoints, or data models consumed from other extensions (e.g., Sales Credit checks querying Finance aging reports).
    *   **Pre-requisite Build Steps**: Explicitly list which files/endpoints MUST be implemented first before the current feature can compile and run.
*   **Exposed Reusable Assets & Shared Reference Notes (Outbound / Shareable)**:
    *   **Shareable Entities & Fields**: Database columns, lookup tables, and enum structures that other modules will import or reference.
    *   **APIs & Hooks**: Shared backend routes (e.g., active catalog lookups) or frontend hooks/stores.
    *   **Shared UI Elements & Utilities**: Generic widgets and helper functions built in this feature that can be extracted to `@bes/shared-ui` or referenced in upcoming stages.

---

## 4. Loose Coupling and "Ghost Foreign Keys"
To prevent circular database dependencies and migration deadlocks across extension modules:
- Standard extensions must not use hard PostgreSQL-level foreign keys referencing other standard extensions.
- Use **Ghost Foreign Keys**: Store references to other extension entities (e.g., a Purchase Order referencing an Inventory Item) inside a JSONB column (`metadata_` on `BESBase`) or validate dynamically via services at runtime.
- **Stubs & Mocks**: If a required feature from another module is not yet implemented (building in stages), the proposed plan must specify the API stub or repository mock required to unblock development without going off track.

---

## 5. Formatting of common-dependants.md
To keep the global store clean, organized, and informative, every module/feature entry in `common-dependants.md` MUST follow this exact structure:
- Organize under `## <Module Name> Module` -> `### Feature: <Feature Name>`.
- Provide a clean, tabular presentation for both inbound and outbound dependencies.

### A. Required External Dependencies (Inbound / Prerequisites)
| Target Module | Dependent Entity / Feature | Dependency Description | Impact / Mitigation & Pre-requisite Build Order |
| :--- | :--- | :--- | :--- |

### B. Exposed Reusable Assets & Shared Reference Notes (Outbound / Shareable)
| Shareable Entity / Utility | Target Consumers | Asset Description | Integration & Reference Notes for Upcoming Stages |
| :--- | :--- | :--- | :--- |

---

## 6. Architectural Rules for Dependency Verification
1. **Graphify Verification**: The plan reviewer MUST run `graphify path` or `graphify query` to verify that proposed external dependencies or shared assets actually exist (or are planned) before referencing them.
2. **Circular Prevention**: Standard extension modules MUST NOT directly import or call each other. All inter-module actions must be event-driven via the Event Bus, or queried dynamically at runtime.
3. **Lookup Registry Maintenance**: Before designing or proposing a new helper service, table, or UI component, the plan reviewer must consult `common-dependants.md` to see if a similar reusable asset has already been exposed. If it exists, the proposed plan must reuse it.
