# Debugging Patterns

Known failure modes and their fixes. Quick-scan format: see symptom → apply fix.

**Cap:** 200 lines (40 entries max) | **Write:** After debugging, /park | **Prune:** Oldest first

---

## Patterns

### Example: API Timeout on First Request (YYYY-MM-DD)
Cold start causes first request to timeout → add warmup call in startup lifecycle.

### Example: Test Flake from Shared State (YYYY-MM-DD)
Tests pass individually but fail in suite → shared module-level state not reset between tests. Use fixtures with proper teardown.

### Example: CSRF Middleware Blocks First POST (YYYY-MM-DD)
403 on first POST → CSRF cookie only set after first GET. Exempt auth-only data-read endpoints, or ensure a GET fires before the first POST.

### Example: Environment Variable Read at Import Time (YYYY-MM-DD)
Config change has no effect → module reads env var at top level during import, not at call time. Move to function-level read or use a settings object with lazy loading.

### Example: React useRef After Early Return — Hooks Order Crash (YYYY-MM-DD)
"Rendered more hooks than previous render" → hook placed after `if (!x) return null`. ALL hooks must be above early returns. React requires consistent hook call order across renders.

### Example: ORM Lazy Load N+1 in API Response (YYYY-MM-DD)
Endpoint takes 5s for 50 records → each record triggers a lazy-loaded relationship query. Use `selectinload()` / `joinedload()` or equivalent eager loading in the query.

### Example: Database Migration Works Locally, Fails in CI (YYYY-MM-DD)
Migration depends on data that exists in dev seed but not in CI's empty database. Make migrations data-independent or add conditional checks. Never rely on specific row existence.

### Example: Auth Token Refresh Race Condition (YYYY-MM-DD)
Multiple concurrent requests all detect expired token → all trigger refresh simultaneously → only first succeeds, rest get 401. Use a mutex/lock so only one refresh runs at a time, others wait for the result.

---

*Add entries as you discover failure modes. Format: symptom → root cause → fix.*
