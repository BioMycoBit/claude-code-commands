# Tech Debt Patterns

Recurring findings from audits and sprint retrospectives.
Items removed when resolved.

**Cap:** 100 lines | **Write:** During audits, sprint-end | **Prune:** When resolved

---

## Active Debt

### Example: Test Coverage Below Target
**Added:** YYYY-MM-DD | **Source:** /audit-test-quality
Auth middleware has 12% test coverage vs 80% target. Login, logout, and token
refresh paths untested. Integration tests exist but mock the database.
**Location:** `src/auth/middleware.py`, `tests/auth/`
**Resolve:** Write integration tests hitting real database for auth flow.

### Example: Deprecated Dependency Still in Use
**Added:** YYYY-MM-DD | **Source:** /audit-dependency
`some-library@2.x` is deprecated, v3 has breaking API changes. Currently used
in 4 modules. v3 migration requires updating all call sites.
**Location:** `package.json`, `src/services/`
**Resolve:** Migrate to v3 API. Update all `someLib.oldMethod()` → `someLib.newMethod()`.

### Example: Missing API Response Types
**Added:** YYYY-MM-DD | **Source:** /audit-types
9 endpoints return raw dicts instead of typed response models. No contract
enforcement — clients can't rely on response shape.
**Location:** `src/api/routers/`
**Resolve:** Add `response_model` to all router decorators. Create Pydantic models.

### Example: .env Tracked in Git History
**Added:** YYYY-MM-DD | **Source:** /audit-secrets
`.env` tracked in git history with local-dev credentials. Acceptable during
local-only development. Must scrub history before any public/cloud deployment.
**Location:** `.env`, git history
**Resolve:** Before deploy: `git filter-repo` to scrub history, rotate credentials,
add `*.pem *.key credentials*` to `.gitignore`.

### ~~Example: Resolved Item~~ RESOLVED vX.Y Session Z
Strikethrough + RESOLVED tag shows how to mark items done. Remove after confirming
the fix is stable for 1+ sprints.

---

*Track shortcuts and known issues here. Remove entries when resolved.*
