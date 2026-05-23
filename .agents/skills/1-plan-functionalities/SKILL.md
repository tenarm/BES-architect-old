---
name: 1-plan-functionalities
description: "PLANNING STEP 1: Create the functionalities list of a feature."
---

# Skill: Plan Functionalities
**Lifecycle Position: STEP 1 of 6 — Requirement**
**Feeds into:** `2-plan-ui-ux` and `3-plan-backend-yaml`

This skill kicks off the planning phase by explicitly outlining everything a feature does, ensuring nothing is missed. Its output (`1-functionalities.md`) is the primary foundation for the subsequent UI and Backend plans.

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

## About the BES Architecture
- **Stack**: React (Nx Monorepo) frontend, FastAPI (PDM Monorepo) backend, PostgreSQL.
- **Shell UI**: `apps/shell` dynamically loads licensed module libraries.
- **Licensing Core**: Built-in subscription tiers (**Basic**, **Pro**, **Premium**) defined in `packages.json`.
- **Graphify Context**: We use `graphify query` to search the codebase.
- **Rule Sets**: Follow established rules in `.agents/rules/`.

---

## Instructions for the Assistant

When the user asks to plan functionalities:

1. Ask the user for any specific details about the feature to analyze first.
2. Use Graphify (`graphify query`) to check for any existing related features or overlaps.
3. Review `core/core/packages.json` to align functionality groupings with our standard subscription plans.
4. Assess if the feature or any of its sub-functionalities require a multi-step, multi-role approval workflow (a **Process Chain/Pipeline**). If yes, explicitly map it out and document it under Section 5.
5. Generate the 6-section document below.
6. Save to `features-plan/<module-name>/<feature-name>/1-functionalities.md`. Create the folder if needed.

---

## Required 6-Section Structure

## 1. Module & Feature Name
State the parent BES module and the specific feature name.

## 2. Core Purpose & Target Subscription Tier
Describe the business value and goal of the feature within the BES system. Explicitly state the target packaging plan (**Basic**, **Pro**, or **Premium**) and justify the placement.

## 3. Functionality Groups
Logically group the functionalities (e.g., Core Setup, Workflow Actions, Dashboards/Reporting). Mark each group with its licensing level (e.g., *Pro Feature*, *Premium Integration*).
*Note: Apply Miller’s Law (Information Chunking). Restrict the total number of functionality groups to between 5 and 7 to keep the layout cognitive-friendly for beginner developers and users.*

## 4. Detailed Functionalities List
List the exact functionalities. For each, describe what it does. Keep definitions clear, modular, and beginner-friendly.
*Example: Chart of Accounts (COA)*
- *Add Account (Basic): Create a new account node.*
- *Deactivate Account (Basic): Soft delete the account.*
- *Advanced Cost Forecasting (Pro): Calculate multi-year cost center trend analytics.*

## 5. Process Pipeline & Concurrency Requirements
Explicitly evaluate and document:
- **Process Chain Assessment**: Does this functionality require a multi-step, multi-role, or approval-driven flow (e.g., clearance checks, multi-layered approvals, offboarding)? Or is it a simple/instant synchronous operation?
- **Workflow Steps (if needed)**: List target workflow steps, approval roles (RBAC) responsible for each step, status keys, and cross-module effects.
- **Concurrency & Locking Assessment**: Evaluate potential multi-user conflict scenarios:
  - *Optimistic Locking*: Identify general entities (e.g., master settings, customer profiles, product detail edits) that need version-based collision protection.
  - *Pessimistic Locking*: Identify transactional or critical resources (e.g., inventory deductions, double ledger postings, payment status transitions) that must use DB row locking (`FOR UPDATE`) to prevent double-processing.
- **Justification**: If no pipeline or specific locking is needed, justify why direct un-locked DB transactions are sufficient.


## 6. Notification Requirements
Explicitly identify the notification alerts triggered by this feature:
- **Trigger Events**: What action/status changes emit events that require notifying users (e.g. `SALES_ORDER_COMPLETED`)?
- **Channels**: Which channels should be default configured (e.g. `IN_APP` for basic notifications, `EMAIL` for high-importance alerts)?
- **Templates**: Draft the default Jinja2 template titles and bodies referencing event data (e.g., "New invoice for {{ customer_name }} of amount {{ total }} created").
- **Recipients**: Define target recipients (e.g. user_id of creator, members of a specific role, or a dynamic email path in the event payload).

---

### Execution Rules
- **Output file**: `features-plan/<module-name>/<feature-name>/1-functionalities.md`
- **Next step**: Run `2-plan-ui-ux`.
