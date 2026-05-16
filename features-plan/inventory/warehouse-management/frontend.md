# Warehouse & Location Management

## 1. Module
Inventory

## 2. Name
Warehouse & Location Management

## 3. Description
The Warehouse & Location Management feature provides the spatial framework for inventory tracking within the Business Execution System (BES). It enables the organization to define a hierarchical structure of storage points, ranging from large-scale regional warehouses to granular storage bins. This module ensures that every item has a precisely defined home, facilitating efficient picking, put-away, and stock accuracy through real-time visibility of sub-warehouse locations (Zones, Aisles, Racks, and Bins).

## 4. Depends on
- **Core (Subsidiary Context):** To map warehouses to specific organizational entities and maintain data isolation.
- **Inventory Item Master:** To establish default storage locations for specific items.
- **Core Integration:** Core provides the unified `BESBase` foundation, allowing the `inventory` module to manage complex physical hierarchies while ensuring that every warehouse and location record is scoped correctly via `subsidiary_id`.
- **Other Dependencies:** 
  - `@bes/shared-ui` for hierarchical tree views and standardized resource lists.

## 5. Feature name, details - the UI as information
**Feature Name:** Warehouse Hierarchy & Bin Management
**Details & UI Information:** 
- **Warehouse List:** A table view showing all primary storage facilities. Columns: `Warehouse Name`, `Code`, `Type` (Physical/Virtual/Transit), `Manager`, and `Status`.
- **Hierarchical Tree View:** A side-panel or drill-down view that visualizes the warehouse structure:
  - **Warehouse** (e.g., Main Distribution Center)
    - **Zone** (e.g., Cold Storage, High-Value Area)
      - **Aisle / Rack / Bin** (e.g., Aisle 01 > Rack B > Bin 04)
- **Location Detail Drawer:** A Right Panel (Drawer) for configuring specific storage points:
  - **Configuration Tab:** `Location Code`, `Location Name`, `Capacity` (Volume/Weight with 4-decimal precision), and `Is Picking Location` (Boolean).
  - **Stock Tab:** A real-time list of all items currently stored in this specific bin/location.
  - **History Tab:** The vertical `Timeline` showing structural changes or stock movements associated with this location.
- **Empty States:** "No warehouses defined. Start by setting up your primary storage facility."
- **Loading States:** Shimmer effects for the tree hierarchy and skeleton loaders for the stock lists.

**User Journey & UX Flow:**
1. User navigates to **Inventory > Warehouse Management**.
2. User clicks **"Create Warehouse"** -> Defines the root facility (e.g., "Chicago WH").
3. User uses the **Tree View** to add **Zones** and **Bins** within that warehouse.
4. User configures a specific **Bin** as a "Picking Location" and sets its volumetric capacity.
5. System validates that the **Location Code** is unique within the warehouse hierarchy.
6. User can click on any bin in the tree to immediately see its current **Stock Level** and recent activity.

## 6. YAML or sample data structure
### YAML Schema
```yaml
Warehouse:
  id: uuid
  subsidiary_id: uuid
  name: string
  code: string
  type: enum [PHYSICAL, VIRTUAL, TRANSIT]
  is_active: boolean
  metadata_: jsonb

Location:
  id: uuid
  warehouse_id: uuid
  parent_id: uuid (optional)
  name: string
  code: string
  type: enum [ZONE, AISLE, RACK, BIN]
  capacity_m3: decimal(20,4)
  is_picking: boolean
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "wh-550e8400",
    "subsidiary_id": "sub-990e8400",
    "name": "Central Warehouse",
    "code": "CWH-01",
    "type": "PHYSICAL",
    "hierarchy": [
      {
        "id": "loc-zone-1",
        "name": "Zone A (Bulk Storage)",
        "type": "ZONE",
        "children": [
          {
            "id": "loc-bin-101",
            "name": "Bin A-01-01",
            "type": "BIN",
            "capacity_m3": "12.5000",
            "is_picking": true
          }
        ]
      }
    ]
  },
  "metadata": {},
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/inventory/warehouses`**: Fetch list of warehouses.
- **GET `/api/v1/inventory/warehouses/{id}/tree`**: Fetch the full hierarchical location structure for a warehouse.
- **POST `/api/v1/inventory/warehouses`**: Create a new warehouse.
- **POST `/api/v1/inventory/locations`**: Add a new location (Zone/Bin) to the hierarchy.
- **GET `/api/v1/inventory/locations/{id}/stock`**: Get current stock breakdown for a specific location.

