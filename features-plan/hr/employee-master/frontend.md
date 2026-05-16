# Employee Master & Onboarding

## 1. Module
HR

## 2. Name
Employee Master & Onboarding

## 3. Description
The Employee Master & Onboarding feature is the central system of record for all human capital within the Business Execution System (BES). It manages the entire lifecycle of an employee, transitioning them from a new hire undergoing onboarding to an active team member, and eventually managing offboarding. By separating the core user identity (used for system login) from HR-specific employment details, this module ensures strict data privacy for sensitive information (like salaries and compliance documents) while enabling seamless integration with Payroll, Attendance, and Performance Management.

## 4. Depends on
- **Core (Master Data & IAM):** Relies on the `core.users` table for system authentication, role-based access control (RBAC), and basic identity (Name, Email).
- **Core Integration:** Core provides the unified `BESBase` foundation. While system-wide identity resides in `core.users`, sensitive employment data (compensations, tax details, employment history) resides securely in the logically isolated `hr` schema (`hr.employees`).
- **Other Dependencies:**
  - `@bes/shared-ui` for multi-step onboarding wizards and Process Transparency components.
  - `Document Management / Storage Service` for handling confidential HR documents (IDs, contracts, tax forms).
  - `Organization Structure (HR)` for assigning employees to specific departments, branches, and reporting managers.

## 5. Feature name, details - the UI as information
**Feature Name:** Employee Directory & Profile Management
**Details & UI Information:**
- **Resource List (Employee Directory):** A searchable table of all staff. Columns: `Employee ID`, `Name`, `Designation`, `Department`, `Manager`, `Status` (Onboarding, Active, Leave, Terminated).
- **Creation/Edit Drawer (Onboarding Wizard & Profile):** A multi-tab Right Panel (Drawer) for complete employee management:
  - **Identity Tab:** `First/Last Name`, `Personal Email`, `Date of Birth`, `Gender`, and `Contact Numbers`.
  - **Employment Tab:** `Employee ID` (auto-sequence), `Date of Joining`, `Designation`, `Department`, `Reporting Manager` (lookup), and `Employment Type` (Full-time, Contract).
  - **Compensation Tab:** `Base Salary` (4-decimal precision), `Pay Frequency`, and `Bank Account Details` (highly restricted access).
  - **Documents Tab:** A secure vault for uploading `Offer Letter`, `ID Proof`, and `Tax Forms`.
  - **History/Activity Tab:** Contains the vertical `Timeline` for audit traceability regarding promotions, role changes, or salary adjustments.
- **Empty States:** "No employees found. Start by onboarding your first team member."
- **Loading States:** Shimmer effects for profile data and skeleton loaders for the document vault.

**User Journey & UX Flow:**
1. User navigates to **HR > Employees**.
2. User clicks **"Initiate Onboarding"** -> Right-panel wizard opens.
3. User enters basic **Identity** details -> System creates a draft profile.
4. User uploads required **Documents** and assigns a **Department/Manager**.
5. User submits for approval -> Status updates to `Pending Activation`.
6. HR Manager approves -> System generates a `core.users` identity, links it to the `hr.employees` record, sends a welcome email, and status becomes `Active`.
7. Later, any salary adjustments or promotions are tracked strictly within the employee's **Timeline**.

## 6. YAML or sample data structure
### YAML Schema
```yaml
Employee:
  id: uuid
  subsidiary_id: uuid
  user_id: uuid (optional, links to core identity for system access)
  employee_code: string
  first_name: string
  last_name: string
  personal_email: string
  date_of_joining: date
  status: enum [ONBOARDING, ACTIVE, ON_LEAVE, TERMINATED]
  employment_details:
    designation: string
    department_id: uuid
    manager_id: uuid (FK to core.users or hr.employees)
    employment_type: enum [FULL_TIME, PART_TIME, CONTRACT]
  compensation:
    base_salary: decimal(20,4)
    currency_id: uuid
    pay_frequency: enum [MONTHLY, BI_WEEKLY]
  metadata_: jsonb
```

### Sample JSON Payload
```json
{
  "status": "success",
  "data": {
    "id": "emp-550e8400-e29b-41d4-a716-446655440000",
    "subsidiary_id": "sub-990e8400-e29b-41d4-a716-446655441111",
    "user_id": "user-system-uuid-123",
    "employee_code": "EMP-2026-042",
    "first_name": "Jane",
    "last_name": "Doe",
    "status": "ACTIVE",
    "date_of_joining": "2026-01-15",
    "employment_details": {
      "designation": "Senior Software Engineer",
      "department_id": "dept-engineering-uuid",
      "manager_id": "emp-manager-uuid-789",
      "employment_type": "FULL_TIME"
    },
    "compensation": {
      "base_salary": "120000.0000",
      "pay_frequency": "MONTHLY"
    },
    "created_at": "2026-05-16T15:00:00Z"
  },
  "metadata": {
    "version": "1.0"
  },
  "error": null
}
```

## 7. Required APIs
- **GET `/api/v1/hr/employees`**: Fetch paginated list of employees with departmental filtering.
- **GET `/api/v1/hr/employees/{id}`**: Fetch full, sensitive employee profile (requires strict RBAC checks).
- **POST `/api/v1/hr/employees/onboard`**: Initiate a new employee onboarding process.
- **POST `/api/v1/hr/employees/{id}/activate`**: Finalize onboarding; creates system credentials in `core.users`.
- **PUT `/api/v1/hr/employees/{id}`**: Update employment or compensation details.
- **PATCH `/api/v1/hr/employees/{id}/terminate`**: Initiate the offboarding process.

