# Leave Management

## 1. Module
HR

## 2. Name
Leave Management

## 3. Description
The Leave Management feature provides a comprehensive system for tracking employee time off, encompassing vacation, sick leave, unpaid leave, and statutory holidays. It automates the request, approval, and balance-tracking processes. By integrating tightly with the Employee Master and Payroll modules, it ensures that available leave balances are accurately maintained, manager approvals are systematically routed, and any unpaid leave directly impacts salary computations in the corresponding pay cycle.

## 4. Depends on
- **Employee Master (HR):** To identify the requesting employee, determine their employment type, and route the request to their assigned Reporting Manager.
- **Core Integration:** Core provides the unified `BESBase` foundation. Leave records are maintained in the logically isolated `hr` schema, ensuring strict data privacy and isolation via `subsidiary_id`.
- **Other Dependencies:**
  - `@bes/shared-ui` for standardized Drawer-based forms, calendar views, and Process Transparency components.
  - `Payroll Processing (HR)` (Optional downstream dependency): To compute deductions for "Loss of Pay" (LOP) leaves.

## 5. Feature name, details - the UI as information
**Feature Name:** Leave Requests & Balance Tracking
**Details & UI Information:**
- **Dashboard / Calendar View:** A unified team calendar allowing managers to view overlapping leaves, alongside a personal "My Balances" widget for employees displaying accrued, consumed, and available days for each leave type.
- **Resource List (Leave Requests):** A data grid of leave applications. Columns: `Employee`, `Leave Type`, `From Date`, `To Date`, `Duration (Days)`, and `Status` (Draft, Pending Approval, Approved, Rejected, Cancelled).
- **Creation/Edit Drawer (Leave Application):** A Right Panel (Drawer) for submitting and reviewing requests:
  - **Application Tab:** `Leave Type` (dropdown), `Start Date`, `End Date`, `Duration` (auto-calculated, supports half-days), `Reason` (text area), and an `Attachment` uploader (e.g., for medical certificates).
  - **Balance Snapshot Tab:** A read-only view of the employee's current balance for the selected leave type at the time of the request.
  - **History/Activity Tab:** Contains the vertical `Timeline` for audit traceability regarding the approval workflow.
- **Empty States:** "You have no upcoming leave requests. Need a break? Apply for leave."
- **Loading States:** Shimmer effects for balance calculation and skeleton loaders for the calendar view.

**User Journey & UX Flow:**
1. User navigates to **HR > Leave Management**.
2. User clicks **"Apply for Leave"** -> Right-panel drawer opens.
3. User selects **Leave Type** and **Dates** -> System auto-calculates duration (excluding weekends/holidays) and verifies against available balance.
4. User clicks **"Submit Request"** -> Status becomes `Pending Approval` and an event is emitted to notify the Reporting Manager.
5. Manager sees the request on their Home Dashboard **Pending Pipeline**.
6. Manager opens the drawer, reviews the team calendar for overlaps, and clicks **"Approve"**.
7. System updates the status to `Approved`, emits a final event, and immediately deducts the approved duration from the employee's `hr.leave_balances`.

## 6. YAML or sample data structure
### YAML Schema
```yaml
LeaveRequest:
  id: uuid
  subsidiary_id: uuid
  employee_id: uuid
  leave_type_id: uuid
  start_date: date
  end_date: date
  duration_days: decimal(20,4)
  reason: string
  status: enum [DRAFT, PENDING_APPROVAL, APPROVED, REJECTED, CANCELLED]
  manager_id: uuid (assigned approver)
  metadata_: jsonb

LeaveBalance:
  id: uuid
  subsidiary_id: uuid
  employee_id: uuid
  leave_type_id: uuid
  accrued_days: decimal(20,4)
  consumed_days: decimal(20,4)
  available_days: decimal(20,4)
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "leave-550e8400-e29b-41d4-a716-446655440000",
    "subsidiary_id": "sub-990e8400-e29b-41d4-a716-446655441111",
    "employee_id": "emp-jane-doe-uuid",
    "leave_type_id": "ltype-annual-uuid",
    "start_date": "2026-06-10",
    "end_date": "2026-06-12",
    "duration_days": "2.5000",
    "reason": "Family vacation",
    "status": "APPROVED",
    "manager_id": "emp-manager-uuid",
    "created_at": "2026-05-16T10:00:00Z"
  },
  "metadata": {
    "version": "1.0"
  },
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/hr/leaves`**: Fetch a paginated list of leave requests (supports filtering by status, employee, or department).
- **GET `/api/v1/hr/leaves/balances/{employee_id}`**: Fetch the current leave balances for a specific employee.
- **POST `/api/v1/hr/leaves`**: Submit a new leave application.
- **PUT `/api/v1/hr/leaves/{id}/approve`**: Manager action to approve the leave (triggers balance deduction).
- **PUT `/api/v1/hr/leaves/{id}/reject`**: Manager action to reject the leave.
- **PUT `/api/v1/hr/leaves/{id}/cancel`**: Employee or HR action to cancel an approved leave (restores balance).

