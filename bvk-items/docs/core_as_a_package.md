# Core as a Package Architecture

## Overview
"Core as a Package" is an Entbesrise-Grade software distribution strategy for multi-tenant SaaS / BES platforms. In this model, the **"Engine"** (the common, rigid framework logic) is treated as a private library (like FastAPI or Pandas). The **"Extensions"** (client-specific business logic) are treated as individual applications that import the Engine.

Essentially: **The Core is your Product, and the Extension is the Service you provide to the client.** This keeps the business scalable and the code unbreakable.

## 1. Repository Architecture
We enforce a strict physical firewall between the high-value Intellectual Property (the Core) and client-specific logic (the Extensions).

- **Repo A: `bes-core` (The Library / Engine)**
  - **Content**: FastAPI engine, database handlers, authentication & JWT handling, abstract base classes, and common middlewares (logging, error handling).
  - **Output**: A versioned Python package (e.g., `bes-core-1.4.2.whl`).
  - **Storage**: Hosted in a Private Package Registry (e.g., GitHub Packages, AWS CodeArtifact).

- **Repo B: `client-[name]-extension` (The App / Chassis)**
  - **Content**: Custom business logic, modules, and reports specific to a client (e.g., `client-acme-extension`).
  - **Dependency**: The `pyproject.toml` or `requirements.txt` specifically requests a version of the core (e.g., `bes-core==1.4.2`).
  - **Output**: A single Docker Image per client.

## 2. CI/CD & The Fused Build Process
Although the code lives in separate repositories, the Docker Image acts as the "blender" that fuses them into one running application.

**The Build Steps:**
1. **Trigger**: Code is pushed to `client-acme-extension`.
2. **Pull**: The Build Server (GitHub Actions/Jenkins) clones the client extension repo.
3. **Install**: The server runs `pip install`. Because `bes-core` is listed as a dependency, it downloads the pre-built Core from the Private Registry.
4. **Package**: The server copies the client's custom files into the image.
5. **Result**: A single, self-contained image: `acme-bes-backend:v1.3`.

## 3. Local Development Workflow
To test changes across both the Core and an Extension simultaneously without constantly publishing to the registry, developers use Python's "Editable Mode":

1. Clone both repositories to the local machine (e.g., `bes-core` and `client-acme-extension`).
2. Inside the client repo, run: `pip install -e ../bes-core`.
3. Changes made to the Core files are instantly reflected in the Extension environment without requiring rebuilds or republishing.

## 4. Operational Advantages

| Feature | Core as a Package | Traditional Monolith |
| :--- | :--- | :--- |
| **Code Storage** | Multiple Repos (Core + Client Repos) | One Giant Repo |
| **Edit Risk** | Isolated (Changes only affect the specific client) | High (Changes can ripple through all clients) |
| **Updating** | Version-based (`pip install --upgrade`) | File-based (Copy/Paste or Git Merge) |
| **Customization** | Native (Each client has their own "App") | Hardcoded (`if-else` statements everywhere) |

### Key Benefits
- **Version Pinning**: Client A can remain on core `v1.0` (for audits or stability) while Client B upgrades to `v2.0`. There is total control over individual release cycles.
- **Access Control & IP Protection**: Junior developers or client developers can be granted access to the Extension repo to build custom reports without ever seeing the source code of `bes-core`.
- **Atomic Rollbacks**: If a new extension deployment fails, roll back that specific client's Docker image without affecting the rest of the fleet.
- **Clean Development Environment**: Developers only see the business logic they need to edit, free from the thousands of lines of Core engine code.

---

## 5. Monorepo Alignment

While this document describes a multi-repo distribution strategy, the current **BES Monorepo** (using PDM and Nx) is designed to simulate this exact isolation during the development phase.

- **PDM Workspaces**: The `bes-backend/core` and `bes-backend/extensions/*` are treated as local packages.
- **Dependency Enforcement**: Extensions explicitly list `core` in their `pyproject.toml`, mirroring the "Core as a Package" dependency model.
- **Path to Distribution**: This monorepo structure allows for seamless transition to private package registries when the platform moves to a global SaaS scale.
