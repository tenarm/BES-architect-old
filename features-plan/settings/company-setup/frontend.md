# Company / Entity Setup

## 1. Module
Settings

## 2. Name
Company / Entity Setup

## 3. Description
The Company / Entity Setup feature is the absolute foundational component of the Business Execution System (BES). It establishes the primary organizational profile and, most importantly, configures the multi-entity (multi-subsidiary) architecture that governs data isolation across the entire system. This module defines the global system currency, statutory tax identifiers, financial year configurations, and the hierarchical structure of subsidiaries, branches, and cost centers. Every other module in BES relies on this setup to enforce data scoping and regional compliance.

## 4. Depends on
- **Core (System Foundation):** This feature directly manipulates the highest-level master data in the `core` schema, specifically the `subsidiaries` table, which forms the basis of the entire `BESBase` security and isolation model.
- **Other Dependencies:**
  - `@bes/shared-ui` for structured multi-step wizards and nested list views.
  - `Finance Module` (Initialization): Creating a subsidiary automatically triggers the creation of a default Chart of Accounts and Fiscal Year calendar in the Finance module.

## 5. Feature name, details - the UI as information
**Feature Name:** Organization & Subsidiary Configuration
**Details & UI Information:**
- **Resource List:** A hierarchical tree or list view of the organizational structure (e.g., Parent Company -> North America Subsidiary -> Chicago Branch). Columns: `Entity Name`, `Registration Code`, `Base Currency`, `Timezone`, and `Status` (Active/Inactive).
- **Creation/Edit Drawer (Entity Profile):** A multi-tab Right Panel (Drawer) for configuring an entity:
  - **General Tab:** `Legal Name`, `Trading Name`, `Registration Number`, `Tax ID`, and `Parent Entity` (for hierarchical structuring).
  - **Localization Tab:** `Base Currency`, `System Language`, `Timezone`, `Date Format`, and `Fiscal Year Start Month`.
  - **Addresses Tab:** `Registered Address`, `Headquarters Address`, and `Billing Address`.
  - **Branding Tab:** Upload fields for `Company Logo` and `Favicon`, plus definitions for `Primary Brand Color` (used to customize the UI theme for that specific subsidiary).
  - **History/Activity Tab:** Contains the vertical `Timeline` for audit traceability regarding structural changes.
- **Empty States:** This should theoretically never be empty after initial deployment, but if accessed in a fresh install: "Welcome to BES. Start by defining your primary legal entity."
- **Loading States:** Shimmer effects for the hierarchy tree and configuration panels.

**User Journey & UX Flow:**
1. User navigates to **Settings > Company Setup**.
2. User clicks **"Add Subsidiary"** -> Right-panel configuration wizard opens.
3. User defines the **Legal Name** and selects the **Base Currency** (Warning: Base Currency cannot be changed once transactions exist).
4. User uploads **Branding** assets.
5. User clicks **"Initialize Entity"** -> System creates the `core.subsidiaries` record, emits a `settings.subsidiary.created` event, and generates the necessary background finance structures (default COA).
6. The new subsidiary is now available in the global context switcher at the top of the BES Shell, allowing users with appropriate access to switch their active session scope.

## 6. YAML or sample data structure
### YAML Schema
```yaml
Subsidiary:
  id: uuid
  parent_id: uuid (optional, for nested hierarchies)
  legal_name: string
  trading_name: string
  registration_code: string
  tax_id: string
  status: enum [ACTIVE, INACTIVE]
  localization:
    base_currency_id: uuid
    timezone: string
    date_format: string
    fiscal_year_start: integer (1-12)
  branding:
    logo_url: string
    primary_color_hex: string
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "sub-990e8400-e29b-41d4-a716-446655441111",
    "parent_id": null,
    "legal_name": "Globex Corporation Intl.",
    "trading_name": "Globex",
    "registration_code": "REG-US-12345",
    "tax_id": "EIN-98-7654321",
    "status": "ACTIVE",
    "localization": {
      "base_currency_id": "curr-usd-uuid",
      "timezone": "America/New_York",
      "date_format": "MM/DD/YYYY",
      "fiscal_year_start": 1
    },
    "branding": {
      "logo_url": "https://storage.bes.local/logos/globex.png",
      "primary_color_hex": "#0F4C81"
    },
    "created_at": "2026-05-16T16:00:00Z"
  },
  "metadata": {
    "version": "1.0"
  },
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/core/subsidiaries`**: Fetch the hierarchical list of all entities.
- **GET `/api/v1/core/subsidiaries/{id}`**: Fetch detailed configuration for a specific entity.
- **POST `/api/v1/core/subsidiaries`**: Create a new subsidiary/entity.
- **PUT `/api/v1/core/subsidiaries/{id}`**: Update entity configurations (branding, addresses).
- **GET `/api/v1/core/currencies`**: Fetch available global currencies for assignment as Base Currency.

