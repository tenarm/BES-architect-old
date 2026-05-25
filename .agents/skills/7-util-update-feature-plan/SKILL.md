---
name: util-update-feature-plan
description: "PLANNING UTILITY: Update existing feature plans with newly requested owner features."
---

# Skill: Update Feature Plan
**Lifecycle Position: UTILITY — Plan Iteration**

This skill is invoked when the Owner identifies missed features, requests new functionalities, or decides to **remove/modify** features after the initial planning documents have been generated. It intelligently propagates the changes (additions, removals, or restructures) through the 4-step planning documents.

---

## Lifecycle Context
```
[PLAN ITERATION TRIGGERED]
Reads: 1-functionalities, 2-ui-ux-flow, 3-backend-plan, proposed-plan
Updates: 1-functionalities, 2-ui-ux-flow, 3-backend-plan, proposed-plan
Feeds into: 5a-frontend-build & 5b-backend-build (with updated plans)
```

---

## About the BES Architecture
- You must enforce the core rules defined in `.agents/rules/1-backend-architecture.md` and `2-frontend-architecture.md`.
- Ensure new or modified features properly utilize `BESBase`, standard API envelopes, licensing guards, and `@bes/shared-ui`.
- If a feature is removed or tier-limited, ensure the UI lock 🔒 and upgrade overlay requirements are integrated into the UI plan.

---

## Instructions for the Assistant

When the owner asks to add, remove, or modify functionalities in an existing plan:

1. Identify the target module and feature, and precisely what functionalities the owner wants to **add, remove, or modify**.
2. Read the existing planning documents in `features-plan/<module-name>/<feature-name>/`:
   - `1-functionalities.md`
   - `2-ui-ux-flow.md`
   - `3-backend-plan.md`
   - `proposed-plan.md`
3. Use Graphify (`graphify query`) to check how the request might impact existing code, database structures, or other features.
4. **Update all 4 documents** to weave in the changes seamlessly. Maintain the rigid structure required by each document:
    - **Update 1 (Functionalities)**: Add, remove, or edit items in the Functionalities List. Apply the Senior ERP Validation Persona to review new/modified requirements, update the Prioritization scope (Critical, Important, Optional, Future, Reject), and update any governance models, UOM/currency rules, ledger posting requirements, and integrations.
    - **Update 2 (UI/UX)**: Add, remove, or modify UI States, Process Chains, or menu items in the navigation tree. If a feature was downgraded or tier-restricted, specify where the lock indicator 🔒 and Upgrade Gate Overlay will reside.
    - **Update 3 (Backend)**: Add or delete database columns in the YAML Schema (ensuring soft-deletes and numeric precision). Create or remove REST APIs. Adjust the event subscriptions/emitters. Update the **RBAC Permissions & Licensing Guards** (e.g., adding or removing `require_licensed_feature` calls or `packages.json` mapping modifications).
    - **Update 4 (Review & Roadmap)**: Restructure the sequence in `proposed-plan.md` (e.g., adding/removing migrations, API tests, and UI steps). Update the Required External Dependencies and Exposed Reusable Assets sections in `proposed-plan.md` and `common-dependants.md` if dependencies, database references, or shareable assets change.
5. Provide a clear, structured summary of the changes made across all 4 documents.

---

### Execution Rules
- **Output**: Direct file edits to the 4 planning documents in `features-plan/<module-name>/<feature-name>/`.
- **Next step**: Prompt the user to proceed to `5a-frontend-build` or `5b-backend-build` once the updated plan is approved.

