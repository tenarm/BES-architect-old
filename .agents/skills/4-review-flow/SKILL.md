---
name: 4-review-flow
description: "PLANNING STEP 4: Review all flow planning documents against BES rules, verify dependencies, and produce the approved proposed-plan.md."
---

# Skill: Review Flow Plan

## Purpose
Given the completed `1-flow-definition.md`, `2-flow-ui-design.md`, and `3-flow-backend-plan.md`, review them for architectural compliance, dependency safety, and completeness. Produce a final `proposed-plan.md` ready for user approval.

## When to Use
- After steps 1-3 of the planning pipeline are complete
- Run FOURTH, BEFORE `5-build-flow`

## Prerequisites
- Read ALL three planning documents for this flow.
- Read ALL rules (1-5) and the graphify rule.
- Read `features-plan/common-dependants.md` for existing dependencies.

---

## Execution Steps

### Step 1: Rule Compliance Audit

Check each planning document against every applicable rule:

#### Backend Compliance (Rule 1)
- [ ] All models inherit from BESBase
- [ ] Financial fields use Decimal(20,4), not float
- [ ] Services don't raise HTTPException — only domain exceptions
- [ ] Services own transaction boundaries — only services call commit
- [ ] Events emitted AFTER commit
- [ ] State machine transitions enforced in service layer
- [ ] No direct cross-module imports — all via Event Bus
- [ ] API responses use StandardResponse envelope
- [ ] All list endpoints support PaginationParams
- [ ] Route prefix follows `/api/v1/<module>` convention
- [ ] Import ordering: stdlib → third-party → core → current module

#### Frontend Compliance (Rule 2)
- [ ] All components use `--wp-*` design tokens (Warm Professional)
- [ ] No inline styles for reusable components — CSS modules only
- [ ] Forms follow Miller's Law (5-7 fields per section)
- [ ] Primary CTAs follow Fitts's Law (prominent, bottom-right)
- [ ] All components from shared-ui — no one-off alternatives
- [ ] TypeScript strict mode — no `any` types
- [ ] Error boundaries on flow root components
- [ ] Accessibility: labels, keyboard nav, contrast, focus management

#### Flow Compliance (Rule 3)
- [ ] Flow has a landing page with metrics, activity feed, and "Start New" CTA
- [ ] Every step has ID, type, entity, statusEvent, and dependsOn
- [ ] Pipeline supports customization (skippable steps identified)
- [ ] Data Hub entities follow DataTable + DetailPanel pattern
- [ ] Events follow `{FLOW}_{STEP}_{ACTION}` naming
- [ ] Cross-module side effects mapped as event subscribers

#### Planning Compliance (Rule 4)
- [ ] Documents focus on domain logic — no framework mechanics repeated
- [ ] Dependencies are identified with existing/stub status
- [ ] Build order respects dependency phases

#### Build Bug Prevention (Rule 5)
- [ ] ComponentRegistry keys match the naming convention (Flow_* / DataHub_*)
- [ ] Design tokens use `--wp-*` prefix, not old `--ui-*`
- [ ] localStorage token key is `'bes_token'`
- [ ] Process definition JSON files are envelope-wrapped

### Step 2: Dependency Verification

Use Graphify to verify that referenced entities, services, and APIs actually exist:

```bash
graphify query "Does Customer model exist in core?"
graphify query "Does BaseRepository have get_with_lock method?"
graphify query "Does EventBus exist in core.events?"
```

For every dependency marked as "Existing" in the backend plan, verify it genuinely exists.

For dependencies marked as "Needs building", verify they're not already built or planned in another flow.

### Step 3: Cross-Flow Impact Assessment

Check `features-plan/common-dependants.md`:
- Does this flow depend on anything not yet built?
- Does this flow expose anything other flows need?
- Are there circular dependencies?

### Step 4: Generate Proposed Plan

Combine all verified information into `proposed-plan.md`:

```markdown
# Flow: [Display Name] — Proposed Implementation Plan

## Summary
[1-2 paragraph overview of what this flow does and how it maps to the architecture]

## User Review Required
[Any decisions that need user input — e.g., "Should credit limit check be a hard block or a warning?"]
[Use > [!IMPORTANT] alerts for critical decisions]

## Build Prerequisites
| Prerequisite | Status | Action |
|:-------------|:-------|:-------|
| [e.g., Number sequence service] | ❌ Not built | Build as part of this flow |
| [e.g., Customer CRUD API] | ✅ Exists | No action |

## Implementation Checklist

### Backend
- [ ] Models: [list files to create/modify]
- [ ] Schemas: [list files]
- [ ] Services: [list files + key methods]
- [ ] Router: [list endpoints]
- [ ] Events: [list event types]
- [ ] Tests: [list test files]

### Frontend
- [ ] Flow landing page: `libs/flows/<flow>/`
- [ ] Step views: [list each step's component]
- [ ] Data Hub pages: [list each entity]
- [ ] Shared-UI prerequisites: [list any components to build first]

### Configuration
- [ ] Flow definition JSON: `features-plan/flows/<flow_id>/flow-definition.json`
- [ ] Permission entries: [list permissions to add to admin_permissions.json]
- [ ] Seed data: [list seed scripts needed]

## Verification Plan
### Automated Tests
- [list exact commands to run]
### Manual Verification
- [list what to check visually]

## Dependency Update
[Changes to add to common-dependants.md]
```

### Step 5: Update Common Dependants

Add this flow's dependencies and exposed assets to `features-plan/common-dependants.md`:

#### A. Required External Dependencies (Inbound)
| Source Flow/Module | Dependency | Description | Phase |
| :--- | :--- | :--- | :--- |

#### B. Exposed Assets (Outbound)
| Asset | Consumers | Description | Notes |
| :--- | :--- | :--- | :--- |

---

## Output

Write the completed documents to:
```
features-plan/flows/<flow_id>/proposed-plan.md
features-plan/common-dependants.md (updated)
```

**Request user feedback on the proposed-plan.md before proceeding to Step 5 (build).**

## Quality Checklist
- [ ] Every rule section has been checked (no skipped checks)
- [ ] Graphify used to verify at least 3 key dependencies
- [ ] common-dependants.md updated with inbound and outbound
- [ ] proposed-plan.md has clear implementation checklist
- [ ] Prerequisites identified with build-or-skip decision
- [ ] User review items highlighted with > [!IMPORTANT] alerts
- [ ] Verification plan includes both automated and manual checks
