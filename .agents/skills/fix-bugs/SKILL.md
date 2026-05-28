---
name: fix-bugs
description: "Diagnose, fix, document, and prevent bugs — combining bug resolution with prevention rule compilation."
---

# Skill: Bug Fixing & Prevention

This skill combines bug diagnosis/fixing with documentation and prevention rule compilation. It ensures bugs are fixed correctly, documented in the registry, and prevented from recurring.

---

## Lifecycle Context
```
[BUG REPORTED / ENCOUNTERED]
     ↓
Phase 1: Diagnose & Fix   → Code changes resolving the bug
     ↓
Phase 2: Document & Prevent → build-bugs.md + 5-build-bug-prevention-rules.md
     ↓
Phase 3: Verify            → graphify update + regression check
```

---

## Phase 1: Diagnose & Fix

### 1. Gather Bug Report
Ask the user for (or identify from context):
- Which **module/feature** is affected
- What is the **symptom** (error message, unexpected behavior, crash)
- Steps to **reproduce** (if known)

### 2. Research Context
1. Read the corresponding feature documents in `features-plan/<module-name>/<feature-name>/` to understand the intended architecture.
2. Read the global bug registry `features-plan/build-bugs.md` — check if this bug (or a similar one) has been seen before.
3. Read `.agents/rules/5-build-bug-prevention-rules.md` — check if a prevention rule already exists but was not followed.
4. Use Graphify (`graphify query` or `graphify explain`) to trace the bug's context in the codebase and identify affected files.

### 3. Formulate & Implement Fix
- The fix MUST adhere to all Frontend and Backend architectural rules (Rule 1, Rule 2).
- Do NOT bypass `BESBase` models, the API envelope, `@bes/shared-ui` components, or domain exception patterns for a quick fix.
- Verify the fix resolves the issue without introducing regressions.

---

## Phase 2: Document & Prevent

### 4. Extract Bug Details
Collect the following information:
- **Date**: `YYYY-MM-DD` (use current local date)
- **Module & Feature**: Parent module and feature names (e.g., `settings` / `company_setup`)
- **Title**: Short, descriptive bug title
- **Layer**: Classification tag: `[Backend]`, `[Frontend]`, `[Database]`, `[Config]`, or `[Events]`
- **Symptom**: Concise error message or behavior description (escape pipe `|` characters)
- **Root Cause**: Technically precise 1-2 sentence explanation
- **Fix**: Summary of changes made
- **Prevention Rule**: Actionable safeguard for future build agents
- **Verification Test**: Path to automated test, or `"Manual"` if not feasible

### 5. Update Bug Registry (`features-plan/build-bugs.md`)
1. Read the existing `features-plan/build-bugs.md`.
2. Locate the correct `## <Module> Module` heading (create if missing).
3. Locate the `### Feature: <Feature Name>` subheading (create if missing).
4. Under the feature heading, locate or create the table:
   ```markdown
   #### Bugs & Resolutions
   | Date / Session | Bug Title | Layer | Symptom | Root Cause | Fix & Prevention Rule | Verification Test |
   | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
   ```
5. Append the bug as a new row. Do NOT duplicate entries with the same title.

### 6. Update Prevention Rules (`.agents/rules/5-build-bug-prevention-rules.md`)
1. Read `.agents/rules/5-build-bug-prevention-rules.md`.
2. Find or create a matching category heading (e.g., `## 1. Process Definition Files (Config/Backend)`, `## 2. ComponentRegistry Route Keys (Frontend)`).
3. Add the prevention safeguard as a bullet starting with layer tag and bold keyword:
   ```
   - **[LayerTag] Keyword**: Rule details...
   ```
   Example: `- **[Config] Envelope Wrapping**: Always wrap process definition JSON files...`

---

## Phase 3: Verify

### 7. Update Knowledge Graph
Run `graphify update .` to ensure the registry and rule files are cataloged.

### 8. Regression Check
- If a test was written, run it to confirm it passes.
- If the fix touched backend code: `cd bes-backend && pdm run pytest extensions/<module>/tests/ -v`
- If the fix touched frontend code: `cd bes-frontend && npx nx test <module-name>`

### 9. Final Confirmation
Provide the user a concise summary:
- ✅ Bug fixed (what was changed)
- 📝 Documented in `build-bugs.md`
- 🛡️ Prevention rule added to Rule 5 (if systemic)
- 🧪 Verification test status

---

## Execution Rules
- **Output**: Code changes + updated `build-bugs.md` + optionally updated `5-build-bug-prevention-rules.md`
- **Architectural Compliance**: All fixes must comply with Rules 1-2. No shortcuts.
- **Prevention Feedback Loop**: The prevention rules feed directly into future `build-frontend` and `build-backend` skills as Step 1 prerequisite reading.
