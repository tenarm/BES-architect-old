## 1. Module
`settings`

## 2. Name
Company / Entity Setup

## 3. Description
The Company / Entity Setup feature is the foundational configuration module for BES. It allows administrators to define the organizational structure by creating and managing legal entities (subsidiaries). Each entity stores critical metadata such as legal names, tax identifiers, registration details, addresses, and localization defaults (currency, fiscal year). This data serves as the anchor for the `subsidiary_id` used throughout the system for multi-tenant data isolation and financial consolidation.

## 4. Depends On
- **Core Kernel:**
  - `BESBase`: For standardized audit fields and UUID primary keys.
  - `Bootstrap`: To provide the active subsidiary context to the Shell.
  - `RBAC`: To restrict access to organizational settings (`settings:subsidiary:*`).
- **Cross-Module:**
  - `Finance`: For default currency and fiscal year alignment.
- **UI Libraries:**
  - `@bes/shared-ui`: `Table`, `Drawer`, `Form`, `Skeleton`, `Timeline`, `ProcessPipeline`.

## 5. Feature UI — The UI as Information
**Feature Name:** Subsidiary Manager
**UI Details:**
- **Entity Table:** Displays `name`, `legal_name`, `tax_id`, `city`, `country`, and `status`.
- **Creation/Edit Drawer:**
  - **General Tab:** Legal Name, Short Name, Entity Type (e.g., Parent, Branch, Subsidiary).
  - **Registration Tab:** Registration Number, Tax ID (VAT/GST), Incorporation Date.
  - **Address Tab:** Full physical and billing address fields.
  - **Localization Tab:** Default Currency (ISO code), Timezone, Fiscal Year Start Month.
- **Empty State:** "No entities configured. Start by adding your primary legal entity."
- **Loading State:** Skeleton rows in the table.

**User Journey:**
1. User navigates to **Settings > Company Setup**.
2. User clicks **"Add Entity"** button.
3. A right-panel **Drawer** opens with a multi-tab form.
4. User fills in legal and tax details.
5. User clicks **"Save"**.
6. System emits `SETTINGS_SUBSIDIARY_CREATED`, updates the table via SSE, and logs an audit entry.

## 6. Sample Data Structure (YAML + JSON)
```yaml
# Subsidiary Entity
id: "550e8400-e29b-41d4-a716-446655440000"
name: "Acme Corp India"
legal_name: "Acme Technologies Private Limited"
registration_number: "U72200KA2023PTC123456"
tax_id: "29AAAAA0000A1Z5"
currency_code: "INR"
fiscal_year_start: 4  # April
address:
  line1: "123 Tech Park"
  city: "Bangalore"
  state: "Karnataka"
  country: "India"
metadata_:
  industry: "Software"
  logo_url: "storage://branding/acme_logo.png"
```

```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Acme Corp India",
    "legal_name": "Acme Technologies Private Limited",
    "subsidiary_id": "550e8400-e29b-41d4-a716-446655440000",
    "currency_code": "INR",
    "tax_id": "29AAAAA0000A1Z5"
  }
}
```

## 7. Required APIs
- `GET /api/v1/settings/subsidiaries`: List all entities (paginated).
- `GET /api/v1/settings/subsidiaries/{id}`: Detailed view.
- `POST /api/v1/settings/subsidiaries`: Create a new entity.
- `PATCH /api/v1/settings/subsidiaries/{id}`: Partial update.
- `DELETE /api/v1/settings/subsidiaries/{id}`: Soft delete (sets `is_deleted=True`).

## 8. Database Tables & Architecture
- **Table:** `core.subsidiaries`
  - Inherits `BESBase`.
  - `name`: `String` (Display name).
  - `legal_name`: `String` (Official name for invoices).
  - `registration_number`: `String` (Unique).
  - `tax_id`: `String`.
  - `currency_code`: `String(3)`.
  - `fiscal_year_start`: `Integer` (1-12).
  - `address_json`: `JSONB` (Stores structured address).
- **Hub-and-Spoke:** The `subsidiaries` table lives in the `core` schema as it is the master tenant-definition table.

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `SETTINGS_SUBSIDIARY_CREATED`
  - `SETTINGS_SUBSIDIARY_UPDATED`
  - `SETTINGS_SUBSIDIARY_DELETED`
- **Listens To:**
  - None (Top-level configuration).

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion is FORBIDDEN.
- **Uniqueness:** `registration_number` must be unique across non-deleted entities.
- **Integrity:** An entity cannot be deleted if it is the only entity in the system or if it has active users assigned.
- **Validation:** `currency_code` must be a valid ISO 4217 code.

## 11. Security, Audit, and RBAC
- **Permission Format:** `settings:subsidiary:read`, `settings:subsidiary:write`.
- **Licensing Mode:** If the `settings` module is `READONLY_EXTENSIONS`, the "Add Entity" button is hidden, and form fields are disabled.
- **Audit Trail:**
  - Log `user_id`, `timestamp`, and diff of changes for `legal_name`, `tax_id`, and `currency_code`.
  - High-severity audit entry for `is_deleted` transitions.

## 12. Process Transparency & Workflow Pipeline
- **Right Panel (Drawer) UX:**
  - **Macro View (`ProcessPipeline`):** `Draft` → `Pending Verification` → `Active`.
  - **Micro View (`Timeline`):** Vertical activity feed in the "History" tab showing who changed which field.
- **Shell Navigation:** Sidebar: **Settings > Organization > Company Setup**.

## 13. Technical Implementation Roadmap (Day 1)
- **Phase 1:** Define `Subsidiary` model in `core.models` and run migrations.
- **Phase 2:** Implement CRUD services in `bes-backend/extensions/settings`.
- **Phase 3:** Create `SubsidiaryTable` and `SubsidiaryDrawer` components in `@bes/settings`.
- **Phase 4:** Register the route in `SettingsModule` and link to Shell Sidebar.
- **Phase 5:** Wire up SSE updates for the table list.

## 14. Verification & QA Strategy
- **Scenario 1:** Create an entity and verify it appears in the table via SSE without manual refresh.
- **Scenario 2:** Attempt to create an entity with a duplicate `registration_number` and verify 400 error.
- **Scenario 3:** Verify that `subsidiary_id` in the new entity matches its own `id` (as it is the root).
- **Scenario 4:** Check `READONLY_EXTENSIONS` mode disables the Save button.
- **Scenario 5:** Verify Audit Timeline shows the "Incorpation Date" change correctly.
