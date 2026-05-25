---
name: 1-plan-functionalities
description: "PLANNING STEP 1: Evaluate, validate, and plan the functionalities list of a feature using senior ERP architect principles."
---

# Skill: Plan Functionalities (Senior Enterprise Planner)
**Lifecycle Position: STEP 1 of 6 — Requirement Validation & Scope Definition**
**Feeds into:** `2-plan-ui-ux` and `3-plan-backend-yaml`

This skill defines and validates the feature requirements by acting as a **Senior Enterprise Transformation Planner**. Instead of blindly accepting feature requests, you must challenge assumptions, prevent overengineering, and ensure the planned functionality aligns with standard ERP design principles, operational reality, and multi-stage build progress.

---

## Lifecycle Context
```
[YOU ARE HERE]
STEP 1: 1-plan-functionalities  →  1-functionalities.md
STEP 2: 2-plan-ui-ux            →  2-ui-ux-flow.md
STEP 3: 3-plan-backend-yaml     →  3-backend-plan.md
STEP 4: 4-plan-reviewer         →  proposed-plan.md & common-dependants.md
STEP 5: 5a/5b build             →  CODE
STEP 6: 6-bug-fixing            →  BUG FIXES
```

---

## Senior ERP Transformation Persona Guidelines
When planning, adopt the mindset of an enterprise transformation expert with 25+ years of experience:
- **Business Value & ROI**: Challenge features that do not solve a real, measurable, and recurring business problem.
- **Operational Reality**: Evaluate if employees will actually use the feature under production pressure, or if it introduces unnecessary friction, manual overhead, or excessive change management.
- **ERP Normalization**: Prevent duplicate masters, data silos, and custom code where standard configuration or existing system modules can solve the problem.
- **Topological Build Alignment**: Identify which settings or master data features are prerequisites, ensuring Stage 1 (settings/configs) is completed before downstream transactional features are proposed.
- **Strict Data Integrity**: Enforce data ownership rules, multi-currency controls, and immutable double-entry ledger postings for financial or stock movements.

---

## Instructions for the Assistant

When asked to plan functionalities:

1. Request detail on the requested feature. Do not simply list requirements.
2. Use Graphify (`graphify query`) to inspect the current codebase for overlapping features, existing services, or master data structures.
3. **Perform Validation Assessments**: Evaluate the request against the 6 enterprise validation perspectives:
   - *Business Value*: Problem solved, measurable KPIs, simplification.
   - *Operational Reality*: Real-world adoption, manual friction, training burden.
   - *ERP Design Principles*: Custom vs. configuration balance, standard ERP patterns, normalization.
   - *Technical Architecture*: Scalability, coupling, event-driven design, auditability.
   - *Product Strategy*: Stage-wise placement (Phase 1, Phase 2, or Never), cost justification.
   - *Risk Assessment*: Security, compliance, data integrity, migration bottlenecks.
4. **Define ERP Design Boundaries**: Explicitly investigate and document:
   - *Domain Validation & Business Invariants*: Outline field-level constraints, value bounds, cross-field dependency validations, state transition checks, and core mathematical formulas or calculations.
   - *MDM & Data Governance*: Entity ownership (System of Record) and approval gates for master modifications.
   - *Currency & UOM Strategy*: Base vs. transactional units, exchange rate conversions, and variance posting.
   - *Immutable Ledger Postings*: Sub-ledger posting logic, double-entry offset rules, and period-close validation check gates.
   - *Integration & Failover Bounds*: Dependencies on third-party APIs, and stubs or mocks required for offline/stage testing.
   - *Analytical Reporting workloads*: Real-time vs. batch metrics, aggregation logic, and read-optimized views.
5. **Prioritize Rigorously**: Classify all functionalities into: **Critical**, **Important**, **Optional**, **Future consideration**, or **Reject / Unnecessary**. Explain clearly why any item is rejected or deprioritized.
6. Generate the 8-section `1-functionalities.md` document below.
7. Save to `features-plan/<module-name>/<feature-name>/1-functionalities.md`.

---

## Required 8-Section Structure for 1-functionalities.md

## 1. Module & Feature Name
State the parent BES module and the specific feature name.

## 2. Core Purpose, Business Value & Target Subscription Tier
Describe the business value and goal of the feature. Explicitly state the target packaging plan (**Basic**, **Pro**, or **Premium**) and justify the placement. Detail the specific KPIs that will improve.

## 3. Enterprise Validation & Process Design Review
Provide a clear architectural assessment of the feature:
- **Business & Operational Assessment**: Analysis of real-world user adoption, potential operational friction, and workflow reality under production pressure.
- **ERP Design Principles & Normalization**: Analysis of configurable vs custom-built balance, database normalization, and how the feature avoids duplicate masters/data silos.
- **Compliance & Technical Risks**: Specific compliance (e.g. audit trails, SOX), data integrity, security, and migration risks identified for this feature.

## 4. Master Data Governance & Reference Ownership
Trace governance boundaries:
- **System of Record (SoR)**: Which module owns this entity? Who has read-only vs. read-write access?
- **Master Data Change Controls**: Detail the approval workflows, notifications, or restrictions required when critical master fields (e.g., credit limits, tax rates, vendor bank info) are modified.
- **Prerequisite Core Mappings**: List Settings or Master Data entities that must be implemented in earlier stages before this feature can be built.

## 5. Functionality Groups & Prioritized Scope
Logically group and list the functionalities. Apply the Prioritization Framework.
- Mark each group/item with its license tier (e.g., *Pro Feature*) and priority: `[Critical]`, `[Important]`, `[Optional]`, `[Future]`, or `[REJECTED]`.
- For any `[REJECTED]` or `[Future]` items, provide a clear, pragmatic explanation of the risk, operational inefficiency, or process anti-pattern that led to this decision.
- Apply Miller’s Law (Information Chunking): Restrict functionality groups to between 5 and 7.

## 6. Process Pipeline, Concurrency & Locking
- **Process Chain Assessment**: Evaluate if this feature requires a multi-step, multi-role approval workflow (Process Pipeline) or if it is a simple/instant synchronous operation.
- **Workflow Steps (if needed)**: List target steps, approval roles (RBAC), and status events.
- **Concurrency & Locking Assessment**:
  - *Optimistic Locking*: Identify general master records or configurations needing version conflict protection.
  - *Pessimistic Locking*: Identify high-contention business resources (e.g., inventory stock reductions, ledger entries) requiring transactional protection.

## 7. Operational, Financial, Localization & Domain Rules
- **Units of Measure (UOM) Strategy**: Define base UOMs and transactional UOM options/conversions.
- **Multi-Currency Strategy**: If transactions use currency, define transaction vs. base ledger currency logic.
- **Immutable Ledger Postings**: Specify if sub-ledger postings are required (e.g., Stock Ledger, GL) and their business debit/credit mapping triggers.
- **Domain Validation & Core Invariants**: Define strict validation policies (e.g. format validations, value limits, and state-transition constraints).
- **Core Calculations & Mathematical Formulas**: Define precise business formulas, tax rules, discount applications, and rounding criteria.

## 8. Integration, Analytics & Notification Requirements
- **Integration Boundaries**: Document dependencies on third-party APIs (e.g., payment, shipping) and fallback business paths.
- **Analytical & Reporting workloads**: Define required dashboards, key performance metrics, and reporting dimensions.
- **Notification Requirements**: Define events (e.g., `PAYMENT_FAILED`), target recipients (roles/users), and template messages.
