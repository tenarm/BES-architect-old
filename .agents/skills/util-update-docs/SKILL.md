---
name: util-update-docs
description: "UTILITY STEP: Systematically keep all project documentation (ARCHITECTURE.md, backend/frontend rules, and READMEs) updated and consistent whenever core patterns or APIs change."
---

# Skill: Update Documentation

This utility skill provides instructions on how to maintain, audit, and update documentation files across the BES codebase when architectural or design changes are introduced.

---

## 1. Documentation Map & Scopes

The codebase contains several primary files representing different scopes of the project:

| Documentation File | Repository Path | Scope / Content |
| :--- | :--- | :--- |
| **Master Blueprint** | [ARCHITECTURE.md](file:///Users/bvk/BVK_Workspace/BES/ARCHITECTURE.md) | High-level system design, layering rules, tenant model, package tiers, core database structures, and dynamic registry systems. |
| **Backend Rules** | [.agents/rules/1-backend-architecture.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/1-backend-architecture.md) | Coding conventions, strict layering rules (models, schemas, services, routers, events), multi-tenancy requirements, API standards, and notification setups. |
| **Frontend Rules** | [.agents/rules/2-frontend-architecture.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/2-frontend-architecture.md) | Frontend directory layout, Nx monorepo rules, UI registration, styling, state management, component registries, and upgrade lockout styles. |
| **Deployments Guide** | [deployment_flow.md](file:///Users/bvk/BVK_Workspace/BES/deployment_flow.md) | Docker, deployment pipelines, multi-tenant databases provisioning, and build instructions. |
| **Workspace README** | [README.md](file:///Users/bvk/BVK_Workspace/BES/README.md) | High-level overview of workspace packages, developer tooling, and basic workspace command outlines. |
| **Backend Workspace README** | [bes-backend/README.md](file:///Users/bvk/BVK_Workspace/BES/bes-backend/README.md) | PDM setup, installation, running client instances, database seed operations, and coding rule directories. |

---

## 2. Trigger Events for Doc Updates

You MUST trigger this skill and update the relevant documentation whenever any of the following code modifications are executed:

1. **Kernel/Core database schema changes**: E.g. adding columns to `BESBase` or introducing a new system table.
   * *Target Docs*: [ARCHITECTURE.md](file:///Users/bvk/BVK_Workspace/BES/ARCHITECTURE.md), [.agents/rules/1-backend-architecture.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/1-backend-architecture.md)
2. **API/HTTP response envelopes or error handling rules**: E.g. changing exception interceptors or error responses.
   * *Target Docs*: [ARCHITECTURE.md](file:///Users/bvk/BVK_Workspace/BES/ARCHITECTURE.md), [.agents/rules/1-backend-architecture.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/1-backend-architecture.md)
3. **Core capabilities and helper utilities**: E.g. introducing locking helpers, storage integrations, or event bus enhancements.
   * *Target Docs*: [ARCHITECTURE.md](file:///Users/bvk/BVK_Workspace/BES/ARCHITECTURE.md), [.agents/rules/1-backend-architecture.md](file:///Users/bvk/BVK_Workspace/BES/.agents/rules/1-backend-architecture.md)
4. **Client instance configuration overrides**: E.g. changes in client onboarding or template generation scripts.
   * *Target Docs*: [ARCHITECTURE.md](file:///Users/bvk/BVK_Workspace/BES/ARCHITECTURE.md), [bes-backend/README.md](file:///Users/bvk/BVK_Workspace/BES/bes-backend/README.md)

---

## 3. How to Update Documentation

When updating documentation:
1. **Locate and Analyze**: Find all files matching the changed scopes using the Documentation Map above.
2. **Draft Modifications**: Use precise markdown syntax. Incorporate new code snippets if they show standard usages.
3. **Use Alerts**: Emphasize safety constraints, performance guidelines, or setup steps using alerts (e.g. `> [!IMPORTANT]` or `> [!WARNING]`).
4. **Link to Code Symbols**: Create clickable markdown links to target classes or files (e.g. [BESBase](file:///Users/bvk/BVK_Workspace/BES/bes-backend/core/core/models.py#L11)).
5. **Update Knowledge Graph**: After updating files, run `graphify update .` to rebuild the AST and semantic nodes, ensuring search tools remain in sync.
