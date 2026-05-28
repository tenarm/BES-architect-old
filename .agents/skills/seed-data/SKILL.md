---
name: seed-data
description: "Create idempotent seed/demo data scripts for BES modules to populate realistic test data."
---

# Skill: Seed Data Script Creation

This skill creates seed scripts that populate realistic demo/initial data for BES modules. Every module should have seed data so developers and testers can work with meaningful records immediately.

---

## Prerequisites
- The module's backend extension is built with models, schemas, and services
- The module has a `3-backend-plan.md` or `proposed-plan.md` documenting entity schemas

---

## Steps

### Step 1: Research the Module Schema

1. Read the module's `models.py` to understand table structures and field types.
2. Read the module's `3-backend-plan.md` (or `proposed-plan.md`) for entity relationships.
3. Check `features-plan/common-dependants.md` for cross-module dependencies (e.g., does this module need Settings data seeded first?).

### Step 2: Create the Seed Script

Create `bes-backend/scripts/seed_<module>.py`:

```python
"""
Seed script for <MODULE> module.
Populates realistic demo data for development and testing.

Usage: pdm run seed-<module>
"""
import asyncio
import sys
from pathlib import Path
from decimal import Decimal
from uuid import uuid4

# Add paths for imports
root = Path(__file__).resolve().parents[1]
sys.path.insert(0, str(root / "core"))
sys.path.insert(0, str(root / "extensions" / "<module_name>"))

from core.database import get_engine, async_session_factory
from <module_name>.models import <Entity>


async def seed():
    """Main seed function — idempotent (safe to run multiple times)."""
    async with async_session_factory() as session:
        # Check if data already exists
        existing = await session.execute(
            select(<Entity>).where(<Entity>.is_deleted == False).limit(1)
        )
        if existing.scalar_one_or_none():
            print("✓ <Module> data already seeded. Skipping.")
            return

        print("Seeding <Module> data...")

        records = [
            # Realistic demo data — see domain templates below
        ]

        for record in records:
            session.add(record)

        await session.commit()
        print(f"✓ Seeded {len(records)} <entity> records.")


if __name__ == "__main__":
    asyncio.run(seed())
```

### Step 3: Use Domain-Specific Data Templates

Use realistic, industry-standard data — NOT Lorem ipsum or "Test 1, Test 2":

#### Settings / General
```python
# Currencies (ISO 4217)
currencies = [
    {"code": "USD", "name": "US Dollar", "symbol": "$", "decimal_places": 2},
    {"code": "EUR", "name": "Euro", "symbol": "€", "decimal_places": 2},
    {"code": "GBP", "name": "British Pound", "symbol": "£", "decimal_places": 2},
    {"code": "INR", "name": "Indian Rupee", "symbol": "₹", "decimal_places": 2},
    {"code": "AED", "name": "UAE Dirham", "symbol": "د.إ", "decimal_places": 2},
]

# UOM Groups
uom_groups = [
    {"group": "Quantity", "units": [("Each", "EA", 1), ("Dozen", "DZ", 12), ("Hundred", "C", 100)]},
    {"group": "Weight", "units": [("Kilogram", "KG", 1), ("Gram", "G", 0.001), ("Tonne", "MT", 1000)]},
    {"group": "Length", "units": [("Meter", "M", 1), ("Centimeter", "CM", 0.01), ("Foot", "FT", 0.3048)]},
    {"group": "Volume", "units": [("Liter", "L", 1), ("Milliliter", "ML", 0.001), ("Gallon", "GAL", 3.785)]},
]
```

#### Sales / Customers
```python
customers = [
    {"name": "Apex Technologies Ltd.", "code": "CUST-001", "credit_limit": Decimal("50000.0000"),
     "payment_terms": "Net 30", "email": "accounts@apextech.com", "phone": "+1-555-0101"},
    {"name": "GreenLeaf Organics", "code": "CUST-002", "credit_limit": Decimal("25000.0000"),
     "payment_terms": "Net 15", "email": "billing@greenleaf.co", "phone": "+1-555-0202"},
    {"name": "NexGen Manufacturing", "code": "CUST-003", "credit_limit": Decimal("100000.0000"),
     "payment_terms": "Net 45", "email": "ap@nexgenm.com", "phone": "+1-555-0303"},
]
```

