---
name: 2-design-flow-ui
description: "PLANNING STEP 2: Design the complete UI/UX for a BES flow — landing page, step screens, Data Hub views, and component mapping."
---

# Skill: Design Flow UI/UX

## Purpose
Given a completed `1-flow-definition.md`, produce a comprehensive `2-flow-ui-design.md` that specifies every screen, layout, interaction, and component the flow needs.

## When to Use
- After `1-plan-flow` has produced the flow definition
- Run SECOND in the planning pipeline, BEFORE `3-plan-flow-backend`

## Prerequisites
- Read the flow's `1-flow-definition.md` (the input for this skill).
- Read Rule 2 (Frontend Architecture) for Warm Professional design system, component requirements, and UX laws.
- Read Rule 3 §4-6 for Flow Page Architecture, Data Hub layout, and Sidebar structure.
- Check `@BES/shared-ui` for existing components to reuse (use Graphify or grep).

---

## Execution Steps

### Step 1: Design the Flow Landing Page

Every flow has a landing page. Design it with:

```markdown
## Flow Landing Page: [Flow Name]

### Layout
- **Header**: Flow icon + name + description + "Start New [Flow]" CTA button (accent color)
- **Metrics Row**: 3-4 MetricCard components showing key KPIs
- **Activity Feed**: DataTable showing recent entities across all steps (combined view)
- **Quick Filters**: Tab-like status filters (All, Draft, Active, Completed, Overdue)

### Metric Cards
| # | Label | Data Source | Format |
|:--|:------|:-----------|:-------|
| 1 | [e.g., Open Quotes] | [API endpoint] | count |
| 2 | [e.g., Pending Orders] | [API endpoint] | count |
| 3 | [e.g., This Month Revenue] | [API endpoint] | currency |

### Activity Feed Columns
| Column | Source | Sortable | Filterable |
|:-------|:-------|:---------|:-----------|
| [e.g., Reference] | entity.ref_number | Yes | No |
| [e.g., Customer] | entity.customer.name | Yes | Yes |
| [e.g., Status] | entity.status | No | Yes (StatusChip) |
| [e.g., Amount] | entity.total_amount | Yes | No |
| [e.g., Date] | entity.updated_at | Yes | Yes |
```

### Step 2: Design Each Flow Step Screen

For every step in the pipeline, design the step view:

```markdown
## Step View: [Step Label]

### FlowStepper State
[Show which steps are completed/active/pending at this point]

### Content Layout
[Describe the main content area: form, data grid, confirmation view, etc.]

### Form Sections (if action step)
#### Section 1: [Section Name] (e.g., "Customer Details")
| Field | Type | Required | Validation | Default |
|:------|:-----|:---------|:-----------|:--------|
| Customer | SearchableSelect | Yes | Must exist in Data Hub | — |
| Reference | TextInput | Auto | Auto-generated (SO-XXXX) | Next in sequence |
| Date | DatePicker | Yes | Cannot be past date | Today |

#### Section 2: [Section Name] (e.g., "Line Items")
[DataTable with inline editing for line items]
| Column | Type | Editable | Computed |
|:-------|:-----|:---------|:---------|
| Product | SearchableSelect | Yes | — |
| Quantity | NumberInput | Yes | — |
| Unit Price | CurrencyInput | Yes | Auto from product |
| Subtotal | Currency | No | qty × price |

### Actions
- **Primary CTA**: [e.g., "Confirm Order →"] (accent color, bottom-right)
- **Secondary**: [e.g., "Save as Draft"] (outline style)
- **Destructive**: [e.g., "Cancel Quote"] (error color, requires confirmation)

### Validation Behavior
- Inline validation on blur for required fields
- CTA disabled until all required fields are valid
- Error summary at top of form if server-side validation fails
```

### Step 3: Design Data Hub Pages

For each Data Hub entity this flow feeds:

```markdown
## Data Hub: [Entity Name]

### DataTable Columns
| Column | Width | Sortable | Filterable | Format |
|:-------|:------|:---------|:-----------|:-------|
| Name | flex | Yes | Yes (search) | text |
| Status | 100px | No | Yes (StatusChip dropdown) | StatusChip |
| Balance | 120px | Yes | No | currency (decimal.js) |
| Last Activity | 140px | Yes | Yes (date range) | relative date |

### DetailPanel Tabs
| Tab | Content |
|:----|:--------|
| Overview | Key fields, editable form. Grouped in FormSections. |
| Transactions | DataTable of related flow entities (orders, invoices). Row click navigates to flow step. |
| Timeline | Chronological audit log. Created, updated, status changes. |

### Empty State
- **Illustration**: [describe the empty state visual]
- **Headline**: "No [entities] yet"
- **Subtext**: "Start a [flow name] to create your first [entity]"
- **CTA**: "Start [Flow]" → navigates to flow landing page
```

### Step 4: Map Components to @BES/shared-ui

List every component this flow needs and whether it exists:

```markdown
## Component Requirements

| Component | From shared-ui? | Custom? | Notes |
|:----------|:----------------|:--------|:------|
| FlowStepper | ✅ Yes | No | Standard 5-step stepper |
| DataTable | ✅ Yes | No | Used in landing, line items, Data Hub |
| DetailPanel | ✅ Yes | No | Data Hub entity details |
| FormSection | ✅ Yes | No | Step form layouts |
| StatusChip | ✅ Yes | No | Status badges |
| MetricCard | ✅ Yes | No | Landing page KPIs |
| SearchableSelect | ⚠️ Check | Maybe | Customer/Product picker |
| CurrencyInput | ⚠️ Check | Maybe | Financial amount input with decimal.js |
```

If a needed component doesn't exist in shared-ui, flag it as a **prerequisite build**.

### Step 5: Define Interactions & Micro-Animations

```markdown
## Interactions

### Flow Step Transitions
- **Forward**: Content slides left, next step slides in from right (--wp-motion-enter)
- **Backward**: Content slides right, previous step slides in from left (--wp-motion-enter)
- **FlowStepper**: Active dot scales up with spring animation (--wp-motion-spring)
- **Completion**: Completed step dot gets a checkmark with a subtle bounce

### Data Hub Row Click
- Row highlights on hover (--wp-stone-100 background)
- DetailPanel slides in from right (--wp-motion-enter)
- Content loads with skeleton shimmer (--wp-motion-enter)

### Form Validation
- Error fields: border transitions to --wp-error with shake animation (--wp-motion-spring)
- Success: field border briefly flashes --wp-success then returns to normal
```

---

## Output

Write the completed document to:
```
features-plan/flows/<flow_id>/2-flow-ui-design.md
```

## Quality Checklist
- [ ] Flow landing page has metrics, activity feed, and "Start New" CTA
- [ ] Every pipeline step has a detailed screen design
- [ ] Form sections follow Miller's Law (5-7 fields max per section)
- [ ] Primary CTA follows Fitts's Law (prominent, bottom-right)
- [ ] All Data Hub entities have DataTable columns and DetailPanel tabs defined
- [ ] Component requirements mapped — missing components flagged as prerequisites
- [ ] Interactions use ONLY the 3 motion primitives from the design system
- [ ] No inline styles described — all styling references design tokens
- [ ] Empty states have illustrations, headlines, and CTAs
- [ ] Accessibility: form labels, keyboard navigation, focus management noted
