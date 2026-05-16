# Modular BES Factory - Master Blueprint

This document bridges the gap between the "Raw Python" setup and "Global SaaS" scale.

## 1. High-Level System Architecture
The system follows a Kernel-and-Plugin (Core + Extension) pattern. The Core acts as the "Operating System," while Extensions act as the "Applications."

| Component | Responsibility | Repository |
| :--- | :--- | :--- |
| **Core Kernel** | Auth, DB Pooling, Event Bus, RBAC Logic, Base Models. | `bes-core` (Private Package) |
| **Extensions** | Domain logic (HR, Sales), Industry-specific tables, API Routes. | `client-xxx-extension` (Per Client) |
| **Shell UI** | User Login, Module Routing, Global Navigation. | Nx Monorepo App |

## 2. The Data Foundation (SQLModel & PostgreSQL)
Every table in the system is built on a unified foundation to ensure consistent auditing and multi-org support.

### A. The Base Model (`BESBase`)
- **UUID Primary Keys**: Prevents record enumeration and ensures safe data merging.
- **Audit Trail**: `created_at`, `updated_at`, `created_by` (UTC-based).
- **Expansion Joint**: A `metadata_` JSONB column.
  - Used for "Ghost Foreign Keys" (referencing Extension tables without hard SQL links).
  - Used for client-specific custom fields without schema changes.

### B. Hierarchical Multi-Tenancy
- **Tenant Isolation**: One dedicated PostgreSQL Database per Client.
- **Organizational Scoping**: A `subsidiary_id` on every table to differentiate between branches/subsidiaries within a group company.
- **Row-Level Safety**: A backend `ContextVar` ensures that queries automatically filter by `subsidiary_id` based on the user's active session.

## 3. RBAC: Context-Aware Security
Permissions are not just "Yes/No" but are dependent on how the action is triggered.
- **The Master JSON**: Defined in the Core; contains granular rules, including manual (User-initiated) vs. auto_trigger (System-initiated) flags.
- **The Simplified JSON**: Computed by the backend upon login and sent to the React UI via a `/bootstrap` endpoint. It is a flat map of booleans for high-speed UI rendering.
- **Permission Elevation**: Allows a user (e.g., Sales Staff) to trigger a process that updates a restricted resource (e.g., Chart of Accounts) via an internal System Context, while blocking them from manual edits.

## 4. The Event-Based "Nervous System"
To keep modules decoupled, the Core provides an internal Event Bus.
- **Mechanism**: An asynchronous "Emit & Subscribe" pattern.
- **Decoupling**: The Sales module emits an `ORDER_CONFIRMED` event. It does not know if the Finance module is listening.
- **Safety**: If the Finance module is not licensed (and thus not loaded), the event simply expires without error.
- **Evolution**: Start as an in-memory asyncio bus; move to a Persistent Outbox (Database) or Redis for "Day 2" reliability.

## 5. Deployment & Operational Workflow (Day 1)
For Day 1, we prioritize low overhead and speed by using a Raw Python + PM2 stack.

### A. Selective Deployment
Only the repositories for licensed modules are cloned/installed on the client’s server. Unpaid code is never physically present on the client's machine, protecting your IP.

### B. The Startup Sync
The backend process performs a "Self-Healing" boot:
1. **Load**: Reads `ACTIVE_EXTENSIONS` from `.env`.
2. **Register**: Dynamically imports extension routers and models.
3. **Sync**: Executes `SQLModel.metadata.create_all()` to automatically generate new tables in the client's private database.

## 6. Day 1 "Licensing" & Extension Management
For the initial phase, we use a simple `.env` file driven approach:
```env
# Day 1: Manual Module Toggle
ACTIVE_EXTENSIONS=hr,finance
READONLY_EXTENSIONS=sales
DATABASE_URL=postgresql://user:pass@localhost/bes_acme
PORT=8001
```
- **Read-Only Mode**: Downgraded modules are loaded in read-only mode, keeping data visible but safe from modification.
- **Bootstrap Endpoint**: Core provides `GET /api/v1/bootstrap` returning active/readonly modules for the Shell UI.

## 7. Day 1 Core Guiding Principles
### A. Decimal Precision (The "Money" Rule)
- **Backend**: Enforce `Numeric`/`Decimal` (4 decimal places).
- **Frontend**: Use `decimal.js` or `big.js`. Never use floating-point for currency.

### B. Standardized API Response Structure
```json
{
  "status": "success",
  "data": { ... },
  "metadata": { "count": 100, "page": 1 },
  "error": null
}
```

### C. Soft Deletes (`is_deleted` vs. Hard Delete)
Physical deletion is forbidden. Use `is_deleted` flags.

### D. Centralized File Storage Strategy
A "Storage Service" provides standard `upload()`/`download()` functions.

### E. System-Wide Unit of Measure (UOM) Registry
A centralized `uoms` table in the Core to prevent terminology fragmentation.
