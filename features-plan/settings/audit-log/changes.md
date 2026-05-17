# Implementation Spec: System Audit Log & Activity Trail

## 1. Impacted Backend Files
| File Path | Change Type | Implementation Notes |
| :--- | :--- | :--- |
| `bes-backend/core/models.py` | `[NO CHANGE]` | `audit_log` and `field_change_log` tables exist. |
| `bes-backend/extensions/settings/schemas.py` | `[MODIFY]` | Add `AuditLogRead`, `FieldChangeRead` for API responses. |
| `bes-backend/extensions/settings/services.py` | `[MODIFY]` | Implement `AuditQueryService` for searching logs by correlation ID, user, module. Ensure masking is applied. |
| `bes-backend/extensions/settings/router.py` | `[MODIFY]` | Add `/audit` routes: `GET /`, `GET /{entity_type}/{entity_id}`, `GET /chain/{correlation_id}`, `GET /stream` (SSE). |
| `bes-backend/extensions/settings/manifest.py` | `[MODIFY]` | Export router. |
| `bes-backend/core/core/admin_permissions.json` | `[MODIFY]` | Add `settings:audit_log:read`. |

## 2. Impacted Frontend Files
| Nx Path | Change Type | `@bes/shared-ui` Components | Registration / Notes |
| :--- | :--- | :--- | :--- |
| `bes-frontend/libs/settings/src/lib/audit-log/` | `[NEW]` | `Table`, `Timeline`, `Drawer` | New feature directory. |
| `bes-frontend/libs/settings/src/lib/audit-log/GlobalAuditTable.tsx` | `[NEW]` | `Table` | High-performance paginated global view. |
| `bes-frontend/libs/settings/src/lib/audit-log/AuditDetailDrawer.tsx` | `[NEW]` | `Drawer`, `Timeline` | Display before/after diffs using `FieldChangeRead` data. |
| `bes-frontend/libs/settings/src/index.ts` | `[MODIFY]` | | Export routing. |

## 3. Integration Checklist
- [ ] **Permissions**: Add `"settings:audit_log:read"` to `admin_permissions.json`.
- [ ] **Performance**: Verify indexing on `entity_id` and `timestamp` in `core.audit_log` via `EXPLAIN ANALYZE`.
- [ ] **SSE Stream**: Verify `GET /stream` correctly streams `AUDIT_RECORDED` events to the frontend.
- [ ] **Masking Rules**: Test updating a user password and confirm it does not appear in `field_change_log`.

## 4. Implementation Order
`Schemas → AuditQueryService → APIs (Search & Filter) → SSE /stream Endpoint → Manifest → Frontend Library → Global Audit Table UI → Drawer Component (Diff View) → QA (Performance & Masking)`
