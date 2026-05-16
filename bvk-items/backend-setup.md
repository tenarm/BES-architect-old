# Core as a Package Architecture Setup

I have scaffolded the complete foundational structure for the "Core as a Package" architecture. We now have two distinct logical blocks: the **Library** (`bes-core`) and the **Application** (`client-acme-extension`).

## 1. The Core Engine (`bes-core`)
This directory acts as the pure technology engine. It is designed to be built into a Python package (e.g., a `.whl` file) and imported by extensions.

### Structure Created
```text
bes-core/
├── pyproject.toml              # Defines this as a package named 'bes-core'
└── bes_core/
    ├── __init__.py             # Exposes the package version
    ├── main.py                 # The factory function `create_core_app()` for FastAPI
    ├── auth/
    │   └── jwt.py              # Centralized authentication logic
    ├── database/
    │   └── session.py          # Centralized database sessions
    └── models/
        └── base.py             # Abstract base models for all modules
```

- **[NEW]** [bes-core/pyproject.toml](file:///Users/bvk/BVK_Workspace/BES_NEW/bes-core/pyproject.toml)
- **[NEW]** [bes-core/bes_core/main.py](file:///Users/bvk/BVK_Workspace/BES_NEW/bes-core/bes_core/main.py)

> [!NOTE]
> The Core does not have a `Dockerfile` because it is not meant to be run on its own; it is imported.

---

## 2. The Client Extension (`client-acme-extension`)
This directory represents a single client. It treats `bes-core` as a standard dependency, exactly like it treats `fastapi` or `pydantic`.

### Structure Created
```text
client-acme-extension/
├── Dockerfile                  # The "Blender" that fuses Core + App into an image
├── pyproject.toml              # Defines the app and lists "bes-core" as a dependency
└── acme_app/
    ├── __init__.py
    ├── main.py                 # Imports `create_core_app` from `bes_core` and fuses custom routers
    └── routers/
        ├── __init__.py
        └── custom_reports.py   # An isolated router exclusively for Acme Corp
```

- **[NEW]** [client-acme-extension/pyproject.toml](file:///Users/bvk/BVK_Workspace/BES_NEW/client-acme-extension/pyproject.toml)
- **[NEW]** [client-acme-extension/acme_app/main.py](file:///Users/bvk/BVK_Workspace/BES_NEW/client-acme-extension/acme_app/main.py)
- **[NEW]** [client-acme-extension/acme_app/routers/custom_reports.py](file:///Users/bvk/BVK_Workspace/BES_NEW/client-acme-extension/acme_app/routers/custom_reports.py)

## Local Development Workflow
To test this "fused" architecture seamlessly on your local machine using the "Editable" trick you mentioned, you can do the following:

1. Open your terminal and navigate to the client extension folder:
   ```bash
   cd client-acme-extension
   ```
2. Create a virtual environment and install the dependencies (including the local core in editable mode):
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -e ../bes-core
   pip install fastapi uvicorn
   ```
3. Run the fused application:
   ```bash
   uvicorn acme_app.main:app --reload
   ```

When you edit a file inside the `bes-core/bes_core/` directory, Uvicorn will detect the change through the editable link and restart the server instantly.