## 8. Database Tables & Architecture
### Schema: Core Module
#### Table: `core.subsidiaries`
- **Inherits**: `BESBase` (Note: This is the table that all other `subsidiary_id` foreign keys in the system point to).
- `parent_id`: UUID (FK to `core.subsidiaries.id`, nullable)
- `legal_name`: String (UNIQUE, NOT NULL)
- `registration_code`: String
- `tax_id`: String
- `base_currency_id`: UUID (FK to `core.currencies`, NOT NULL)
- `timezone`: String
- `status`: String (Default: 'ACTIVE')
- `branding_json`: JSONB

#### Table: `core.currencies`
- **Inherits**: `BESBase`
- `code`: String (e.g., 'USD', 'EUR')
- `symbol`: String
- `decimal_places`: Integer (Default: 4, per the Money Rule)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `settings.subsidiary.created`: Signals Finance to create default Ledgers and HR to create default employment policies.
  - `settings.subsidiary.deactivated`: Immediately halts transactional creation across all modules for that specific scope.
  - `settings.branding.updated`: Signals the frontend shell to hot-reload the UI theme/colors for active sessions in that subsidiary.
- **Listens To:**
  - This module primarily acts as the foundational publisher; it rarely listens to transactional events.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion of a subsidiary is strictly forbidden once created, as it compromises the integrity of millions of transactional records. Use `status = 'INACTIVE'`.
- **Immutability of Base Currency:** Once a subsidiary is initialized and a single financial transaction (e.g., a GL Entry) is posted, the `base_currency_id` becomes strictly read-only and immutable to prevent ledger corruption.
- **Scoping Bedrock:** All other database tables in BES must enforce a `subsidiary_id` column. This Settings module manages the source data for those scoping rules.

## 11. Security, Audit, and RBAC
- **Roles:**
  - `Super Admin / System Administrator`: The only role permitted to create new subsidiaries or alter global tax IDs and base currencies.
  - `Entity Admin`: Can update branding, local addresses, and minor settings for their specific subsidiary, but cannot create new entities.
- **Context Switching:** Users granted access to multiple subsidiaries utilize a dropdown in the global shell header to switch contexts. The backend middleware intercepts every request to ensure the active `JWT` token matches the requested `subsidiary_id`.
- **Audit Trail:** Every configuration change to a subsidiary's profile is logged in the system audit trail, providing a complete history of organizational restructuring.

## 12. Process Transparency & Workflow Pipeline
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** Not typically used for entity setup, as creation is immediate. However, an "Initialization Status" tracker could be used if background provisioning (e.g., creating hundreds of default COA accounts) takes time.
  - **Micro View (`Timeline`):** Placed in the "History/Activity" tab of the Entity drawer. Logs structural changes: "Subsidiary created by Admin Bob", "Base Currency locked after first GL transaction", "Company Logo updated".
- **Navigation:** Main Sidebar > Settings > Company Setup.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `core.subsidiaries` and `core.currencies`. Establish the system-wide middleware that enforces `subsidiary_id` scoping on all API requests.
- **Phase 2: Core Logic & APIs:** Implement the Subsidiary Service and the immutable checks for Base Currency.
- **Phase 3: Frontend Infrastructure:** Register the Settings module in the `@bes/settings` library and hook it into the global shell (as recently added in `main.tsx`).
- **Phase 4: UI Development:** Build the organizational hierarchy tree and the configuration drawer. Implement the global Context Switcher dropdown in the ERP Shell header.
- **Phase 5: Event Integration:** Wire up the `subsidiary.created` event to auto-provision necessary finance and HR defaults.

## 14. Verification & QA Strategy
- **Scoping Enforcement Check:** Create `Subsidiary A` and `Subsidiary B`. Verify that a user assigned only to A cannot switch their session context to B.
- **Immutability Check:** Post a single GL Journal Entry in Subsidiary A. Attempt to change Subsidiary A's Base Currency via the UI and API; verify the system rejects both attempts.
- **Functional Scenarios:**
  1. Create a nested subsidiary (Branch X under Parent Y) and verify the hierarchy tree renders correctly.
  2. Change the primary brand color for Subsidiary B and verify the UI shell theme updates dynamically upon saving.
- **Integration Test:** Verify that creating a new subsidiary automatically emits the correct event, resulting in the successful automated creation of a default Chart of Accounts for that new entity.
