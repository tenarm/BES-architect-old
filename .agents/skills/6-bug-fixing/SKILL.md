---
name: 6-bug-fixing
description: "IMPLEMENTATION STEP 3: Review documents and user-reported bugs, then fix them."
---

# Skill: Bug Fixing
**Lifecycle Position: STEP 6 of 6 — Maintenance**

This skill ensures that bug fixes align with the original feature plan and do not violate architectural rules.

---

## Lifecycle Context
```
STEP 1: 1-plan-functionalities  →  1-functionalities.md
STEP 2: 2-plan-ui-ux            →  2-ui-ux-flow.md
STEP 3: 3-plan-backend-yaml     →  3-backend-plan.md
STEP 4: 4-plan-reviewer         →  proposed-plan.md & common-dependants.md
STEP 5: 5a/5b build             →  CODE
[YOU ARE HERE]
STEP 6: 6-bug-fixing            →  BUG FIXES
```

---

## About the BES Architecture
- Bug fixes must adhere to the same stringent rules as feature development.
- Do not bypass the `BESBase` models, the API envelope, or the `@bes/shared-ui` components to create a quick fix.
- Verify changes against Graphify AST logic.

---

## Instructions for the Assistant

When asked to fix a bug:

1. Ask the user for the specific bug report and which module/feature it affects.
2. Read the corresponding feature documents in `features-plan/<module-name>/<feature-name>/` to understand the intended architecture.
3. Use Graphify (`graphify query` or `graphify explain`) to trace the bug's context in the codebase.
4. Formulate a fix that adheres to all Frontend and Backend architectural constraints.
5. Implement the fix and verify it resolves the issue without introducing regressions.
6. If the bug arose post-build, invoke the `build-bugs` skill to document the bug in `features-plan/build-bugs.md` and append an actionable prevention safeguard to `.agents/rules/5-build-bug-prevention-rules.md` to prevent future build agents from repeating this issue.

---

### Execution Rules
- **Output**: Code changes resolving the bug.
- **Next step**: Update `graphify update .` to reflect the AST changes, and invoke the `build-bugs` skill to record the post-build resolution details.
