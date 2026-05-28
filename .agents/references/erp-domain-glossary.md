# ERP Domain Glossary — BES Reference

A comprehensive glossary of standard ERP terms for AI agents to reference when making domain decisions, per Rule 4 §11 (SME-less Validation). When in doubt, default to these industry-standard definitions.

---

## General ERP

| Term | Definition |
|:---|:---|
| **ERP** | Enterprise Resource Planning — integrated software managing core business processes (finance, HR, supply chain, manufacturing, sales). |
| **MDM** | Master Data Management — strategy for maintaining a single, consistent version of key business entities (customers, items, suppliers). |
| **SoR** | System of Record — the authoritative source for a specific data entity. Only one module should own each master record. |
| **Multi-tenancy** | Architecture where a single software instance serves multiple independent organizations (tenants) with data isolation. |
| **Subsidiary** | A legally distinct business entity within a corporate group, each maintaining its own books and compliance requirements. |
| **CRUD** | Create, Read, Update, Delete — the four basic database operations forming the foundation of most application features. |
| **Workflow** | A defined sequence of tasks/approvals that a business document must pass through before completion. |
| **Audit Trail** | Immutable record of all changes to a document, including who changed what and when. |

---

## Finance & Accounting

| Term | Definition |
|:---|:---|
| **GAAP** | Generally Accepted Accounting Principles — US accounting standards framework. |
| **IFRS** | International Financial Reporting Standards — global accounting standards used in 140+ countries. |
| **CoA** | Chart of Accounts — structured list of all ledger accounts used to record financial transactions (Assets, Liabilities, Equity, Revenue, Expenses). |
| **GL** | General Ledger — the master accounting record containing all financial transactions posted as journal entries. |
| **AP** | Accounts Payable — money owed to suppliers/vendors for goods or services received. |
| **AR** | Accounts Receivable — money owed by customers for goods or services delivered. |
| **Journal Entry** | A double-entry accounting record with equal debits and credits posted to the GL. |
| **Trial Balance** | Report listing all GL account balances to verify debits equal credits. |
| **Balance Sheet** | Financial statement showing assets, liabilities, and equity at a point in time. |
| **P&L** | Profit & Loss (Income Statement) — financial statement showing revenue, expenses, and net income over a period. |
| **Fiscal Year** | The 12-month accounting period used for financial reporting (may differ from calendar year). |
| **Posting Period** | A subdivision of the fiscal year (usually monthly) that can be opened or closed for transaction posting. |
| **Double-Entry** | Accounting principle where every transaction has equal debit and credit entries, maintaining the accounting equation (A = L + E). |
| **Reconciliation** | Process of verifying that two sets of records (e.g., bank statement vs. GL) match. |
| **Cost Center** | An organizational unit that incurs costs but does not directly generate revenue (e.g., IT department). |
| **Profit Center** | An organizational unit responsible for both revenue and costs (e.g., a product line). |
| **Accrual** | Recording revenue/expense when earned/incurred, regardless of when cash is exchanged. |
| **Depreciation** | Systematic allocation of an asset's cost over its useful life. |

---

## Sales & CRM

| Term | Definition |
|:---|:---|
| **Lead** | A potential customer who has shown interest but has not yet been qualified. |
| **Opportunity** | A qualified lead with a potential deal value and estimated close date. |
| **Quotation** | A formal price offer to a customer, specifying items, quantities, prices, and validity period. |
| **Sales Order** | A confirmed customer order authorizing delivery and invoicing. The primary sales transaction document. |
| **Delivery Note** | Document accompanying shipped goods, confirming items and quantities dispatched. |
| **Sales Invoice** | A legally binding document requesting payment from the customer for delivered goods/services. |
| **Credit Note** | A document reducing the amount owed by a customer (for returns, discounts, or billing errors). |
| **Payment Terms** | Agreed schedule for when payment is due (e.g., Net 30 = payment due within 30 days of invoice). |
| **Credit Limit** | Maximum outstanding receivable amount allowed for a customer before orders are blocked. |
| **Aging Report** | Analysis of outstanding receivables/payables grouped by age (0-30, 31-60, 61-90, 90+ days). |

---

## Procurement & Supply Chain

