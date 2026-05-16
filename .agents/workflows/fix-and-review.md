# Workflow: Architecture Review and Fix

This workflow describes how to perform architecture reviews and implement fixes for the BES Factory project.

---

## When to Use This Workflow

- Before a major release or deployment
- After implementing a new module
- When onboarding a new client
- Periodically (monthly) as a health check

## Phase 1: Review

### 1.1 Security Audit
- [ ] No hardcoded secrets in source code (`grep -r "password" --include="*.py"`)
- [ ] JWT_SECRET_KEY sourced from environment
- [ ] INITIAL_ADMIN_PASSWORD sourced from environment (or DEBUG-only fallback)
- [ ] All authenticated endpoints use `Depends(get_current_user)`
- [ ] `/api/v1/bootstrap` requires authentication
- [ ] CORS origins are explicitly whitelisted (no wildcards)
- [ ] No `.env` files with real secrets committed to git

### 1.2 Data Integrity Check
- [ ] All monetary fields use `Decimal` with `Numeric(20,4)` — never `float`
- [ ] All tables inherit from `BESBase`
- [ ] No physical deletions — all delete operations use `is_deleted = True`
- [ ] `updated_at` auto-updates on modifications
- [ ] All list queries filter by `is_deleted == False`

### 1.3 Architecture Compliance
- [ ] Every extension has a `manifest.py` with `ExtensionManifest`
- [ ] Every extension follows the layer structure: models / schemas / services / router / events
- [ ] No extension imports another extension directly (use Event Bus)
- [ ] Routers are thin — no business logic in HTTP handlers
- [ ] API schemas are separate from ORM models (no raw models as API input)
- [ ] All list endpoints have pagination

### 1.4 RBAC Verification
- [ ] `require_permission()` uses role-based checking (not global admin schema)
- [ ] Default roles exist: admin, manager, staff
- [ ] Users have `role_id` assigned
- [ ] Superuser bypass works correctly
- [ ] Permission elevation (`elevate_context`) works for system operations

### 1.5 Operational Readiness
- [ ] `/health` endpoint exists and returns module count
- [ ] `RequestLoggingMiddleware` is active
- [ ] `ContextAwareSecurityMiddleware` is active
- [ ] Database files are not created at repository root
- [ ] `X-Request-ID` header is present in responses

## Phase 2: Fix

For each issue found:

1. **Categorize**: Critical (security/data) vs High (architecture) vs Medium (code quality)
2. **Document**: Record the issue, affected file, and proposed fix
3. **Implement**: Apply the fix following the rules in `.gemini/rules/`
4. **Verify**: Test the fix (API call, startup check, or unit test)

## Phase 3: Verify

### Quick Verification Script

```bash
# Backend startup test
cd bes-backend
JWT_SECRET_KEY=test-secret DEBUG=true \
  pdm run uvicorn instances.acme_corp.acme_corp.main:app --port 8000 &

sleep 3

# Health check
curl -s http://localhost:8000/health | python -m json.tool

# Login
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -d "username=admin&password=admin123" | python -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Bootstrap (should require auth)
curl -s http://localhost:8000/api/v1/bootstrap \
  -H "Authorization: Bearer $TOKEN" | python -m json.tool

# Unauthenticated bootstrap (should fail)
curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/api/v1/bootstrap
# Expected: 401

# Cleanup
kill %1
```

## Common Issues and Fixes

| Issue | Root Cause | Fix Reference |
|:------|:-----------|:-------------|
| `float` for money | Missing `sa_column=Column(Numeric(...))` | Rules: backend.md §1 |
| ORM model as API input | Missing schemas.py | Rules: backend.md §4 |
| No pagination | Missing `PaginationParams` dependency | Rules: backend.md §6 |
| Business logic in router | Missing services.py | Rules: backend.md §4 |
| Extension cross-import | Direct import instead of Event Bus | Rules: architecture.md §6 |
| Hardcoded secret | Missing env var lookup | Rules: security.md §1 |
