# Implementation Spec: Company / Entity Setup

## 1. Impacted Backend Files
| File Path | Change Type | Implementation Notes |
| :--- | :--- | :--- |
| `bes-backend/core/models.py` | `[MODIFY]` | Ensure `subsidiaries` table inherits `BESBase`. Include standard columns (name, legal_name, registration_number, tax_id, currency_code, fiscal_year_start, address_json). |
| `bes-backend/extensions/settings/schemas.py` | `[NEW]` | Create `SubsidiaryCreate`, `SubsidiaryUpdate`, `SubsidiaryRead` schemas. Exclude protected fields in `*Create`. |
| `bes-backend/extensions/settings/services.py` | `[NEW]` | CRUD logic. Validate `registration_number` uniqueness, ISO currency code. Soft delete checks. |
| `bes-backend/extensions/settings/router.py` | `[NEW]` | Thin FastAPI layer. `GET /subsidiaries`, `POST`, `PATCH`, `DELETE` with `PaginationParams` and `require_permission`. Handle 403 for `READONLY_EXTENSIONS`. |
| `bes-backend/extensions/settings/events.py` | `[NEW]` | Emits `SETTINGS_SUBSIDIARY_CREATED`, `UPDATED`, `DELETED`. |
| `bes-backend/extensions/settings/manifest.py` | `[NEW]` | Subclass `ExtensionManifest`. Expose router. |
| `bes-backend/core/core/admin_permissions.json` | `[MODIFY]` | Add `settings:subsidiary:write`, `settings:subsidiary:read`. |

## 2. Impacted Frontend Files
| Nx Path | Change Type | `@bes/shared-ui` Components | Registration / Notes |
| :--- | :--- | :--- | :--- |
| `bes-frontend/libs/settings/src/lib/company-setup/` | `[NEW]` | `Table`, `Drawer`, `Form`, `ProcessPipeline` | New feature directory. |
| `bes-frontend/libs/settings/src/lib/company-setup/SubsidiaryTable.tsx` | `[NEW]` | `Table`, `Skeleton` | Display entities. Handles SSE updates for live data. |
| `bes-frontend/libs/settings/src/lib/company-setup/SubsidiaryDrawer.tsx` | `[NEW]` | `Drawer`, `Form`, `Timeline` | Multi-tab form for Create/Edit. Read-only disable logic. |
| `bes-frontend/libs/settings/src/index.ts` | `[MODIFY]` | | Export routing for Company Setup. |
| `bes-frontend/apps/shell/src/app/ComponentRegistry.ts` | `[MODIFY]` | | Register lazy-loaded settings module if not already done. |

## 3. Integration Checklist
- [ ] **Database**: Run `alembic revision --autogenerate -m "Add core.subsidiaries"` and upgrade.
- [ ] **Permissions**: Add `"settings:subsidiary:read"` and `"settings:subsidiary:write"` to `admin_permissions.json`.
- [ ] **Bootstrap**: Verify these permissions are correctly loaded in `GET /api/v1/bootstrap`.
- [ ] **Events**: Verify `SETTINGS_SUBSIDIARY_CREATED` is emitted upon successful `POST`.
- [ ] **Licensing**: Simulate `READONLY_EXTENSIONS=true` and confirm UI Save button is disabled and `POST` API returns 403.

## 4. Implementation Order
`Core Master Data (models.py) → Schemas → Services → APIs (RBAC) → Events → Manifest → Frontend Library → Shell Registration → UI Components → SSE Integration → QA`
