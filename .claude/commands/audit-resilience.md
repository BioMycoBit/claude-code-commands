# /audit-resilience

Error handling & resilience audit: error swallowing, missing timeouts, retry logic gaps, graceful degradation paths, error propagation chains, and resource cleanup across Python backend, TypeScript frontend, and Rust relay.

**Target:** `backend/`, `src/`, `your-rust-service/src/`

---

## Step 1: Load Shared Infrastructure

1. Read `.claude/commands/audit/shared.md`
2. All output format, severity levels, delta tracking, archival, tech debt integration, commit format, sprint chain, error handling, and completion report rules come from shared.md — do NOT deviate

---

## Step 2: Parse Flags

Parse `$ARGUMENTS` per shared Flag Parsing rules:

1. Strip flags from arguments, set booleans:
   - `AGENT_MODE` — `true` unless `--quick`
   - `COMPARE` — `true` if `--since last`
   - `SPRINT_CHAIN` — `true` if `--sprint`
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
   - `AUTO_FIX` — `false` always (resilience audit is read-only, `--fix` is a no-op — report but do not modify any error handling code)
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for resilience audit (read-only by design)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Resilience_Audit.md 2>/dev/null | sort -r | head -1
```

If found, read it for delta comparison later. If not, set `Prior: None`.

---

## Step 4: Archive Prior (if exists)

Follow shared.md Archival rules:
- `git mv` prior to `docs/archive/audits/MMDDYY/`
- Handle collision with `_2` suffix
- Run BEFORE writing new audit document

---

## Step 5: Run Analysis Phases

Run all 6 phases (or 4 if `--quick` — skip agent analysis in Phase 4 and Phase 5), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use **Semgrep** (via Bash tool) for AST-aware pattern scans in Phases 1-2
(error swallowing, timeout coverage). Semgrep eliminates false positives from comments,
strings, and test fixtures. Use the built-in **Grep tool** for patterns Semgrep can't
express (contextual checks, Rust expect() message review). Issue independent tool calls
in a **single parallel message**.

### Resilience-Specific Delta Logic

In delta mode (`--since last`):
- Filter CHANGED_FILES to `.py`, `.ts`, `.tsx`, `.rs` extensions only
- Re-scan only changed files for Phases 1-3 and Phase 6 (grep-based)
- Agent phases (4, 5) always run full scope in full mode (degradation paths don't localize to single files)
- Carry forward findings from unchanged files

### Phase 1: Error Swallowing

Scan Python, TypeScript, and Rust for swallowed or silenced errors.

**Semgrep AST-aware scan (Python + TypeScript):** 1 Bash call
```bash
semgrep --config .semgrep/resilience.yml --json backend/ src/ 2>/dev/null
```
Parse JSON output: `results[]` array. Filter by `extra.metadata.phase: 1`.

This replaces grep patterns 1-7 with AST-aware matching that won't flag:
- Error handling in comments or docstrings
- Exception patterns inside string literals
- Test fixture exception handling (paths excluded in rules)

Covers: bare except+pass, broad except Exception, empty catch blocks, .catch(() => {}), catch-log-only, Rust unwrap() (non-test).

**Grep fallback — expect() context review (Rust):** 1 Grep call
Pattern: `\.expect\("` in `your-rust-service/src/**/*.rs` — review for generic messages
(Kept as Grep — requires human review of message quality, not a structural issue)

**Analysis:**
- Bare `except:` and `except Exception: pass` are the highest risk — errors vanish silently
- `.catch(() => {})` in TypeScript is equivalent to bare except
- `except` with only logging but no re-raise or recovery may hide failures
- Rust `unwrap()` in non-test code is a panic risk — `?` operator is preferred

**Severity mapping:**
- Bare `except: pass` or empty `.catch(() => {})` in production hot path (WS handlers, device callbacks, signal processing) = CRITICAL
- `except Exception` without re-raise in data pipeline = HIGH
- `unwrap()` in relay hot path (message routing, connection handling) = HIGH
- Catch-log-only without recovery action in error path = MEDIUM
- Broad `except Exception` with logging and recovery = LOW
- `unwrap()` in initialization/config code = LOW

**LOC Impact:** ~1-5 lines per finding (the try/except or catch block)
**Effort:** `trivial` (add specific exception type) to `small` (add recovery logic)

### Phase 2: Timeout Coverage

Scan for external calls missing timeout parameters.

**Semgrep AST-aware scan (Python):** included in Phase 1 Semgrep run above.
Filter by `extra.metadata.phase: 2`. Covers:
- httpx calls without `timeout=` parameter
- Redis connections without `socket_timeout=`

**Grep fallback patterns (parallel)** — for patterns Semgrep can't express (contextual proximity checks):

1. **aiohttp without timeout (Python)** — `aiohttp\.ClientSession\(` and `session\.(get|post)` in `**/*.py` — then verify `timeout=` within ~5 lines
2. **Database long queries (Python)** — `\.execute\(|\.sql\(` in `**/*.py` — check for threading.Timer timeout guard (Some databases lack `statement_timeout`)
3. **WebSocket connect without timeout (Python)** — `websockets\.connect\(|websocket\.connect\(` in `**/*.py` — verify `timeout` nearby
4. **fetch without AbortController (TypeScript)** — `fetch\(` in `src/**/*.{ts,tsx}` — check for `AbortController` or `signal` nearby
5. **WebSocket without timeout (TypeScript)** — `new WebSocket\(` in `src/**/*.{ts,tsx}` — check for connection timeout logic nearby
6. **reqwest without timeout (Rust)** — `reqwest::Client` in `your-rust-service/src/**/*.rs` — verify `.timeout(` present
7. **tokio timeout (Rust)** — `tokio::time::timeout` in `your-rust-service/src/**/*.rs` — presence = good

**Analysis:**
- For each external call found, check if a timeout is specified within ~5 lines of context
- External calls: HTTP, Redis, databases, WebSocket, external APIs, device communication
- Missing timeout = call can block indefinitely, stalling the event loop or thread
- Some databases lack `statement_timeout` (debugging-patterns.md) — must use `threading.Timer` + `conn.interrupt()`

**Severity mapping:**
- External HTTP/WS call without timeout in production request path = CRITICAL
- Redis/database call without timeout in hot path = HIGH
- `fetch()` without AbortController in user-facing flow = HIGH
- WebSocket connect without timeout = HIGH
- External call without timeout in background/initialization = MEDIUM
- Internal-only call without timeout (localhost services) = LOW

**LOC Impact:** ~1-3 lines per finding (adding timeout parameter)
**Effort:** `trivial` (add timeout param) to `small` (add AbortController wrapper)

### Phase 3: Retry Logic

Scan for retry patterns and flag infinite or backoff-less retries.

**Grep patterns (parallel):**

1. **Tenacity usage (Python)** — `@retry|tenacity` in `**/*.py`
2. **Manual retry loops (Python)** — `for.*in range.*:\s*\n.*try:` or `while.*True.*:\s*\n.*try:` in `**/*.py` (multiline)
3. **Backoff decorator (Python)** — `@backoff` in `**/*.py`
4. **Sleep in retry (Python)** — `except.*:\s*\n.*sleep\(` in `**/*.py` (multiline)
5. **Retry in TypeScript** — `retry|retries|maxRetries|retryCount` in `src/**/*.{ts,tsx}`
6. **setTimeout retry (TypeScript)** — `setTimeout.*retry|setTimeout.*reconnect` in `src/**/*.{ts,tsx}`
7. **Reconnect patterns (TypeScript)** — `reconnect|auto.?reconnect` in `src/**/*.{ts,tsx}`
8. **Tokio retry (Rust)** — `loop\s*\{.*tokio|retry` in `your-rust-service/src/**/*.rs`

**Analysis:**
- Identify all retry mechanisms across the stack
- For each retry: does it have a max attempt limit? Does it use exponential backoff?
- Flag: `while True` retry without `max_retries` or backoff = infinite loop risk
- Flag: fixed `sleep()` between retries without backoff = thundering herd risk
- Cross-reference: retried operations should also have timeouts (Phase 2)

**Severity mapping:**
- Infinite retry loop (`while True` + no max) on external call = HIGH
- Retry without backoff on shared resource (Redis, relay) = HIGH
- Fixed sleep retry (no exponential backoff) on external service = MEDIUM
- Retry exists but max_retries too high (>10 for fast-fail service) = MEDIUM
- Missing retry on transient-failure-prone call (BLE connect, WS reconnect) = MEDIUM
- Retry with proper backoff and max = no finding (good pattern)

**LOC Impact:** ~5-15 lines per finding (adding/fixing retry logic)
**Effort:** `small` (add tenacity decorator) to `medium` (implement retry with backoff)

### Phase 4: Graceful Degradation (agent-driven)

**Grep patterns (always run):**

1. **Redis connection (Python)** — `redis|Redis|StrictRedis` in `**/*.py`
2. **Database connection (Python)** — `connect|Connection|database|db_client` in `**/*.py`
3. **Relay connection (Python)** — `relay|RelayClient|relay_client` in `**/*.py`
4. **LiveKit connection (Python)** — `livekit|LiveKit` in `**/*.py`
5. **WebSocket dependency (TypeScript)** — `wsManager|WebSocket|websocket` in `src/**/*.{ts,tsx}`
6. **Health check endpoints** — `health|/health` in `**/*.py`

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze graceful degradation paths in this project:
>
> **Findings:** [Insert dependency grep results — Redis, database, relay, external services usage locations]
>
> **Analyze for each dependency:**
> 1. **Redis down** — Does the backend crash, return errors, or degrade to local-only mode? Can sessions still function without shared state?
> 2. **Database down** — Does time-series storage fail silently, queue data, or crash the pipeline? Are reads vs writes handled differently?
> 3. **Relay down** — Does the backend detect relay absence? Do WebSocket clients get notified? Is there fallback (direct WS)?
> 4. **LiveKit down** — Does video fail independently of application data? Are connection errors surfaced to the user?
> 5. **BLE device disconnect** — Does the backend detect disconnect promptly? Is the frontend notified? Does reconnection attempt automatically?
>
> For each: Rate as FULL_CRASH / PARTIAL_FUNCTION / GRACEFUL_DEGRADE
>
> **Return:** Dependency failure matrix (dependency × failure mode × current behavior × recommended behavior)"

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- Dependency failure causes full application crash (FULL_CRASH) = CRITICAL
- Dependency failure causes silent data loss = CRITICAL
- Dependency failure crashes one subsystem but others continue (PARTIAL_FUNCTION without user notification) = HIGH
- Dependency failure detected but no user-visible feedback = HIGH
- Dependency failure degrades gracefully with user notification = no finding (good pattern)

**LOC Impact:** ~10-50 lines per finding (adding fallback logic)
**Effort:** `medium` (add fallback path) to `large` (implement degraded mode)

**On timeout/failure:** Output raw dependency grep results without agent analysis. Note `Agent: None (timeout)` in header.

### Phase 5: Error Propagation (agent-driven)

**Grep patterns (always run):**

1. **Error events emitted (Python)** — `emit.*error|error.*callback|notify.*error` in `**/*.py`
2. **Error WS messages (Python)** — `error.*message|send.*error|ws.*error` in `**/*.py`
3. **Error handlers (TypeScript)** — `onError|onerror|handleError|error.*handler` in `src/**/*.{ts,tsx}`
4. **Toast/notification (TypeScript)** — `toast|notification|alert|showError` in `src/**/*.{ts,tsx}`
5. **Error state (TypeScript)** — `error.*state|setError|errorMessage` in `src/**/*.{ts,tsx}`

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Trace error propagation in this project from device to user:
>
> **Findings:** [Insert error handling grep results — error events, WS error messages, frontend error handlers]
>
> **Trace these failure scenarios end-to-end:**
> 1. **External device disconnects mid-session** — Device layer → Backend → Relay → Frontend → What does the user see?
> 2. **Redis connection drops** — Backend detects → How is frontend notified? → What does user see?
> 3. **WebSocket connection drops** — Frontend detects → Reconnect attempt → User feedback during reconnect
> 4. **Backend crashes** — Process exits → Frontend WS closes → User sees...?
> 5. **Database write fails** — Data loss? Error surfaced? User aware?
>
> For each trace: identify gaps where errors are swallowed, transformed into generic messages, or never reach the user.
>
> **Return:** Error propagation matrix (scenario × stages × gap locations × user-visible result)"

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- Error in data pipeline with no user notification and silent data loss = CRITICAL
- Device disconnect with no frontend notification = HIGH
- Backend error transformed to generic "something went wrong" (no actionable info) = HIGH
- Error reaches frontend but no UI treatment (logged to console only) = MEDIUM
- Error fully propagated with clear user message = no finding (good pattern)

**LOC Impact:** ~5-20 lines per finding (adding error propagation)
**Effort:** `small` (add WS error message) to `medium` (implement error UI)

**On timeout/failure:** Output raw error handling grep results without agent analysis. Note `Agent: None (timeout)` in header.

### Phase 6: Resource Cleanup

Scan for missing cleanup in exception paths.

**Grep patterns (parallel):**

1. **finally blocks (Python)** — `finally:` in `**/*.py`
2. **Context managers (Python)** — `async with|with ` in `**/*.py`
3. **Manual close (Python)** — `\.close\(\)` in `**/*.py`
4. **atexit/shutdown hooks (Python)** — `atexit|shutdown|on_shutdown|lifespan` in `**/*.py`
5. **useEffect cleanup (TypeScript)** — `return\s*\(\)\s*=>|return\s*\(\s*\)\s*\{` inside useEffect in `src/**/*.{ts,tsx}`
6. **removeEventListener (TypeScript)** — `removeEventListener|unsubscribe|cleanup|dispose` in `src/**/*.{ts,tsx}`
7. **AbortController cleanup (TypeScript)** — `abort\(\)|AbortController` in `src/**/*.{ts,tsx}`
8. **Drop trait (Rust)** — `impl Drop|drop\(` in `your-rust-service/src/**/*.rs`
9. **Rust cleanup (Rust)** — `tokio::signal|graceful.*shutdown|shutdown` in `your-rust-service/src/**/*.rs`

**Analysis:**
- For resources opened in `try:` blocks, verify `finally:` or context manager handles cleanup
- For `useEffect` with subscriptions/timers, verify cleanup function returned
- Cross-reference Phase 1: if error is swallowed AND no cleanup → resource leak
- Check: file handles, DB connections, WS connections, BLE connections, timers, event listeners

**Severity mapping:**
- DB connection or file handle opened without cleanup in exception path = HIGH
- WebSocket/BLE connection without disconnect on error = HIGH
- `useEffect` with subscription but no cleanup return = HIGH (React memory leak)
- Missing `finally:` on non-critical path with resource = MEDIUM
- Timer without `clearTimeout`/`clearInterval` on cleanup = MEDIUM
- Cleanup exists but only on happy path (not in except/catch) = MEDIUM
- `async with` used properly = no finding (good pattern)

**LOC Impact:** ~2-10 lines per finding (adding finally/cleanup)
**Effort:** `trivial` (add finally block) to `small` (refactor to context manager)

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Resilience_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Resilience Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Resilience_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Read (file analysis) [list any additional tools used]
```

### 6b. Executive Summary

2-3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., bare except in hot path, missing timeout on Redis)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Python files scanned | N | — | — |
| TypeScript files scanned | N | — | — |
| Rust files scanned | N | — | — |
| Bare except/empty catch | N | — | — |
| External calls without timeout | N | — | — |
| Retry patterns found | N | — | — |
| Dependencies with graceful degradation | N/5 | — | — |
| useEffect hooks with cleanup | N/N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **E-** prefix (E-1, E-2, E-3... for Error/resilience findings).

### 6e. Auto-Fixed Section

**Not applicable for resilience audit** — this audit is read-only by design. If `--fix` was passed, note:
```markdown
## Auto-Fixed

N/A — Resilience audit is read-only. Use findings to manually improve error handling.
```

### 6f. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules:
- New Findings (not in prior)
- Resolved (was in prior, now gone)
- Unchanged (count only)
- Trend Summary table

### 6g. Historical Section

If prior existed, list resolved items:
```markdown
## Historical (Resolved)

- [MMDDYY] Fixed: [brief description from prior findings no longer present]
```

### 6h. Dependency Failure Matrix

Include the agent-generated matrix (or stub if `--quick`):
```markdown
## Dependency Failure Matrix

| Dependency | Failure Mode | Current Behavior | Rating | Recommended |
|------------|-------------|-----------------|--------|-------------|
| Redis | Connection drop | ? | FULL_CRASH / PARTIAL / GRACEFUL | ? |
| Database | Write failure | ? | FULL_CRASH / PARTIAL / GRACEFUL | ? |
| Relay | Unreachable | ? | FULL_CRASH / PARTIAL / GRACEFUL | ? |
| LiveKit | Connection drop | ? | FULL_CRASH / PARTIAL / GRACEFUL | ? |
| BLE Device | Disconnect | ? | FULL_CRASH / PARTIAL / GRACEFUL | ? |
```

### 6i. Error Propagation Matrix

Include the agent-generated matrix (or stub if `--quick`):
```markdown
## Error Propagation Matrix

| Scenario | Device Layer | Backend | Relay | Frontend | User Sees |
|----------|-------------|---------|-------|----------|-----------|
| Device disconnect | ? | ? | ? | ? | ? |
| Redis drop | — | ? | ? | ? | ? |
| WS drop | — | — | — | ? | ? |
| Backend crash | — | crash | — | ? | ? |
| DB fail | — | ? | — | ? | ? |
```

### 6j. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. CRITICAL findings first (bare except in hot path, missing timeout on production call)
2. HIGH findings (no graceful degradation, missing cleanup, infinite retry)
3. Quick wins with highest resilience-to-effort ratio

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

1. Read `.claude/rules/tech-debt-patterns.md`
2. For each new CRITICAL or HIGH finding: check if already has an entry, if not append per shared format
3. For resolved findings: mark existing entries as `~~RESOLVED~~` with date
4. Respect 100-line cap — evaluate oldest entries for removal if near cap

### 7b. Git Commit

```bash
git add work/MMDDYY_Resilience_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): resilience audit — N findings (X critical, Y high)

