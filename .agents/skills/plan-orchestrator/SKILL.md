---
name: plan-orchestrator
description: "Orchestrates the complete 5-step flow planning pipeline for TenArm — from flow definition through to build approval."
---

# Skill: Flow Planning Orchestrator

## Purpose
Guides the complete planning pipeline for a new TenArm flow. This is the entry point skill — it coordinates running skills 1→2→3→4 in sequence, collecting user feedback at each stage.

## When to Use
- Starting the planning process for a new flow
- The user says "plan the Sell flow" or "let's plan a new workflow"

---

## Pipeline Overview

```
1-plan-flow          →  Output: 1-flow-definition.md
    ↓ (review)
2-design-flow-ui     →  Output: 2-flow-ui-design.md
    ↓ (review)
3-plan-flow-backend  →  Output: 3-flow-backend-plan.md
    ↓ (review)
4-review-flow        →  Output: proposed-plan.md
    ↓ (USER APPROVAL)
5-build-flow         →  Output: Working code
```

## How to Run

### Quick Start
1. User names the flow (e.g., "Sell", "Buy", "Stock")
2. Read `1-plan-flow/SKILL.md` and execute it
3. Present the output to user for review
4. If approved, read `2-design-flow-ui/SKILL.md` and execute it
5. Present the output to user for review
6. If approved, read `3-plan-flow-backend/SKILL.md` and execute it
7. Present the output to user for review
8. If approved, read `4-review-flow/SKILL.md` and execute it
9. Present the proposed-plan.md to user for FINAL APPROVAL
10. If approved, read `5-build-flow/SKILL.md` and execute it

### User Feedback Points
- After Step 1: "Does this flow definition capture your business process correctly?"
- After Step 2: "Does this UI design match what you envision?"
- After Step 3: "Does the backend plan cover all the data and logic?"
- After Step 4: "This is the final plan. Shall I build it?"

### All Output Goes To
```
features-plan/flows/<flow_id>/
├── 1-flow-definition.md
├── 2-flow-ui-design.md
├── 3-flow-backend-plan.md
├── proposed-plan.md
└── flow-definition.json
```

## Standard Flows to Plan

Reference these when the user asks "what flows should we build?"

| # | Flow | Priority | Phase |
|:--|:-----|:---------|:------|
| 1 | Sell | P0 | Phase 2 |
| 2 | Buy | P0 | Phase 2 |
| 3 | Stock | P0 | Phase 2 |
| 4 | Money | P1 | Phase 3 |
| 5 | People | P1 | Phase 4 |
| 6 | Customers (CRM) | P1 | Phase 4 |
| 7 | Manufacture | P2 | Phase 5 |
| 8 | Projects | P2 | Phase 5 |
| 9 | Assets | P2 | Phase 5 |
| 10 | Support | P2 | Phase 5 |

**Always start with Phase 1 (Design System + Settings) before planning any flows.**
