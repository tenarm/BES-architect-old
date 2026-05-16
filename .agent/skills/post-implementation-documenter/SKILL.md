---
name: post-implementation-documenter
description: This skill instructs the AI assistant to generate a detailed post-implementation technical manual for a feature or module after coding is complete.
---

# Skill: Post-Implementation Documenter

This skill instructs the AI assistant to perform a "Code-to-Doc" audit of a finished implementation. It analyzes the actual source code (backend and frontend) and generates a comprehensive technical manual that serves as the final documentation for the feature.

---

## Instructions for the Assistant

When the user asks you to "document the implementation" of a specific feature:

### 1. Source Code Audit
- Scan the backend extension: `bes-backend/extensions/<module>/`.
- Scan the frontend library: `bes-frontend/libs/<module>/`.
- Read the final `models.py`, `services.py`, `router.py`, and `events.py`.
- Read the React components and their registration logic.
- Compare the code against the original `changes.md` to identify any implementation-time deviations.

### 2. Documentation Sections

#### A. Final API Specification
- Document the exact FastAPI routes.
- Provide example Request/Response JSON payloads.
- List the enforced `require_permission` keys for each endpoint.

#### B. Database Schema (As-Built)
- List all SQLModel tables.
- Confirm inclusion of all `BESBase` audit and multi-tenancy fields.
- Document any complex relationships or `metadata_` JSONB usage.

#### C. Business Logic & Services
- Summarize the core logic within `services.py`.
- Document transaction handling and any system-elevated (`elevate_context`) flows.

#### D. Frontend Architecture
- List the primary components and their purpose.
- Document the state management (e.g., React Query, local state) and SSE integrations.
- Show the shell navigation path and registration details.

#### E. Event & Pub/Sub Catalog
- List every event emitted and subscribed to by this feature.
- Document the exact schema for each event payload.

#### F. Security & Compliance Audit
- Verify that `subsidiary_id` filters are present in all service-level queries.
- Confirm that the UI correctly degrades for `READONLY_EXTENSIONS` based on the implementation.

### 3. Output Format

Generate a **Post-Implementation Technical Manual** at `features-plan/<module>/<feature>/implementation-manual.md`.

## 1. Implementation Summary
A high-level overview of the as-built feature.

## 2. Technical Specs (Backend)
Detailed breakdown of models, services, and APIs.

## 3. Technical Specs (Frontend)
Detailed breakdown of UI components, routing, and shared-ui usage.

## 4. Security & Audit Trail
Final list of permissions and audited actions.

## 5. Integration Points
External events, dependencies, and shell configuration details.

## 6. Maintenance & Troubleshooting
Common failure points, specific error codes, and how to debug the feature (e.g., which logs to check).