Tools: Grep, Read
Mode: Full | Quick
[Delta: +N new, -N resolved]  (only if --since last)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### 7c. Completion Report

Output to user per shared.md Completion Report format — findings summary table, top 3 findings, output file paths.

### 7d. Sprint Chain

Only if `SPRINT_CHAIN` is true:
- If 0 critical/high findings: skip chain, show "No critical/high findings — sprint chain skipped"
- Otherwise: build sprint context per shared.md Sprint Chain rules and invoke `/sprint`

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key resilience-specific items:

- [ ] All 6 phases ran (or 4 in quick mode — skip Phases 4 and 5) — each phase produced findings or confirmed clean
- [ ] Phase 1: All bare `except:`, `except: pass`, empty `.catch()`, `unwrap()` cataloged
- [ ] Phase 2: All external calls (HTTP, Redis, databases, WS, external services) checked for timeout parameters
- [ ] Phase 3: All retry patterns checked for max attempts and backoff
- [ ] Phase 4: All 5 dependencies assessed for graceful degradation (agent in full mode)
- [ ] Phase 5: All 5 failure scenarios traced end-to-end (agent in full mode)
- [ ] Phase 6: All resource-opening code checked for cleanup in exception paths
- [ ] Severity mapping applied per phase-specific rules above
- [ ] Resilience-specific metrics populated (bare except count, timeout coverage, degradation ratings)
- [ ] Findings numbered sequentially with E- prefix (E-1, E-2, ...)
- [ ] `--fix` noted as N/A (read-only audit)
- [ ] Delta filters to `.py`, `.ts`, `.tsx`, `.rs` files only
- [ ] Dependency Failure Matrix and Error Propagation Matrix included (or stubbed in quick mode)
- [ ] Cross-stack scope verified: Python + TypeScript + Rust all scanned

---

## See Also

- `/audit-docker` — Docker infrastructure audit (container security, resource limits, network isolation)
- `/audit-python` — Python code quality audit (radon, vulture, ruff)
- `/audit-typescript` — TypeScript audit (circular deps, dead exports, oxlint)
- `/audit-rust` — Rust audit (clippy, unwrap audit, PyO3 bindings)
- `/audit-security` — OWASP security audit (overlaps with error handling checks)
- `/eod-docs day1` — Day 1 Code Quality includes Best Practices audit (complementary)
