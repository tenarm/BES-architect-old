---
trigger: always_on
description: Mandates querying the Graphify knowledge graph at graphify-out/ to establish context before planning, architecting, or coding.
---

# Rule 3: Use Graphify for Context

1. **Graphify as Source of Truth**: Before planning, architecting, or coding, you must use Graphify to establish the current state of the codebase.
2. **Primary Commands**:
   - `graphify query "<question>"`: Use this to discover existing architectures, components, or feature patterns.
   - `graphify path "<A>" "<B>"`: Use this to trace the dependency path between two features or modules (crucial for cross-module dependency planning).
   - `graphify explain "<concept>"`: Use this to understand specific nodes or components deeply.
3. **Graph Integrity**: After completing an implementation or modifying the folder structure, run `graphify update .` to ensure the AST map remains current.
4. **Dependency Resolution**: Rely on Graphify to identify if `@bes/shared-ui` components or core API endpoints already exist before proposing new ones.
