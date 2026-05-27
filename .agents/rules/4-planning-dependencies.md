---
trigger: always_on
description: Scope definition, topological build stages, inbound/outbound dependencies, master data governance, and ledger rules governing planning and plan reviews.
---

# 4. Planning & Dependency Management Rules — BES

These rules govern how feature plans, cross-module dependencies, and shareable references are reviewed, documented, and stored, especially for massive applications built in stages.

---

## 1. Domain-Only Feature Planning Principle
To maintain clean, focused, and high-value documentation, feature planning documents (`1-functionalities.md`, `2-ui-ux-flow.md`, `3-backend-plan.md`, `proposed-plan.md`) must **strictly focus on domain-specific business logic, processes, columns, and invariants**. 
Feature plans **MUST NOT duplicate or repeat generic mechanical/framework rules** (such as technical money-rule database/code declarations, soft delete queries, standard API response envelopes, sessionStorage caching mechanisms, and CSS z-indexes). These framework rules are defined globally in these central Rules files, which serve as the absolute, non-negotiable source of truth for the entire coding chassis.

---

## 2. Scope and Purpose of proposed-plan.md & common-dependants.md
- **`proposed-plan.md`**: Generated for each feature under `features-plan/<module>/<feature>/`. It acts as the final architectural road-map before build step implementation.
- **`features-plan/common-dependants.md`**: The global, shared reference registry for all cross-module interactions, configuration mappings, and reusable assets. It serves as the single source of truth to ensure the application build progress remains on track.

---

## 3. Staged Building of Massive Applications
To prevent development bottlenecks and maintain pipeline integrity in a multi-stage enterprise build, features must align with the system's topological order:

1. **Stage 1 (Core Foundations & Global Settings)**: Auth, Subsidiaries, Users, Roles, currencies, base UOM, and master configuration entities.
2. **Stage 2 (Master Data Management - MDM)**: Customer Master, Supplier Master, Item Master.
3. **Stage 3 (Domain Transaction Modules)**: Inventory counts, Sales Orders, Purchase Orders, General Ledger.
4. **Stage 4 (Advanced Pipelines & Cross-Module Features)**: Process approval pipelines, automated ledger reconciliations, cross-extension triggers.

---

## 4. Inbound vs. Outbound Dependencies
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

## 5. Loose Coupling and "Ghost Foreign Keys"
To prevent circular database dependencies and migration deadlocks across extension modules:
- Standard extensions must not use hard PostgreSQL-level foreign keys referencing other standard extensions.
- Use **Ghost Foreign Keys**: Store references to other extension entities (e.g., a Purchase Order referencing an Inventory Item) inside a JSONB column (`metadata_` on `BESBase`) or validate dynamically via services at runtime.
- **Stubs & Mocks**: If a required feature from another module is not yet implemented (building in stages), the proposed plan must specify the API stub or repository mock required to unblock development without going off track.

---

## 6. Master Data Governance (MDM) Rules
To maintain a single source of truth across staging phases:
- **System of Record (SoR)**: Every master entity (e.g., Tax Code, Employee Record, Item Master) must have exactly one owner module. Other modules must access this data read-only.
- **Change Control Gates**: Critical master fields (e.g., Customer Credit Limit, Vendor Bank Details) must not allow direct database edits. All modifications must route through designated approval workflows or trigger notification alerts.

---

## 7. Financial, Compliance & Ledger Posting Rules
- **Immutable Ledgers**: Transactional records affecting stock, financials, or assets must post to write-once, read-many sub-ledgers. Corrections must use offset reversal transactions; direct UPDATE/DELETE operations on posted ledgers are forbidden.
- **UOM & Currency Scaling**: Transactional conversion rules and exchange rate variance mappings must be explicitly planned at the schema and service level.
- **Period Close Gates**: Financial and inventory transaction services must validate accounting period lock status before writing postings.

---

## 8. Formatting of common-dependants.md
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

## 9. Architectural Rules for Dependency Verification
1. **Graphify Verification**: The plan reviewer MUST run `graphify path` or `graphify query` to verify that proposed external dependencies or shared assets actually exist (or are planned) before referencing them.
2. **Circular Prevention**: Standard extension modules MUST NOT directly import or call each other. All inter-module actions must be event-driven via the Event Bus, or queried dynamically at runtime.
3. **Lookup Registry Maintenance**: Before designing or proposing a new helper service, table, or UI component, the plan reviewer must consult `common-dependants.md` to see if a similar reusable asset has already been exposed. If it exists, the proposed plan must reuse it.

---

## 10. Value-Driven Planning Principle (Core Transactional Value vs. Advanced Governance)
To prevent feature bloat and ensure high usability, planners must strictly prioritize core value-creating actions:
- **Core Transactional Value (CTV)**: Define the absolute minimum data fields, database schemas, and simple CRUD paths needed for basic operations (e.g., creating a supplier profile to write a purchase order). This belongs in the Basic/Standard tier.
- **Advanced Efficiency & Governance (AEG)**: Move heavy automation, multi-role verification pipelines, complex scorecards, and third-party integrations to Pro/Premium tiers.
- **Graceful Isolation**: Core transactional paths must execute successfully even if optional AEG side-effects (like scoring calculations or external checks) fail.

## 11. SME-less Validation & Defensive Planning Rules
To avoid building non-standard, impractical, or incorrect workflows when human Subject Matter Experts (SMEs) are unavailable:
- **Benchmark Against Industry Standards**: Base all database schemas, financial calculations, and state machines on open standards (e.g., GAAP/IFRS for Finance, APICS/ASCM for Supply Chain) and reference open ERP platforms (e.g., Odoo, ERPNext).
- **The Defensive Planning Rule**: When in doubt or lacking SME validation, default to manual user inputs (e.g., a simple text field or dropdown) rather than complex, automated heuristics. Simple manual inputs are safe and standard, while automated algorithms designed without SME feedback risk breaking real-world operations.
- **Mandatory Reference Citations**: Planners must document the standard reference or ERP benchmark pattern that justifies any complex state machine, status hold, or core calculation.

