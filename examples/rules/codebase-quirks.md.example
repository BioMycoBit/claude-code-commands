# Codebase Quirks

Knowledge about module dependencies, import patterns, and gotchas.
Discovered during implementation. Updated when quirks are fixed.

**Cap:** 150 lines | **Write:** During /implement when quirk discovered | **Prune:** When fixed

---

## Active Quirks

### Example: Lazy Imports Break unittest.mock.patch
**Added:** YYYY-MM-DD
Router uses lazy import inside function. `patch("module.name")` fails with
`AttributeError` because the name never exists at module level. Patch at the
**source** module instead: `patch("original.module.name")`.

### Example: CRLF Line Endings from Write Tool
**Added:** YYYY-MM-DD
The Write tool may produce CRLF line endings. Shell scripts (.sh) break with
"cannot execute: required file not found". Run `sed -i 's/\r$//' <file>`
after creating or editing .sh files.

### Example: CSS Module Classes Not Queryable by Name
**Added:** YYYY-MM-DD
CSS modules (`.module.css`) hash class names at build time. Test scripts can't use
`[class*="myClass"]` to find elements. Use `data-*` attributes instead
(e.g., `data-testid`). Add `data-*` attrs to components that need test/debug queries.

### Example: Test Client Needs Rate Limit Env Vars
**Added:** YYYY-MM-DD
Rate limiter reads env vars at module import time, not at call time. Set
`RATE_LIMIT_*` env vars before importing the server module in tests.
CSRF middleware also blocks POST requests — use matching cookie + header in test setup.

### Example: Root Logger Level Hides New Module Logs
**Added:** YYYY-MM-DD
Logging setup sets root logger to `WARNING`. Only allowlisted loggers get `INFO`.
New modules adding `logger.info()` calls will silently drop unless added to the
allowlist in the logging setup function.

### Example: Venv Rename Breaks Activate Script + Shebangs
**Added:** YYYY-MM-DD
`python3 -m venv` bakes absolute paths into `bin/activate` (VIRTUAL_ENV) and
wrapper script shebangs. If you rename the venv directory, activation silently
fails. Fix: `sed -i` all references in `bin/activate*` and wrapper shebangs.

---

*Add entries as you discover gotchas. Remove when the quirk is fixed.*
