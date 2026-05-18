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
- **Graphify Context**: We use `graphify query` to search the codebase.
- **Rule Sets**: Follow established rules in `.agents/rules/`.

---

## Instructions for the Assistant

When the user asks to plan functionalities:

1. Ask the user for any specific details about the feature to analyze first.
2. Use Graphify (`graphify query`) to check for any existing related features or overlaps.
3. Generate the 4-section document below.
4. Save to `features-plan/<module-name>/<feature-name>/1-functionalities.md`. Create the folder if needed.

---

## Required 4-Section Structure

## 1. Module & Feature Name
State the parent BES module and the specific feature name.

## 2. Core Purpose
Describe the business value and goal of the feature within the BES system.

## 3. Functionality Groups
Logically group the functionalities (e.g., Core Setup, Workflow Actions, Dashboards/Reporting).

## 4. Detailed Functionalities List
List the exact functionalities. For each, describe what it does.
*Example: Chart of Accounts (COA)*
- *Add Account: Create a new account node.*
- *Deactivate Account: Soft delete the account.*

---

### Execution Rules
- **Output file**: `features-plan/<module-name>/<feature-name>/1-functionalities.md`
- **Next step**: Run `2-plan-ui-ux`.
