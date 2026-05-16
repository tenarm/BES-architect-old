## 1. Module
`settings`

## 2. Name
System Audit Log & Activity Trail

## 3. Description
The System Audit Log is the "Black Box" of the BES. It provides a permanent, immutable, and searchable record of every significant action taken within the system. This includes data mutations (create, update, delete), status changes, and administrative actions. The Audit Log is essential for compliance, internal controls, and forensic debugging. It directly powers the real-time activity feeds and the historical timelines visible in entity drawers across all modules.

## 4. Depends On
- **Core Kernel:**
  - `AuditLog` & `FieldChangeLog` models: Immutable storage.
  - `AuditService`: The centralized engine for recording entries.
  - `SSE Broadcaster`: For real-time activity streaming.
- **Cross-Module:**
  - Every module that uses `AuditService.log_action()`.
- **UI Libraries:**
  - `@bes/shared-ui`: `Table`, `Timeline`, `Skeleton`, `ProcessPipeline`.

## 5. Feature UI — The UI as Information
**Feature Name:** Global Activity Monitor
**UI Details:**
- **Global Audit Table:** Columns for `Timestamp`, `Actor`, `Module`, `Action`, `Entity Type`, `Description`.
- **Search & Filter Bar:** Filter by Module, User, Entity ID, or Date Range.
- **Audit Detail Drawer:**
  - **Summary Tab:** Who, when, and where.
  - **Changes Tab:** A side-by-side comparison or list of field-level changes (Old Value vs New Value).
  - **Traceability Tab:** Displays the `correlation_id` chain (e.g., see the event that triggered this action).
- **User Journey:**
  1. Admin goes to **Settings > Audit Logs**.
  2. Searches for "SalesOrder" entity type.
  3. Finds a "DELETE" action by a specific user.
  4. Opens the drawer to see exactly what the data was before it was deleted.

## 6. Sample Data Structure (YAML + JSON)
```yaml
# Audit Entry
id: "audit-uuid-001"
timestamp: "2023-10-27T10:00:00Z"
actor_name: "Admin User"
module: "finance"
entity_type: "finance.JournalEntry"
entity_id: "je-uuid-999"
action: "POSTED"
description: "Posted Journal Entry JE-001"
changes:
  status: { old: "Draft", new: "Posted" }
  posted_at: { old: null, new: "2023-10-27T10:00:00Z" }
```

```json
{
  "status": "success",
  "data": [
    {
      "id": "audit-uuid-001",
      "description": "Posted Journal Entry JE-001",
      "actor_name": "Admin User",
      "created_at": "2023-10-27T10:00:00Z"
    }
  ]
}
```

## 7. Required APIs
- `GET /api/v1/auth/audit`: Global search/list (Admin only).
- `GET /api/v1/auth/audit/{entity_type}/{entity_id}`: Entity-specific timeline (Used by drawers).
- `GET /api/v1/auth/audit/chain/{correlation_id}`: View full causal chain.
- `GET /api/v1/auth/audit/stream`: SSE endpoint for real-time dashboard updates.

## 8. Database Tables & Architecture
- **Table:** `core.audit_log` (Immutable, no updates).
- **Table:** `core.field_change_log` (Detailed mutations).
- **Design:** Uses an append-only architecture to ensure integrity. Linked to the `core.event_store` via `correlation_id`.

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `AUDIT_RECORDED` (via SSE broadcaster).
- **Listens To:**
  - ALL system events that require logging.

## 10. Business Rules & Validations
- **Immutability:** Audit records MUST NEVER be edited or deleted (no soft delete here).
- **Retention:** Supports archival policies (Day 3) but records are kept indefinitely by default.
- **Privacy:** Sensitive fields (like `hashed_password`) must be masked or excluded from `field_change_log`.

## 11. Security, Audit, and RBAC
- **Permission:** `settings:audit_log:read`. Only top-level admins should have global access.
- **Tamper Evidence:** Since the table is immutable and actor info is denormalized, it serves as a high-fidelity record.

## 12. Process Transparency & Workflow Pipeline
- **Right Panel (Drawer) UX:**
  - **Macro View (`ProcessPipeline`):** Not applicable for logs.
  - **Micro View (`Timeline`):** This feature IS the source of the `Timeline` component.
- **Shell Navigation:** Sidebar: **Settings > Security > Audit Logs**.

## 13. Technical Implementation Roadmap (Day 1)
- **Phase 1:** Core `AuditService` implementation (Backend) - **DONE**.
- **Phase 2:** Implement `GET /audit` search endpoints with high-performance indexing.
- **Phase 3:** Create the Global Audit Table UI.
- **Phase 4:** Build the "Change Comparison" component for the Drawer.
- **Phase 5:** Wire up the real-time Activity Stream on the Settings Home Dashboard.

## 14. Verification & QA Strategy
- **Search Check:** Filter by `actor_id` and verify only their actions appear.
- **Chain Check:** Verify that one user action correctly shows the cascading event effects via `correlation_id`.
- **Performance Check:** Verify fast loading even with 100k+ log entries (requires proper indexing on `entity_id` and `timestamp`).
- **Masking Check:** Verify that updating a user profile doesn't leak password hashes into the audit log.