## 8. Database Tables & Architecture
### Schema: Core Module (Identity Hub)
#### Table: `core.users`
- **Inherits**: `BESBase`
- Acts as the global identity for system login. Does NOT contain sensitive HR data like salary or SSN.

### Schema: HR Module (Isolated Spoke)
#### Table: `hr.employees`
- **Inherits**: `BESBase`
- `subsidiary_id`: UUID (NOT NULL)
- `user_id`: UUID (FK to `core.users`, UNIQUE, NULLABLE during early onboarding)
- `employee_code`: String (UNIQUE, NOT NULL)
- `first_name`: String
- `last_name`: String
- `date_of_joining`: Date
- `status`: String (Default: 'ONBOARDING')
- `department_id`: UUID
- `manager_id`: UUID (FK to `hr.employees` or `core.users`)
- `base_salary`: Numeric(20,4) (Encrypted at rest or strictly isolated)
- `employment_type`: String

## 9. Events & Real-Time Updates (Pub/Sub)
- **Emits:**
  - `hr.employee.onboarded`: Triggers IT provisioning tasks (laptop, email creation).
  - `hr.employee.activated`: Signals the Payroll module to include this employee in the next cycle.
  - `hr.employee.terminated`: Immediately triggers `core.user.deactivated` to revoke system access.
- **Listens To:**
  - `hr.leave.approved`: To potentially update real-time "Status" in the directory.

## 10. Business Rules & Validations
- **Soft Deletes:** Physical deletion of employee records is strictly forbidden due to historical payroll compliance. Use `status = 'TERMINATED'` and `is_deleted = true` only for data entry errors before activation.
- **Money Rule:** All compensation and salary figures MUST use 4-decimal precision (`Numeric(20,4)`).
- **Data Privacy & Isolation:** Only users with specific `HR_ADMIN` or `PAYROLL_ADMIN` roles can query the `base_salary` fields. Standard `HR_VIEWER` roles see a masked payload.
- **Manager Validation:** An employee cannot be assigned a reporting manager who is currently in a `TERMINATED` or `ON_LEAVE` status.

## 11. Security, Audit, and RBAC
- **Roles:**
  - `HR Manager`: Full CRUD access, can activate and terminate employees, can view compensation.
  - `HR Assistant`: Can initiate onboarding and upload documents, but cannot view or set compensation.
  - `Employee (Self-Service)`: Can view/edit their own Personal Identity Tab, but cannot edit Employment or Compensation tabs.
- **Read-Only Licensing:** If `READONLY_EXTENSIONS` is active, the "Initiate Onboarding" button is hidden. Forms are rendered disabled.
- **Audit Trail:** Extremely critical for HR compliance. Every change to `base_salary`, `designation`, or `status` MUST be logged in the central audit system, capturing the exact timestamp, `previous_state`, `new_state`, and the HR user responsible.

## 12. Process Transparency & Workflow Pipeline
- **Pending Pipeline:** "Pending Onboardings" and "Upcoming Anniversaries/Probation Ends" surface on the **HR Dashboard**.
- **Process UI Integration:**
  - **Macro View (`ProcessPipeline`):** At the top of the Employee Drawer during the hire phase. Stages: `Draft` → `Document Collection` → `Pending Activation` → `Active`.
  - **Micro View (`Timeline`):** In the "History" tab. Logs: "Offer Letter uploaded by Jane", "Salary revised from 100k to 120k by HR Admin", "Promoted to Senior Engineer".
- **Traceability:** Visualizing the employee lifecycle: `Candidate (CRM/ATS)` → **`Onboarding (HR)`** → `Active Employee` → `Payroll Processing`.
- **Navigation:** Main Sidebar > HR > Employee Directory.

## 13. Technical Implementation Roadmap
- **Phase 1: Backend Foundation:** Migrations for `hr.employees` extending `BESBase` and establishing the FK link to `core.users`.
- **Phase 2: Core Logic & APIs:** Implement the Onboarding Service, handling the atomic creation of the HR record and the subsequent generation of the Core User identity.
- **Phase 3: Frontend Infrastructure:** Register the Employee Master component in the `@bes/hr` library.
- **Phase 4: UI Development:** Build the Employee Directory grid and the multi-step Onboarding Drawer with document upload capabilities.
- **Phase 5: Event Integration:** Wire up Pub/Sub for IT provisioning and system access revocation upon termination.

## 14. Verification & QA Strategy
- **Scoping Check:** Verify that an HR Manager in `Subsidiary A` cannot view or search for employees in `Subsidiary B`.
- **Data Privacy Check:** Ensure that API responses for `HR_ASSISTANT` roles strip out the `compensation` object entirely.
- **Functional Scenarios:**
  1. Complete the onboarding wizard and verify that activating the employee correctly generates a linked `core.users` record.
  2. Attempt to terminate an employee and verify that the `hr.employee.terminated` event successfully locks their system login.
  3. Verify the Audit Timeline accurately captures a change in `base_salary`.
- **Integration Test:** Verify that the "Money Rule" applies correctly to payroll calculations by testing fractional salary increments.
