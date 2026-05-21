# Modular BES Deployment Flow

This document details the containerized architecture, build processes, and deployment commands for shipping the Business Execution System (BES) to clients.

---

## 1. Architecture Overview

We use a **Two-Container Architecture** to separate concern, optimize performance, and keep configurations simple.

```
                  ┌──────────────────────────────────────────┐
                  │              Client Browser              │
                  └────────────────────┬─────────────────────┘
                                       │
                                       │ Port 80
                                       ▼
                  ┌──────────────────────────────────────────┐
                  │            Nginx (Frontend)              │
                  │        (bes-frontend-shell Container)    │
                  └──────────────┬────────────────────┬──────┘
                                 │                    │
                  Static Assets  │                    │ Proxy /api
                  (JS/CSS/HTML)  ▼                    ▼
                           [Serve Locally]     http://backend:8000
                                                      │
                                                      ▼
                                   ┌──────────────────────────────────┐
                                   │         FastAPI (Backend)        │
                                   │    (bes-backend-test_custom)     │
                                   └──────────────────┬───────────────┘
                                                      │
                                                      ▼
                                          SQLite/Postgres Database
                                          [Mounted Volume on Host]
```

### Key Architectural Strengths:
1. **Zero CORS Issues**: Because Nginx serves both the UI assets and proxies the API requests, the browser sees a single host/port (origin). CORS headers are not required on the backend.
2. **Dynamic Backend Routing**: The backend container destination (`BACKEND_URL`) is replaced at container startup in Nginx via a template environment variable. You don't need to rebuild the frontend image if the backend URL changes.
3. **Data Integrity & Persistence**: Database files are stored in a mounted host volume, preventing data loss when containers are deleted or updated.

---

## 2. Quick Start: Local Orchestration

The easiest way to run the entire system is using **Docker Compose** from the workspace root directory.

### Running the Default Client (`test_custom`)
By default, Docker Compose reads from the `.env` file in the root directory. If you haven't changed it, it will boot the `test_custom` instance:

```bash
# Build and start all services in detached mode
docker compose up --build -d
```

### Running a Specific Client Instance
If you want to run a different client instance (e.g. `acme_inc`), you can boot it using one of two methods:

#### Method A: Override Inline (Recommended for quick testing)
```bash
CLIENT_ID=acme_inc docker compose up --build -d
```

#### Method B: Modify the `.env` File (Recommended for persistent deployments)
Edit the `.env` file in the project root:
```env
CLIENT_ID=acme_inc
FRONTEND_PORT=80
BACKEND_PORT=8000
```
Then run:
```bash
docker compose up --build -d
```

### Access Points:
- **Web UI**: [http://localhost](http://localhost) (Served by Nginx on port `FRONTEND_PORT`)
- **API & Docs**: [http://localhost:8000/docs](http://localhost:8000/docs) (Served by FastAPI on port `BACKEND_PORT`)
- **Database File**: Persisted locally at `bes-backend/instances/<CLIENT_ID>/data/<CLIENT_ID>.db`.

```bash
# Shut down services and preserve volumes
docker compose down

# View logs
docker compose logs -f
```

---

## 3. Manual Build (Individual Images)

If you want to package the images separately to upload them to a container registry (like AWS ECR, Docker Hub, or client repository):

### A. Build Backend Image (Client Instance)
To build a backend image for a specific client (e.g., `<CLIENT_ID>`), run from the `bes-backend` directory so PDM has the correct monorepo build context:

```bash
cd bes-backend
docker build -f instances/<CLIENT_ID>/Dockerfile -t bes-backend-<CLIENT_ID>:latest .
```

To run the backend image standalone:
```bash
docker run -d -p 8000:8000 \
  -v $(pwd)/instances/<CLIENT_ID>/data:/app/instances/<CLIENT_ID>/data \
  --name backend-<CLIENT_ID> bes-backend-<CLIENT_ID>:latest
```

### B. Build Frontend Image
To build the frontend shell image, run from the `bes-frontend` directory:

```bash
cd bes-frontend
docker build -t bes-frontend-shell:latest .
```

To run the frontend image standalone (proxying to the backend container):
```bash
docker run -d -p 80:80 \
  -e BACKEND_URL=http://your-backend-ip:8000 \
  --name frontend-shell bes-frontend-shell:latest
```

---

## 4. Environment Variables

### Frontend Container (`frontend`)
| Variable | Default Value | Description |
| :--- | :--- | :--- |
| `BACKEND_URL` | `http://backend:8000` | The endpoint address where Nginx will forward all `/api` requests. |

### Backend Container (`backend`)
| Variable | Default Value | Description |
| :--- | :--- | :--- |
| `ENV` | `production` | Running environment (e.g., `production` or `development`). |
| `DATABASE_URL` | *(configured in config)* | Optional DB connection string. If omitted, uses the custom client SQLite database defined in `onboard_config.json`. |
