# Implementation Spec: Number Series / Sequence Management

## 1. Impacted Backend Files
| File Path | Change Type | Implementation Notes |
| :--- | :--- | :--- |
| `bes-backend/core/models.py` | `[MODIFY]` | Add `number_series` table inheriting `BESBase`. Include code, prefix, suffix, padding, current_value, reset_frequency. |
| `bes-backend/extensions/settings/schemas.py` | `[MODIFY]` | Add `NumberSeriesUpdate`, `NumberSeriesRead`, `NumberSeriesPreview`. |
| `bes-backend/extensions/settings/services.py` | `[MODIFY]` | Implement atomic `get_next_number` with DB locking/returning. Handle dynamic token replacement `{YYYY}`. |
| `bes-backend/extensions/settings/router.py` | `[MODIFY]` | Add `/number-series` routes. `GET`, `PATCH`, `POST /preview`. Internal `/next/{code}`. |
| `bes-backend/extensions/settings/events.py` | `[MODIFY]` | Emit `SETTINGS_SERIES_UPDATED`. |
| `bes-backend/core/core/admin_permissions.json` | `[MODIFY]` | Add `settings:number_series:write`. |

## 2. Impacted Frontend Files
| Nx Path | Change Type | `@bes/shared-ui` Components | Registration / Notes |
| :--- | :--- | :--- | :--- |
| `bes-frontend/libs/settings/src/lib/number-series/` | `[NEW]` | `Table`, `Drawer`, `Form` | New feature directory. |
| `bes-frontend/libs/settings/src/lib/number-series/SeriesTable.tsx` | `[NEW]` | `Table` | Shows list of active sequence configs and their previews. |
| `bes-frontend/libs/settings/src/lib/number-series/SeriesDrawer.tsx` | `[NEW]` | `Drawer`, `Form` | Pattern editing and live preview fetching. |
| `bes-frontend/libs/settings/src/index.ts` | `[MODIFY]` | | Export routing. |

## 3. Integration Checklist
- [ ] **Database**: Create `core.number_series` via Alembic.
- [ ] **Permissions**: Add `"settings:number_series:write"` to `admin_permissions.json`.
- [ ] **Atomic Safety**: Write unit test simulating concurrent `get_next_number` calls to ensure no duplicates.
- [ ] **Pattern Engine**: Validate correct token parsing for `{YYYY}`, `{MM}`.
- [ ] **Licensing**: Verify `READONLY_EXTENSIONS` disables the pattern config save button.

## 4. Implementation Order
`Core DB Migration → Schemas → Atomic Sequence Service → APIs (RBAC) → Preview Endpoint Logic → Manifest → Frontend Library → UI Components & Live Preview Form → QA (Concurrency tests)`
