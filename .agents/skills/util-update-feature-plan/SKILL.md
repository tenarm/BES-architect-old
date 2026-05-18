---
name: util-update-feature-plan
description: "PLANNING UTILITY: Update existing feature plans with newly requested owner features."
---

# Skill: Update Feature Plan
**Lifecycle Position: UTILITY — Plan Iteration**

This skill is invoked when the Owner identifies missed features or requests new functionalities after the initial plans have been generated. It intelligently merges the new requirements into the existing 4-step planning documents.

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
- Ensure new features properly utilize `BESBase`, the standard API envelope, Hub-and-Spoke MDM, and `@bes/shared-ui`.

---

## Instructions for the Assistant

When the owner asks to update a plan with a new feature:

1. Identify the target module and feature, and precisely what new functionality the owner wants to add.
2. Read the existing planning documents in `features-plan/<module-name>/<feature-name>/`:
   - `1-functionalities.md`
   - `2-ui-ux-flow.md`
   - `3-backend-plan.md`
   - `proposed-plan.md`
3. Use Graphify (`graphify query`) to check how the new request might impact existing code or architecture.
4. **Update all 4 documents** to weave in the new feature seamlessly. Maintain the rigid structure required by each document:
   - **Update 1**: Add the new item to the Functionalities List and adjust the Core Purpose if necessary.
   - **Update 2**: Integrate into the Information Architecture, UI States, and Process Chain.
   - **Update 3**: Update the YAML DB Schema, add required REST APIs, and define new Events/RBAC.
   - **Update 4**: Modify the Implementation Sequence in `proposed-plan.md` to include the new DB migrations, APIs, and UI steps. Also update `common-dependants.md` if the new feature introduces cross-module dependencies.
5. Provide a summary of the additions made across the plans.

---

### Execution Rules
- **Output**: Direct file edits to the 4 planning documents in `features-plan/<module-name>/<feature-name>/`.
- **Next step**: Prompt the user to proceed to `5a-frontend-build` or `5b-backend-build` once the plan is approved.
