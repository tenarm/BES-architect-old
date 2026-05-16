# Business Execution System (BES) Factory

Welcome to the **BES Factory**, a modular, multi-tenant monorepo designed for building and deploying scalable Business Execution Systems.

## 🚀 Overview

The BES Factory uses a **Kernel-and-Plugin** (Core + Extension) architecture. This ensures that the core engine remains stable and secure while allowing for rapid, client-specific customizations.

- **Kernel (Core)**: Handles Authentication, Database Pooling, RBAC, Event Bus, and Base Models.
- **Plugins (Extensions)**: Domain-specific modules like Finance, HR, CRM, and Sales.
- **Shell UI**: A unified React-based frontend that dynamically loads licensed modules.

---

## 📂 Repository Structure

| Directory | Description |
| :--- | :--- |
| **[`bes-backend/`](./bes-backend)** | Python monorepo (PDM) containing the Core, Extensions, and Client Instances. |
| **[`bes-frontend/`](./bes-frontend)** | React monorepo (Nx) containing the Shell UI and Module Libraries. |
| **[`bvk-items/`](./bvk-items)** | Supporting scripts, deployment commands, and internal documentation. |
| **[`.gemini/`](./.gemini)** | AI assistant rules and workflows for maintainability and consistency. |

---

## 🛠️ Quick Start

### Backend (Python)
```bash
cd bes-backend
pdm install
pdm run uvicorn instances.acme_corp.acme_corp.main:app --reload
```
*See [bes-backend/README.md](./bes-backend/README.md) for more details.*

### Frontend (React)
```bash
cd bes-frontend
npm install
npm run dev
```
*See [bes-frontend/README.md](./bes-frontend/README.md) for more details.*

---

## 📖 Key Documentation

- **[Architecture Blueprint](./ARCHITECTURE.md)**: High-level system design and guiding principles.
- **[Core as a Package](./bvk-items/docs/architecture/core_as_a_package.md)**: Deep dive into the distribution strategy.
- **[Backend Rules](./.gemini/rules/backend.md)**: Coding standards for Python developers.
- **[Frontend Rules](./.gemini/rules/frontend.md)**: Coding standards for React developers.

---

## 🛡️ Security & Principles

- **Multi-Tenancy**: DB-per-tenant isolation for maximum data security.
- **Money Rule**: Strict use of `Decimal` (4 decimal places) for all financial values.
- **Soft Deletes**: Physical deletion is forbidden; use the `is_deleted` flag.
- **Event-Driven**: Cross-module communication happens exclusively via the Event Bus.
