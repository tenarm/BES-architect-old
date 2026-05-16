# Item / Material Master

## 1. Module
Inventory

## 2. Name
Item / Material Master

## 3. Description
The Item / Material Master is the central repository for all products, components, and services handled by the Business Execution System (BES). It provides a unified definition for every physical and digital asset, ensuring consistency across sales, purchasing, manufacturing, and inventory modules. By separating global identity (core) from module-specific configurations (spokes), the Item Master enables detailed tracking of specifications, valuation methods, unit of measurements, and inventory control parameters.

## 4. Depends on
- **Core (Master Data):** Relies on the `core` schema for base item identity and shared attributes like UOM (Unit of Measure) and Item Categories.
- **Finance Module:** For linking items to default General Ledger accounts (Revenue, Expense, and Inventory Asset accounts).
- **Core Integration:** Core provides the unified `BESBase` foundation, allowing the `core.items` table to act as the single source of truth for item identifiers while the `inventory` module extends this with parameters like reorder points and lead times.
- **Other Dependencies:** 
  - `@bes/shared-ui` for standardized Drawer-based forms and resource lists.
  - `Image Storage Service` for handling product images and technical diagrams.

## 5. Feature name, details - the UI as information
**Feature Name:** Item Profile & Inventory Settings
**Details & UI Information:** 
- **Resource List:** A comprehensive table of all items. Columns: `Item Name`, `SKU/Code`, `Category`, `UOM`, `On Hand Quantity`, and `Status` (Active/Inactive).
- **Creation/Edit Drawer:** A multi-tab Right Panel (Drawer):
  - **General Tab:** `Item Name`, `SKU`, `Internal Description`, `Item Category`, `Base UOM`, and `Tax Category`.
  - **Inventory Tab:** `Valuation Method` (FIFO/LIFO/Weighted Average), `Min/Max Stock Levels`, `Reorder Point`, `Safety Stock`, and `Default Warehouse`.
  - **Finance Tab:** `Income Account`, `Expense Account`, and `Inventory Asset Account` (Lookup from COA).
  - **Purchasing/Sales Tab:** `Purchase UOM`, `Sales UOM`, `Default Vendor`, and `Lead Time`.
  - **History/Activity Tab:** Contains the vertical `Timeline` for audit traceability.
- **Empty States:** "No items found. Create your first item to begin managing stock."
- **Loading States:** Shimmer effects for the item attributes and skeleton loaders for the stock summary.

**User Journey & UX Flow:**
1. User navigates to **Inventory > Item Master**.
2. User clicks **"New Item"** -> Drawer opens.
3. User fills in **General Identity** (saved to `core.items`).
4. User switches to **Inventory Tab** to set reorder thresholds and valuation logic (saved to `inventory.item_details`).
5. User clicks **"Save"** -> System validates the SKU uniqueness, emits an `inventory.item.created` event, and refreshes the list.
6. User can later view the **Timeline** to see when an item's reorder point was adjusted or when its valuation method changed.

## 6. YAML or sample data structure
### YAML Schema
```yaml
Item:
  id: uuid
  subsidiary_id: uuid
  name: string
  sku: string
  category_id: uuid
  base_uom_id: uuid
  status: enum [ACTIVE, INACTIVE]
  inventory_details:
    valuation_method: enum [FIFO, LIFO, AVG]
    reorder_point: decimal(20,4)
    min_stock: decimal(20,4)
    max_stock: decimal(20,4)
    default_warehouse_id: uuid
  finance_details:
    income_account_id: uuid
    expense_account_id: uuid
    asset_account_id: uuid
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "item-550e8400",
    "subsidiary_id": "sub-990e8400",
    "name": "MacBook Pro M3",
    "sku": "HW-LAP-MBP-001",
    "category_id": "cat-hardware-uuid",
    "base_uom_id": "uom-each-uuid",
    "status": "ACTIVE",
    "inventory_details": {
      "valuation_method": "FIFO",
      "reorder_point": "10.0000",
      "min_stock": "5.0000",
      "max_stock": "50.0000",
      "default_warehouse_id": "wh-main-uuid"
    },
    "created_at": "2026-05-16T10:00:00Z"
  },
  "metadata": {},
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/inventory/items`**: Paginated list of items with search and category filtering.
- **GET `/api/v1/inventory/items/{id}`**: Detailed view including inventory and finance extensions.
- **POST `/api/v1/inventory/items`**: Atomic creation of an item across core and inventory schemas.
- **PUT `/api/v1/inventory/items/{id}`**: Update item specifications.
- **PATCH `/api/v1/inventory/items/{id}/status`**: Activate or deactivate an item.

