# Security Rules — BES Factory

These rules are non-negotiable security requirements for the BES platform.

---

## 1. Secrets Management

- **NEVER** hardcode secrets (JWT keys, database passwords, API keys) in source code.
- **NEVER** provide fallback defaults for secret values. Use `os.environ["KEY"]` (fail-fast).
- Secrets MUST be injected via environment variables or a secrets manager.
- `.env` files with secrets MUST be in `.gitignore`.
- Development-only secrets are allowed ONLY when gated behind `DEBUG=true`.

## 2. Authentication

### JWT Tokens
- Access tokens: short-lived (default 60 minutes, configurable).
- Refresh tokens: longer-lived (default 7 days), stored in database, revocable.
- Token type is embedded in the payload (`type: "access"` or `type: "refresh"`).
- Refresh tokens CANNOT be used as access tokens (enforced via type checking).

### Password Security
- Passwords are hashed using bcrypt via `passlib`.
- NEVER log or return passwords in API responses.
- Initial admin credentials MUST come from environment variables.
- Dev-only admin seeding is gated behind `DEBUG=true`.

## 3. Authorization (RBAC)

- Every API endpoint that modifies or reads sensitive data MUST use `require_permission()`.
- Permission format: `"module:resource:action"` (e.g., `"finance:gl:write"`).
- Permissions are role-based: each `Role` has a `permissions` JSON column.
- Three default roles: `admin` (full), `manager` (read/write), `staff` (read-only).
- Superusers bypass all permission checks.
- SYSTEM context (`elevate_context()`) bypasses checks for event-driven operations.

## 4. Multi-Tenant Data Isolation

- Each client has a **dedicated database**. Data NEVER leaks between clients.
- Within a client's database, `subsidiary_id` provides organizational scoping.
- The `ContextAwareSecurityMiddleware` extracts `X-Subsidiary-Id` from request headers.
- `BaseRepository._apply_scopes()` automatically filters queries by subsidiary.

## 5. API Security

- The `/api/v1/bootstrap` endpoint requires authentication.
- The `/health` endpoint is intentionally unauthenticated (for load balancer probes).
- CORS origins are explicitly whitelisted (no wildcard `*` in production).
- All responses use the `StandardResponse` envelope to prevent data leakage.
- Internal error details are NEVER exposed to clients (global exception handler returns generic messages).

## 6. Input Validation

- NEVER use ORM models as API input. Always use Pydantic `*Create` schemas.
- This prevents users from injecting `id`, `is_deleted`, `subsidiary_id`, `created_at`, etc.
- All input goes through Pydantic validation before reaching service layer.
- Use `RequestValidationError` handler for consistent error responses.

## 7. Audit Trail

- Every record has `created_at`, `updated_at`, and `created_by` fields.
- `updated_at` auto-updates on every modification (SQLAlchemy before_update event).
- Physical deletion is FORBIDDEN — use soft delete (`is_deleted = True`).
- All request/response pairs are logged with a unique `X-Request-ID`.

## 8. Frontend Security

- Tokens are stored in localStorage (consider HttpOnly cookies for production).
- Dev-only features (quick login, debug tools) are gated behind `import.meta.env.DEV`.
- The UI MUST NOT display modules/features the user lacks permissions for.
- Permission checks happen both on the backend (authoritative) and frontend (UX).