## 8. Database Tables & Architecture
### Schema: Inventory Module
#### Table: `inventory.warehouses`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `name`: String (NOT NULL)
- `code`: String (UNIQUE, NOT NULL)
- `type`: String (Default: 'PHYSICAL')
- `is_active`: Boolean (Default: true)

#### Table: `inventory.locations`
- **Inherits**: `BESBase`
- `warehouse_id`: UUID (NOT NULL, FK to `inventory.warehouses`)
- `parent_id`: UUID (FK to `inventory.locations.id`) - Self-reference for hierarchy.
- `subsidiary_id`: UUID (NOT NULL)
- `name`: String (NOT NULL)
- `code`: String (NOT NULL)
- `type`: String (e.g., 'ZONE', 'BIN')
- `capacity_m3`: Numeric(20,4)
- `is_picking`: Boolean (Default: false)

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:** 
  - `inventory.warehouse.created`: Published when a new facility is added.
  - `inventory.location.capacity_alert`: Triggered if a bin's volumetric capacity is exceeded during put-away.
- **Listens To:** 
  - `inventory.stock.moved`: Updates the "Current Stock" view for both source and destination locations in real-time.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion forbidden; use `is_deleted`.
- **Money Rule:** Volume and weight capacity fields must use 4-decimal precision (`Numeric(20,4)`).
- **Hierarchy Constraints:** A warehouse cannot be deleted if it contains active child locations or stock.
- **Code Uniqueness:** Location codes must be unique within their parent warehouse.

## 11. Security, Audit, and RBAC
- **Roles:** 
  - **Warehouse Manager:** Can create/edit hierarchies and define capacities.
  - **Warehouse Associate:** Can view the tree and stock levels but cannot modify the structure.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, the tree view remains interactive for navigation, but the "Add Zone/Bin" buttons are hidden.
- **Audit Trail:** Log all structural changes (e.g., "Bin A-01 moved from Zone A to Zone B") and capacity adjustments.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** Warehouses with "Critical Capacity" (e.g., >90% bins full) surface on the **Inventory Dashboard** to signal a need for stock rebalancing.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** Displayed at the top of the Warehouse Drawer. Stages: `Inactive` → `Setup` → `Operational`.
  - **Micro View (`Timeline`):** In the "Activity" tab of the Location drawer. Logs: "Bin created by Alice", "Capacity increased by 5.0000 m3 by Bob".
- **Navigation:** Main Sidebar > Inventory > Warehouse Management.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `warehouses` and `locations` with self-referencing foreign keys.
- **Phase 2: Core Logic & APIs:** Implement recursive tree-fetching logic and capacity validation services.
- **Phase 3: Frontend Infrastructure:** Register the Warehouse module in the Inventory library.
- **Phase 4: UI Development:** Build the Hierarchical Tree View and the Location Detail Drawer.
- **Phase 5: Event Integration:** Implement real-time stock-at-location updates using SSE.

## 14. Verification & QA Strategy
- **Scoping Check:** Verify that users in `Subsidiary A` cannot see the warehouse tree of `Subsidiary B`.
- **Precision Check:** Verify that a capacity of `0.0015 m3` is correctly handled without rounding to zero.
- **Functional Scenarios:**
  1. Build a 4-level hierarchy (WH -> Zone -> Rack -> Bin) and verify the tree renders correctly.
  2. Attempt to add two bins with the same code in the same warehouse and verify the uniqueness error.
  3. Verify that stock movements correctly update the counts at the specific bin level.
- **Integration Test:** Verify that the "Default Warehouse" set in the Item Master correctly resolves to an active warehouse in the hierarchy.
