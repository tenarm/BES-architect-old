---
name: verify-build
description: "Post-build verification checklist — backend tests, frontend lint/build, integration checks, and bug prevention compliance."
---

# Skill: Post-Build Verification

This skill runs a systematic verification checklist after building a new feature. It catches common issues before they reach the user.

---

## Prerequisites
- Backend and/or frontend code has been written for the feature
- The module has a `proposed-plan.md` defining what was built

---

## Verification Modes

Run the applicable mode(s) based on what was built:

---

### Mode A: Backend Verification

#### A1. Run Module Tests
```bash
cd bes-backend
pdm run pytest extensions/<module_name>/tests/ -v --tb=short
```
**Expected**: All tests pass. If no tests exist, flag this as a gap.

#### A2. Attempt Application Startup
```bash
cd bes-backend/instances/msme
pdm run uvicorn msme.main:app --port 8099 --log-level warning
```
**Expected**: Server starts without import errors, model registration errors, or manifest failures. Ctrl+C after successful boot.

#### A3. Verify API Endpoints
For each new route in the module's `router.py`:

1. **StandardResponse Envelope**: Every endpoint must return `{ status, data, metadata, error }`:
   ```bash
   curl -s http://localhost:8099/api/v1/<module>/<entity> \
     -H "Authorization: Bearer <token>" | python -m json.tool
   ```

2. **Pagination**: List endpoints must accept `?page=1&page_size=10` and return paginated metadata.

3. **RBAC**: Request without auth token must return `401 Unauthorized`:
   ```bash
   curl -s http://localhost:8099/api/v1/<module>/<entity>  # No auth header
   # Expected: 401
   ```

4. **Soft Delete**: DELETE endpoint must set `is_deleted=True`, NOT physically remove the row:
   ```bash
   # After DELETE, verify row still exists in DB with is_deleted=True
   ```

#### A4. Validate Process Definitions
Check that process definition JSONs parse correctly:
```bash
cd bes-backend
python -c "
from core.processes import load_process_definitions
defs = load_process_definitions()
for pid, pdef in defs.items():
    print(f'✓ {pid}: {len(pdef.get(\"steps\", []))} steps')
"
```

#### A5. Cross-Check Bug Prevention Rules
Read `.agents/rules/5-build-bug-prevention-rules.md` and verify EACH applicable rule:

| Rule | Check |
|:---|:---|
| **[Config] Envelope Wrapping** | Process JSON is wrapped in `{ "process_id": { ... } }` dict |
| **[Config] Process Step Validation** | Steps use `id` (not `stepId`), have `type` and `statusEvent` |
| **[Config] Conform to Process Definition Schemas** | Uses `"label"` (not `"name"`), defines `"entity"` |
| **[Backend] Preserve Boilerplate Imports** | No broken imports from overwritten files |

---

### Mode B: Frontend Verification

#### B1. Lint Check
```bash
cd bes-frontend
npx nx lint <module-name>
```
**Expected**: No lint errors.

#### B2. Run Module Tests
```bash
cd bes-frontend
npx nx test <module-name>
```
**Expected**: All tests pass.

#### B3. Full Shell Build
```bash
cd bes-frontend
npx nx build shell
```
**Expected**: Build succeeds with no errors. This catches missing exports, type errors, and broken imports.

#### B4. ComponentRegistry Key Verification
Verify that all registered component keys match the `RESOURCE_NAMES` mapping in `libs/shared-ui/src/lib/constants.ts`:

```bash
# The shell builds lookup keys as: `Route_${activeItem}` where activeItem = RESOURCE_NAMES[key]
# Verify each ComponentRegistry.registerLazy key matches this pattern
grep -r "registerLazy" bes-frontend/libs/<module-name>/src/
grep -r "RESOURCE_NAMES" bes-frontend/libs/shared-ui/src/lib/constants.ts
```

#### B5. Cross-Check Bug Prevention Rules

| Rule | Check |
|:---|:---|
| **[Frontend] Display-Name Key Matching** | Registry keys use `Route_${RESOURCE_NAMES['key']}`, not hand-crafted aliases |
| **[Frontend] Template Literal Formatting** | No Python-style formatters (`:02d`, `%s`) in JS/TS |
| **[Frontend] localStorage Token Key** | Uses `'bes_token'` (not `'bes_access_token'` or other variants) |
| **[Frontend] Sidebar Module Gating** | Module in `allModulesList` AND initializer in `auth-store.ts` |

#### B6. Subscription Gate Check
If the module has premium features:
- Verify lock indicators (🔒) render for Basic tier
- Verify `UpgradeGateOverlay` opens on locked feature click
- Test by setting `localStorage.setItem('bes_plan', 'basic')` in browser console

#### B7. Inline Style Audit
Scan for inline styles in reusable components (Rule 2 §6 violation):
```bash
grep -rn "style={{" bes-frontend/libs/<module-name>/src/ --include="*.tsx"
```
**Expected**: Inline styles only in page-level components, never in shared/reusable components.

---

### Mode C: Integration Verification

Only run this mode when both backend and frontend are complete for the feature.

#### C1. Full Stack Startup
```bash
# Terminal 1: Backend
cd bes-backend/instances/msme && pdm run uvicorn msme.main:app --port 8000 --reload

# Terminal 2: Frontend
cd bes-frontend && npm run dev
```

#### C2. Login Flow
- Login with valid credentials
- Verify JWT token stored in `localStorage` as `bes_token`
- Verify bootstrap data loaded (sidebar, permissions, plan)

#### C3. Module Visibility
- New module appears in sidebar with correct icon
- Clicking the module navigates to the home page
- TabGroup layout matches existing module patterns

#### C4. CRUD Operations
- **Create**: Fill form and submit → record appears in grid
- **Read**: Grid loads with pagination → detail drawer opens on row click
- **Update**: Edit via drawer → changes persist after refresh
- **Delete**: Soft delete → record disappears from grid but exists in DB

#### C5. Process Pipeline
- Process definition loads via SSE or API call
- `FloatingProcessPipeline` renders correctly with step statuses
- Status transitions update in real-time

---

## Verification Summary Template

After running all applicable checks, provide a summary:

```markdown
## Build Verification Results — <Module> / <Feature>

### Backend: ✅ / ❌
- [ ] Tests: X passed, Y failed
- [ ] Startup: Clean / Errors
- [ ] API Envelope: Correct / Missing
- [ ] Pagination: Working / Missing
- [ ] RBAC: Enforced / Missing
- [ ] Process Defs: Valid / Invalid
- [ ] Bug Prevention: All checked / Issues found

### Frontend: ✅ / ❌
- [ ] Lint: Clean / X errors
- [ ] Tests: X passed, Y failed
- [ ] Shell Build: Success / Failed
- [ ] Registry Keys: Correct / Mismatched
- [ ] Subscription Gates: Working / Missing
- [ ] Inline Styles: Clean / Violations found

### Integration: ✅ / ❌ / Not Tested
- [ ] Login: Working
- [ ] Module Visible: Yes
- [ ] CRUD: All operations work
- [ ] Process Pipeline: Renders correctly
```