## 8. Database Tables & Architecture
### Schema: Core Module
#### Table: `core.items`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `name`: String (NOT NULL)
- `sku`: String (UNIQUE, NOT NULL)
- `category_id`: UUID (FK to `core.item_categories`)
- `base_uom_id`: UUID (FK to `core.uoms`)
- `status`: String (Default: 'ACTIVE')

### Schema: Inventory Module
#### Table: `inventory.item_details`
- **Inherits**: `BESBase`
- `item_id`: UUID (NOT NULL, FK to `core.items.id`)
- `subsidiary_id`: UUID (NOT NULL)
- `valuation_method`: String (Default: 'FIFO')
- `reorder_point`: Numeric(20,4)
- `min_stock`: Numeric(20,4)
- `max_stock`: Numeric(20,4)
- `default_warehouse_id`: UUID

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:** 
  - `inventory.item.created`: Published when a new SKU is registered.
  - `inventory.item.updated`: Published when attributes or reorder points are changed.
  - `inventory.item.status_changed`: Notifies Sales and Purchase to update product availability.
- **Listens To:** 
  - `inventory.stock.updated`: Triggers a UI refresh of the "On Hand Quantity" in the resource list.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion is forbidden. Use `is_deleted` flag.
- **Money Rule:** All quantity, threshold, and valuation fields must use 4-decimal precision (`Numeric(20,4)`).
- **SKU Uniqueness:** The `sku` must be unique within the `subsidiary_id` scope.
- **Valuation Integrity:** Changing the `valuation_method` is restricted if the item has existing stock transactions; it requires a manual revaluation process.

## 11. Security, Audit, and RBAC
- **Roles:** 
  - **Inventory Manager:** Full CRUD access and reorder point configuration.
  - **Inventory Clerk:** Read access and ability to update stock counts, but cannot change item master configurations.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, users can view item specifications and stock levels but cannot create or modify items.
- **Audit Trail:** Every change to `valuation_method`, `sku`, or `reorder_point` must be logged with `previous_state` and `new_state`.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** Items that have fallen below their "Reorder Point" surface on the **Inventory Dashboard** "Low Stock" pipeline to trigger purchasing actions.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** At the top of the Item Drawer. Stages: `Draft` → `Active` (or `Inactive`).
  - **Micro View (`Timeline`):** In the "Activity" tab. Logs: "SKU updated from OLD-1 to NEW-1", "Reorder point changed by Manager Mark".
- **Navigation:** Main Sidebar > Inventory > Item Master.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `core.items` and `inventory.item_details` inheriting from `BESBase`.
- **Phase 2: Core Logic & APIs:** Implement the Item Service handling transactions across core and inventory schemas.
- **Phase 3: Frontend Infrastructure:** Register the Item Master module in the `@bes/inventory` library.
- **Phase 4: UI Development:** Build the searchable Item List and the multi-tab Drawer.
- **Phase 5: Event Integration:** Wire up stock update listeners to show real-time quantity availability.

## 14. Verification & QA Strategy
- **Scoping Check:** Verify that items created in `Subsidiary A` are not accessible to `Subsidiary B`.
- **Precision Check:** Test reorder points with 4-decimal values (e.g., `10.5025`) and verify they are stored without truncation.
- **Functional Scenarios:**
  1. Create an item and verify it correctly appears in the search index.
  2. Update the reorder point and verify the "Low Stock" pipeline logic triggers correctly.
  3. Attempt to deactivate an item and verify it is hidden from Sales Order selection.
- **Integration Test:** Verify that changing the valuation method in the Item Master updates the downstream cost calculation logic.