| Term | Definition |
|:---|:---|
| **Purchase Requisition** | Internal request to procure goods/services, typically requiring approval before becoming a Purchase Order. |
| **RFQ** | Request for Quotation — formal request sent to multiple suppliers to compare prices and terms. |
| **Purchase Order** | A confirmed order to a supplier authorizing delivery of goods/services at agreed prices. |
| **GRN** | Goods Receipt Note — document recording the physical receipt of goods from a supplier against a Purchase Order. |
| **Three-Way Match** | Verification that PO, GRN, and Supplier Invoice all agree on quantities and amounts before payment approval. |
| **Supplier Evaluation** | Periodic assessment of supplier performance on quality, delivery, pricing, and compliance metrics. |
| **AP Invoice** | Supplier's invoice received for goods/services, matched against PO and GRN for payment processing. |
| **Lead Time** | Time between placing an order with a supplier and receiving the goods. |

---

## Inventory & Warehouse

| Term | Definition |
|:---|:---|
| **Item Master** | Central record defining a product/material with attributes like name, UOM, category, pricing, and accounting codes. |
| **SKU** | Stock Keeping Unit — unique identifier for each distinct product variant. |
| **UOM** | Unit of Measure — the unit used to count/measure inventory (e.g., Each, Kilogram, Meter, Liter). |
| **BOM** | Bill of Materials — structured list of raw materials, components, and sub-assemblies needed to manufacture a finished product. |
| **Lot/Batch** | A group of items produced or received together, tracked as a unit for quality and traceability. |
| **Serial Number** | Unique identifier assigned to each individual unit of a product for precise tracking. |
| **Bin Location** | A specific storage position within a warehouse (aisle, rack, shelf, bin). |
| **Cycle Count** | Periodic physical counting of a subset of inventory items to verify system accuracy without a full stock take. |
| **Reorder Point** | Minimum stock level that triggers a purchase requisition or replenishment order. |
| **Safety Stock** | Extra inventory held as a buffer against demand variability and supply delays. |
| **FIFO** | First In, First Out — costing method where oldest inventory is sold/consumed first. |
| **LIFO** | Last In, First Out — costing method where newest inventory is sold/consumed first (not allowed under IFRS). |
| **Weighted Average** | Costing method calculating average unit cost across all available stock. |
| **Stock Valuation** | Calculating the monetary value of on-hand inventory using a defined costing method. |

---

## Manufacturing

| Term | Definition |
|:---|:---|
| **Routing** | Sequence of manufacturing operations/steps required to produce a product, with time and resource estimates. |
| **Work Center** | A production resource (machine, workstation, or team) where manufacturing operations are performed. |
| **Production Order** | Authorization to manufacture a specific quantity of a product using a BOM and routing. |
| **Work Order** | A task-level instruction for a specific operation within a production order. |
| **MRP** | Material Requirements Planning — calculation engine that determines what materials to order, how much, and when, based on demand, BOM, and lead times. |
| **WIP** | Work in Progress — inventory that is partially completed in the manufacturing process. |
| **Scrap** | Material wasted or spoiled during manufacturing that cannot be used as finished product. |
| **Yield** | The percentage of input material that becomes usable output after manufacturing. |

---

## HR & Payroll

| Term | Definition |
|:---|:---|
| **Employee Master** | Central record for each employee containing personal info, position, department, compensation, and employment history. |
| **Org Chart** | Organizational structure showing reporting relationships and department hierarchy. |
| **Leave Management** | System for requesting, approving, and tracking employee time off (vacation, sick, personal). |
| **Attendance** | Recording employee work hours, typically via clock-in/clock-out or timesheet entry. |
| **Payroll** | Process of calculating and disbursing employee compensation including base pay, deductions, and taxes. |
| **ESS** | Employee Self-Service — portal where employees manage their own leave requests, timesheets, and personal information. |

---

## Quality Management

| Term | Definition |
|:---|:---|
| **QC Inspection** | Quality Control check performed on materials/products at defined stages (incoming, in-process, final). |
| **Non-Conformance** | A documented deviation from quality standards requiring investigation and corrective action. |
| **CAPA** | Corrective and Preventive Action — systematic process to identify root causes and implement measures to prevent recurrence. |

---

## Standard References

| Standard | Scope |
|:---|:---|
| **ISO 4217** | Currency codes (USD, EUR, INR, etc.) |
| **ISO 8601** | Date and time formats (YYYY-MM-DD, YYYY-MM-DDTHH:MM:SS) |
| **UNSPSC** | Product classification taxonomy for procurement |
| **GAAP / IFRS** | Financial reporting frameworks |
| **APICS / ASCM** | Supply chain and manufacturing standards |
| **ISO 9001** | Quality management system standards |