## 8. Database Tables & Architecture
### Schema: HR Module
#### Table: `hr.leave_requests`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `employee_id`: UUID (FK to `hr.employees`, NOT NULL)
- `leave_type_id`: UUID (FK to `hr.leave_types`)
- `start_date`: Date
- `end_date`: Date
- `duration_days`: Numeric(20,4)
- `status`: String (Default: 'DRAFT')
- `manager_id`: UUID (FK to `hr.employees`)

#### Table: `hr.leave_balances`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `employee_id`: UUID (FK to `hr.employees`, NOT NULL)
- `leave_type_id`: UUID (FK to `hr.leave_types`)
- `accrued_days`: Numeric(20,4)
- `consumed_days`: Numeric(20,4)
- `available_days`: Numeric(20,4)

#### Table: `hr.leave_types`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `name`: String (e.g., 'Annual Leave', 'Sick Leave', 'Loss of Pay')
- `is_paid`: Boolean

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `hr.leave.submitted`: Triggers an alert to the Reporting Manager's dashboard.
  - `hr.leave.approved`: Triggers the balance deduction logic and updates the team calendar.
  - `hr.leave.cancelled`: Triggers balance restoration.
- **Listens To:**
  - `hr.payroll.cycle_started`: To lock approved leaves that fall within the current payroll cycle to prevent retroactive cancellations that would affect processed payroll.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion is strictly forbidden. Use `status = 'CANCELLED'` for revoked leaves.
- **Money Rule (Fractional Days):** All duration and balance fields MUST use 4-decimal precision (`Numeric(20,4)`) to accurately support fractional leaves (e.g., 0.5000 days for a half-day).
- **Balance Validation:** The system must block submission of a paid leave request if `duration_days` > `available_days`, unless the specific `leave_type` allows negative balances.
- **Overlap Prevention:** An employee cannot submit multiple leave requests for overlapping date ranges.
- **Weekend/Holiday Exclusion:** The duration calculation engine must automatically exclude defined subsidiary weekends and statutory holidays from the requested date range.

## 11. Security, Audit, and RBAC
- **Roles:**
  - `Employee (Self-Service)`: Can apply for leave, cancel their own pending/approved future leaves, and view their own balances.
  - `Manager`: Can view balances of direct reports, and approve/reject requests routed to them.
  - `HR Admin`: Full CRUD access, can override balances, force-approve/cancel requests on behalf of employees.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, the "Apply for Leave" button is hidden. The UI degrades to a read-only calendar and balance viewer.
- **Audit Trail:** Strict logging required for status transitions (Submission, Approval, Rejection, Cancellation), capturing the exact timestamp, the user performing the action, and any provided reasons.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** "Pending Leave Approvals" surface prominently on the Reporting Manager's **Home Dashboard** as actionable items.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** Anchored at the top of the Leave Application Drawer. Stages: `Draft` → `Pending Manager Approval` → `Approved` (or `Rejected`). Action buttons (Approve/Reject) reside here for managers.
  - **Micro View (`Timeline`):** In the "Activity" tab of the drawer. Logs the full lineage: "Leave requested by John", "Medical Certificate attached", "Approved by Manager Sarah".
- **Traceability:** Visualizing the impact: **`Leave Request`** → `Balance Deduction` → `Payroll LOP Calculation` (if applicable).
- **Navigation:** Main Sidebar > HR > Leave Management.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `leave_types`, `leave_balances`, and `leave_requests` extending `BESBase`.
- **Phase 2: Core Logic & APIs:** Implement the Leave Duration Calculation Service (excluding weekends/holidays) and the balance validation logic.
- **Phase 3: Frontend Infrastructure:** Register the Leave Management component in the `@bes/hr` library.
- **Phase 4: UI Development:** Build the Team Calendar, "My Balances" widget, and the Leave Application Drawer using `@bes/shared-ui`.
- **Phase 5: Event Integration:** Wire up Pub/Sub for routing approvals to managers and reflecting real-time balance deductions upon approval.

## 14. Verification & QA Strategy
- **Scoping Check:** Verify that a Manager in `Subsidiary A` cannot view or approve leave requests for employees belonging to `Subsidiary B`.
- **Precision Check:** Submit a request for two half-days and verify the balance deducts exactly `1.0000` days using `Numeric(20,4)` logic.
- **Functional Scenarios:**
  1. Submit a leave request that exceeds the available balance and verify the validation block.
  2. Test the duration engine by requesting a week off that includes a statutory holiday; verify the calculated duration is 4 days instead of 5.
  3. Approve a request, verify balance deduction, then Cancel the request and verify balance restoration.
- **Integration Test:** Ensure that an approved "Loss of Pay" leave triggers the required data payload for the downstream Payroll processing engine.
