---
name: build-bugs
description: "POST-BUILD UTILITY: Document post-build bugs into features-plan/build-bugs.md and compile actionable prevention safeguards in .agents/rules/5-build-bug-prevention-rules.md in a scannable, minimal way."
---

# Skill: Build Bugs and Resolutions Recorder
**Lifecycle Position: UTILITY — Post-Build Resolution & Quality Assurance**

This skill is invoked when a post-build bug has been identified and resolved (or is in the process of being documented). It records the bug details inside the global `features-plan/build-bugs.md` registry and dynamically updates the compiled prevention rules in `.agents/rules/5-build-bug-prevention-rules.md`. Future build agents must inspect these files before commencing any backend or frontend builds to prevent repeating past mistakes.

---

## Lifecycle Context
```
[POST-BUILD BUG ENCOUNTERED / RESOLVED]
Reads: features-plan/build-bugs.md, .agents/rules/5-build-bug-prevention-rules.md
Updates: features-plan/build-bugs.md, .agents/rules/5-build-bug-prevention-rules.md
Feeds into: Future 5a-frontend-build & 5b-backend-build steps as a prerequisite lookup
```

---

## Instructions for the Assistant

When the developer or process invokes this skill, perform the following steps:

### 1. Identify Target Feature & Extract Bug Details
Identify or ask the user for the following concise information:
- **Date**: Format: `YYYY-MM-DD` (or use the current local date).
- **Module & Feature**: The parent module and active feature names (e.g., `settings` and `company_setup`).
- **Title**: A short, descriptive title of the bug.
- **Layer**: The architectural layer affected (e.g., `Backend / Config`, `Frontend`, `Database`). Specify a classification tag from `[Backend]`, `[Frontend]`, `[Database]`, `[Config]`, or `[Events]`.
- **Symptom**: Concise explanation or error message snippet. Escape any pipe (`|`) characters.
- **Root Cause**: Technically precise explanation of why it failed (1-2 sentences).
- **Fix**: Summary of the changes made to resolve it.
- **Prevention Rule**: An actionable safeguard/check for future build agents to completely avoid this issue.
- **Verification Test**: Path to the automated test case (e.g., unit/integration test) written to prevent regression, or specify "Manual" if automated validation is not feasible. If no test exists, prompt the developer or offer to write one.

### 2. Update the Consolidated Registry (`features-plan/build-bugs.md`)
- Read the existing contents of `features-plan/build-bugs.md`.
- Locate the correct `## <Module> Module` heading. (Create it if missing).
- Locate the `### Feature: <Feature Name>` subheading. (Create it if missing).
- Under the feature heading, locate or create the table:
  ```markdown
  #### Bugs & Resolutions
  | Date / Session | Bug Title | Layer | Symptom | Root Cause | Fix & Prevention Rule | Verification Test |
  | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
  ```
- Format the new bug entry as a single row in the table, escaping pipe characters (`|`) to prevent breaking the markdown table:
  `| Date | Title | Layer | Symptom | Root Cause | Fix. **Prevention Rule:** RuleText | [Test File Name](file:///path/to/test) or "Manual" |`
- Append the row to the table. Do not duplicate entries if the exact same bug title exists in the table.

### 3. Update the Prevention Rules (`.agents/rules/5-build-bug-prevention-rules.md`)
- Read `.agents/rules/5-build-bug-prevention-rules.md`.
- Find or create a matching category heading under a clear division (e.g. `## 1. Process Definition Files (Config/Backend)`, `## 2. API Endpoints (Backend)`, or `## 3. Component and State Management (Frontend)`).
- Add the actionable prevention safeguard as a bullet point. Make sure the instruction is clear, highly technical, and immediately actionable for future LLM build agents.
- Ensure the rule starts with the layer classification tag and a bold keyword:
  `- **[LayerTag] Keyword**: Rule details... (e.g., `- **[Config] Envelope Wrapping**: ...`)`

### 4. Update the Knowledge Graph
- Run `graphify update .` to ensure the registry and rule files are immediately cataloged in the knowledge graph.

### 5. Final Confirmation
- Provide the user with a concise confirmation of the recorded bug, the compiled prevention rule, and the associated verification test. Keep the response minimal and highly informative.