#### Inventory / Items
```python
items = [
    {"name": "Steel Rod 10mm", "code": "INV-STL-001", "category": "Raw Materials",
     "uom": "KG", "cost_price": Decimal("2.5000"), "sale_price": Decimal("3.7500")},
    {"name": "Copper Wire 2.5mm", "code": "INV-CPR-001", "category": "Raw Materials",
     "uom": "M", "cost_price": Decimal("1.2000"), "sale_price": Decimal("1.8000")},
    {"name": "Hydraulic Pump HP-200", "code": "INV-PMP-001", "category": "Finished Goods",
     "uom": "EA", "cost_price": Decimal("450.0000"), "sale_price": Decimal("675.0000")},
]
```

#### Finance / Chart of Accounts (IFRS-aligned)
```python
coa_accounts = [
    # Assets (1xxx)
    {"code": "1000", "name": "Cash and Cash Equivalents", "type": "Asset", "group": "Current Assets"},
    {"code": "1100", "name": "Accounts Receivable", "type": "Asset", "group": "Current Assets"},
    {"code": "1200", "name": "Inventory", "type": "Asset", "group": "Current Assets"},
    {"code": "1500", "name": "Property, Plant & Equipment", "type": "Asset", "group": "Non-Current Assets"},
    # Liabilities (2xxx)
    {"code": "2000", "name": "Accounts Payable", "type": "Liability", "group": "Current Liabilities"},
    {"code": "2100", "name": "Accrued Expenses", "type": "Liability", "group": "Current Liabilities"},
    {"code": "2500", "name": "Long-Term Debt", "type": "Liability", "group": "Non-Current Liabilities"},
    # Equity (3xxx)
    {"code": "3000", "name": "Share Capital", "type": "Equity", "group": "Owner's Equity"},
    {"code": "3100", "name": "Retained Earnings", "type": "Equity", "group": "Owner's Equity"},
    # Revenue (4xxx)
    {"code": "4000", "name": "Sales Revenue", "type": "Revenue", "group": "Operating Revenue"},
    {"code": "4100", "name": "Service Revenue", "type": "Revenue", "group": "Operating Revenue"},
    # Expenses (5xxx-6xxx)
    {"code": "5000", "name": "Cost of Goods Sold", "type": "Expense", "group": "Direct Costs"},
    {"code": "6000", "name": "Salaries & Wages", "type": "Expense", "group": "Operating Expenses"},
    {"code": "6100", "name": "Rent Expense", "type": "Expense", "group": "Operating Expenses"},
    {"code": "6200", "name": "Utilities Expense", "type": "Expense", "group": "Operating Expenses"},
]
```

#### Supply Chain / Suppliers
```python
suppliers = [
    {"name": "Global Steel Corp", "code": "SUP-001", "category": "Raw Materials",
     "payment_terms": "Net 60", "tax_id": "GST-98765", "email": "orders@globalsteel.com"},
    {"name": "Precision Parts Inc", "code": "SUP-002", "category": "Components",
     "payment_terms": "Net 30", "tax_id": "GST-45678", "email": "sales@precisionparts.co"},
]
```

### Step 4: Key Rules for Seed Data

- **Idempotent**: Always check if data exists before inserting. Use upsert or skip patterns.
- **Multi-Tenancy**: Assign correct `subsidiary_id` and `created_by` to all records.
- **Money Rule**: Use `Decimal("0.0000")` for ALL financial amounts — NEVER `float` (Rule 1 §3).
- **Soft Delete**: Set `is_deleted = False` explicitly on all seeded records.
- **Referential Integrity**: Seed dependencies first (e.g., currencies before customers, CoA before journals).

### Step 5: Register as PDM Script

Add to `bes-backend/pyproject.toml` under `[tool.pdm.scripts]`:
```toml
[tool.pdm.scripts]
seed-<module> = "python scripts/seed_<module>.py"
```

### Step 6: Run & Verify

```bash
cd bes-backend
pdm run seed-<module>
```

Verify via API:
```bash
curl http://localhost:8000/api/v1/<module>/<entity> \
  -H "Authorization: Bearer <token>"
```
