# /audit-test-quality

Test quality audit: test inventory, execution time profiling, flaky indicators, test isolation, mock hygiene, and coverage gaps across Python (~1,373 tests), frontend (Vitest ~1,037 + Playwright 107), and Rust (relay 115, signals 321).

**Target:** `tests/`, `src/**/__tests__/`, `e2e/`, `your-rust-service/tests/`, `your-rust-crate/src/`, `vitest.config.ts`, `pyproject.toml`

---

## Scope Boundary

**This audit covers:** Test suite quality — test inventory and density, execution time profiling, flaky test indicators, test isolation (shared state), mock hygiene (patch targets, over-mocking), coverage gap analysis by feature, and Plan agent coverage strategy.

**NOT covered here (see /audit-regression):** Regression risk mapping — bug fix commit analysis, danger zone identification, swallowed error detection in production code, and risk scoring based on commit history. This audit assesses test quality; `/audit-regression` maps where regressions are likely based on historical bug patterns.

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
   - `AUTO_FIX` — `false` always (test quality audit is read-only, `--fix` is a no-op — test changes require human review)
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for test quality audit (read-only by design)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_TestQuality_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 6 phases (or 5 if `--quick` — skip agent analysis in Phase 6), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent. Use Bash for test tool commands (`pytest --collect-only`, `vitest --reporter=json`) where applicable.

### Test Quality-Specific Delta Logic

In delta mode (`--since last`):
- Filter CHANGED_FILES to test files: `**/tests/**/*.py`, `**/__tests__/**/*.{ts,tsx}`, `**/e2e/**/*.ts`, `**/tests/**/*.rs`
- Also include config changes: `vitest.config.ts`, `pyproject.toml` (pytest config), `playwright.config.ts`
- Re-scan only changed test files for Phases 1-5 (grep-based + tool-based)
- Agent phase (6) always runs full scope in full mode (coverage strategy is holistic)
- Carry forward findings from unchanged test files
- If test config changed → full re-scan for all phases

### Phase 1: Test Inventory

Count and categorize tests across all stacks.

**Tool commands:**

1. **Python test count:**
   ```bash
   cd <backend-dir> && python -m pytest --collect-only -q 2>&1 | tail -5
   ```
   If pytest not available, fall back to grep-based count.

2. **Vitest test count:**
   ```bash
   cd <frontend-dir> && npx vitest --reporter=json --run 2>/dev/null | head -50
   ```
   If not available, fall back to grep-based count.

3. **Playwright test count:**
   ```bash
   cd <frontend-dir> && npx playwright test --list 2>&1 | tail -5
   ```
   If not available, fall back to grep-based count.

**Grep patterns (always run as supplement/fallback, parallel):**

1. **Python test functions** — `def test_|async def test_` in `tests/**/*.py`
2. **Python test classes** — `class Test` in `tests/**/*.py`
3. **Vitest test cases** — `(it|test)\(["']` in `src/**/__tests__/**/*.{ts,tsx}`
4. **Vitest describe blocks** — `describe\(["']` in `src/**/__tests__/**/*.{ts,tsx}`
5. **Playwright test cases** — `test\(["']` in `e2e/**/*.ts`
6. **Rust test functions** — `#\[test\]|#\[tokio::test\]` in `your-rust-service/tests/**/*.rs` and `your-rust-crate/src/**/*.rs`
7. **Rust test modules** — `#\[cfg\(test\)\]|mod tests` in `your-rust-service/src/**/*.rs` and `your-rust-crate/src/**/*.rs`

**Analysis:**
- Count tests by directory/feature area
- Identify directories with zero tests (coverage blind spots)
- Compare test count vs source file count per directory (test density)
- Note: 5 pre-existing Vitest failures (authStore/fetchWithAuth/useAppWebSocket) are known (tech-debt-patterns.md)

**Severity mapping:**
- Feature directory with zero tests (e.g., critical service with no test file) = HIGH
- Test density <0.5 (fewer tests than source files) in critical path = MEDIUM
- Test file exists but contains only 1-2 trivial tests = LOW
- Good test density (>1 test per source file) = no finding (good pattern)

**LOC Impact:** N/A for inventory (informational)
**Effort:** `medium` (write missing test files) to `large` (comprehensive test suite for untested feature)

### Phase 2: Execution Time

Profile test execution times and flag slow tests.

**Tool commands:**

1. **Python slow tests:**
   ```bash
   cd <backend-dir> && python -m pytest --durations=20 -q 2>&1
   ```
   If pytest not available, skip this tool command.

**Grep patterns (parallel):**

1. **Sleep in Python tests** — `time\.sleep\(|asyncio\.sleep\(` in `tests/**/*.py`
2. **Sleep in Vitest tests** — `setTimeout|sleep|delay|waitFor` in `src/**/__tests__/**/*.{ts,tsx}`
3. **Sleep in Playwright tests** — `page\.waitForTimeout|setTimeout|\.sleep\(` in `e2e/**/*.ts`
4. **Large fixture data** — `fixture|factory|generate.*data|bulk.*create|range\(\d{3,}` in `tests/**/*.py`
5. **Network calls in tests** — `httpx|requests\.get|fetch\(|aiohttp` in `tests/**/*.py`

**Analysis:**
- Flag unit tests >5s execution time (likely doing I/O or sleeping)
- Flag integration tests >30s execution time
- Flag `time.sleep()` in tests — usually a sign of timing-dependent test
- Flag network calls in unit tests — should be mocked
- `page.waitForTimeout()` in Playwright is acceptable for E2E but flag if >10s

**Severity mapping:**
- Unit test with `time.sleep(>5)` = HIGH (test is 10x slower than necessary)
- Unit test making real network calls = HIGH (should be mocked)
- Integration test >30s without clear reason = MEDIUM
- `page.waitForTimeout(>10000)` in Playwright = MEDIUM
- Unit test >5s (any cause) = MEDIUM
- Slow test with documented reason (hardware-dependent, large dataset) = LOW
- Fast, isolated tests = no finding (good pattern)

**LOC Impact:** ~1-10 lines per finding (remove sleep, add mock, optimize fixture)
**Effort:** `trivial` (remove unnecessary sleep) to `medium` (refactor test to use mocks)

### Phase 3: Flaky Indicators

Scan for non-deterministic patterns that cause intermittent test failures.

**Grep patterns (parallel):**

1. **time.time() in assertions (Python)** — `assert.*time\.time\(\)|time\.time\(\).*assert` in `tests/**/*.py`
2. **Date.now() in tests (TypeScript)** — `Date\.now\(\)|new Date\(\)` in `src/**/__tests__/**/*.{ts,tsx}`
3. **Random in tests (Python)** — `random\.|randint|randrange|choice\(|shuffle\(` in `tests/**/*.py`
4. **Math.random in tests (TypeScript)** — `Math\.random\(\)` in `src/**/__tests__/**/*.{ts,tsx}`
5. **Port binding in tests** — `bind\(|listen\(|port\s*=|:808[0-9]|:3[0-9]{3}` in `tests/**/*.py`
6. **File system in tests** — `open\(|write\(|os\.path|pathlib|tmp|temp` in `tests/**/*.py` (excluding fixtures)
7. **Environment-dependent (Python)** — `os\.environ|getenv|env\[` in `tests/**/*.py`
8. **Timing assertions** — `assert.*<\s*\d+\.\d+|assert.*elapsed|assert.*duration|assert.*latency` in `tests/**/*.py`
9. **Playwright Date.now() quirk** — `Date\.now\(\)` in `e2e/**/*.ts` (known issue: module-level constants differ across spec files — codebase-quirks.md)

**Analysis:**
- `time.time()` in assertions is fragile — depends on system load, CI environment speed
- `Date.now()` in fixtures produces different values across spec files (known Playwright quirk from codebase-quirks.md)
- Random values without fixed seed make failures non-reproducible
- Port binding in tests can collide with running services
- File system operations can fail on different OS or concurrent test runs
- Environment-dependent tests break on CI vs local mismatch

**Severity mapping:**
- Time-based assertion on fast operation (`assert elapsed < 0.1`) = HIGH (flaky on slow CI)
- `Date.now()` in shared test fixture (Playwright quirk) = HIGH (known cause of mismatches)
- Random values without seed in test logic = HIGH
- Port binding without dynamic port allocation = MEDIUM
- File system write without cleanup = MEDIUM
- `os.environ` read without fallback/default = MEDIUM
- Environment-specific test with skip decorator = LOW (handled correctly)
- Deterministic tests = no finding (good pattern)

**LOC Impact:** ~1-5 lines per finding (fix seed, use dynamic port, add cleanup)
**Effort:** `trivial` (add random seed) to `small` (refactor to use dynamic ports)

### Phase 4: Isolation

Scan for shared state between tests that can cause order-dependent failures.

**Grep patterns (parallel):**

1. **Module-level state (Python)** — `^[a-zA-Z_]\w*\s*=\s*(?!.*def |.*class |.*import)` at module level in `tests/**/*.py`
2. **Session-scoped fixtures (Python)** — `@pytest\.fixture\(scope=["']session` in `tests/**/*.py`
3. **Module-scoped fixtures (Python)** — `@pytest\.fixture\(scope=["']module` in `tests/**/*.py`
4. **Class-level setup (Python)** — `@classmethod.*setup|setUpClass|setup_class` in `tests/**/*.py`
5. **Global mocks (TypeScript)** — `vi\.mock\(|jest\.mock\(|vi\.stubGlobal` in `src/**/__tests__/**/*.{ts,tsx}`
6. **beforeAll shared state (TypeScript)** — `beforeAll\(` in `src/**/__tests__/**/*.{ts,tsx}`
7. **Shared variables in describe (TypeScript)** — `let\s+\w+\s*[;=]` at describe scope in `src/**/__tests__/**/*.{ts,tsx}`
8. **Global state mutation (Python)** — `global\s+\w+|app\.state|settings\.` in `tests/**/*.py`
9. **Database state (Python)** — `db\.|database\.|collection\.|insert|create` in `tests/**/*.py` (tests that write to real databases)

**Analysis:**
- Session/module-scoped fixtures share state across tests — if one test mutates, others break
- `vi.mock()` at module level affects all tests in the file — may leak between test files if not restored
- `let` variables at describe scope get mutated by individual tests — order-dependent
- Tests that write to real databases without cleanup create order-dependent state
- `beforeAll` vs `beforeEach` — shared setup can leak state

**Severity mapping:**
- Test writes to real database without cleanup/rollback = HIGH
- Global state mutation in test without restore = HIGH
- Session-scoped fixture that holds mutable state = HIGH
- `vi.mock()` without `vi.restoreAllMocks()` in afterEach = MEDIUM
- Module-scoped fixture with read-only data = LOW
- `beforeAll` with immutable setup = LOW
- Properly isolated tests (function-scoped fixtures, `beforeEach` reset) = no finding (good pattern)

**LOC Impact:** ~2-10 lines per finding (add cleanup, narrow fixture scope, add afterEach)
**Effort:** `trivial` (add afterEach restore) to `medium` (refactor shared fixture to per-test)

### Phase 5: Mock Hygiene

Scan for mocking anti-patterns that reduce test effectiveness.

**Grep patterns (parallel):**

1. **patch at consumer module (Python anti-pattern)** — `patch\(["']api\.routers\.\w+\.` in `tests/**/*.py` (lazy imports break this — codebase-quirks.md)
2. **patch at source module (Python correct)** — `patch\(["']api\.services\.\w+\.|patch\(["']api\.websocket\.\w+\.` in `tests/**/*.py`
3. **Deep mock chains (Python)** — `\.return_value\.return_value|mock\.\w+\.\w+\.\w+` in `tests/**/*.py`
4. **Mock everything (Python)** — count `@patch|patch\(` decorators per test function — flag >5 patches
5. **vi.fn() without implementation (TypeScript)** — `vi\.fn\(\)(?!\.)` in `src/**/__tests__/**/*.{ts,tsx}`
6. **Mock module entirely (TypeScript)** — `vi\.mock\(["'][^"']+["']\)(?!.*factory)` in `src/**/__tests__/**/*.{ts,tsx}` (no factory = auto-mock everything)
7. **Spy without restore (Python)** — `mocker\.spy|MagicMock|Mock\(\)` without corresponding restore in `tests/**/*.py`
8. **Type assertions bypassed** — `as any|as unknown` in `src/**/__tests__/**/*.{ts,tsx}` (type-unsafe mocks)

**Analysis:**
- **Known quirk:** Routers use lazy imports inside functions — `patch("api.routers.system.check_relay_health")` fails. Must patch at source module: `patch("api.routers.health.check_relay_health")` (codebase-quirks.md)
- Over-mocking (>5 patches per test) means the test is testing mocks, not code
- Deep mock chains (`mock.return_value.return_value.method`) are fragile and break on refactoring
- `vi.fn()` without implementation silently returns undefined — may hide real bugs
- Auto-mocking entire modules (`vi.mock('module')`) without factory replaces everything — test may pass for wrong reasons
- `as any` in test mocks bypasses TypeScript's type checking — mock may not match real interface

**Severity mapping:**
- `patch()` at consumer module for lazy-imported dependency (known broken pattern) = HIGH
- Test with >5 `@patch` decorators (over-mocking) = HIGH
- Deep mock chain (>3 levels of `.return_value`) = MEDIUM
- `vi.mock()` without factory on complex module = MEDIUM
- `as any` on mock that should match a specific interface = MEDIUM
- `vi.fn()` returning undefined where return value matters = MEDIUM
- Mock at source module = no finding (correct pattern)
- Focused mocking (1-3 patches) = no finding (good pattern)

**LOC Impact:** ~2-10 lines per finding (fix patch target, reduce mock depth, add factory)
**Effort:** `trivial` (fix patch path) to `medium` (refactor over-mocked test)

### Phase 6: Coverage Gaps (agent-driven)

**Grep patterns (always run):**

1. **Coverage config (Python)** — `\[tool\.coverage\]|coverage|omit|source` in `pyproject.toml`
2. **Coverage config (Vitest)** — `coverage|c8|istanbul|v8` in `vitest.config.ts`
3. **Test file mapping** — For each source directory, check if corresponding test directory exists:
   - `api/routers/` → `tests/test_routers/` or `tests/routers/`
   - `api/services/` → `tests/test_services/` or `tests/services/`
   - `api/websocket/` → `tests/test_websocket/` or `tests/websocket/`
   - `devices/` → `tests/test_devices/` or `tests/devices/`
   - `src/services/` → `src/**/__tests__/`
   - `src/components/` → `src/**/__tests__/`
4. **Untestable patterns** — `Canvas|WebRTC|getUserMedia|createObjectURL|requestAnimationFrame` in `src/**/*.{ts,tsx}` (structural blockers for jsdom testing)
5. **Known failures** — `skip|xfail|xit|xdescribe|pending|todo` in test files across all stacks

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze test coverage strategy for this project:
>
> **Findings:** [Insert coverage config, test file mapping, untestable patterns, known failures]
>
> **Known context:**
> - Python: ~1,373 tests (~84% coverage). Gap: LiveKit integration (requires hardware)
> - Frontend: ~1,037 Vitest + 107 Playwright E2E. 5 pre-existing failures. Structural blocker: 75% of frontend LOC is Canvas/WebRTC/rendering untestable with jsdom
> - Rust: relay 115, signals 321
>
> **Analyze:**
> 1. **Coverage by Feature** — Which features have good coverage? Which are blind spots?
> 2. **Critical Path Coverage** — Are the most important paths (auth, device connection, data pipeline) well tested?
> 3. **Structural Blockers** — What code CAN'T be unit tested (Canvas, WebRTC, hardware)? What alternative strategies exist (visual regression, E2E, integration)?
> 4. **Coverage Strategy** — Where would 10 new tests have the highest impact? Prioritize by risk × coverage gap
> 5. **Pre-existing Failures** — What's blocking the 5 Vitest failures? Effort to fix?
>
> **Return:** Coverage gap matrix (feature × stack × current coverage × risk level × recommended action)"

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- Critical path (auth, data pipeline, device connection) with <50% coverage = HIGH
- Feature directory with zero test files = HIGH
- 5 pre-existing test failures unfixed (regression risk) = HIGH
- Non-critical feature with <50% coverage = MEDIUM
- Structural coverage blocker with no alternative strategy (no E2E, no visual regression) = MEDIUM
- Known skip/xfail without linked issue or timeline = MEDIUM
- Good coverage (>80%) on critical paths = no finding (good pattern)

**LOC Impact:** ~10-100 lines per finding (write new test files)
**Effort:** `small` (add test for uncovered utility) to `large` (design coverage strategy for Canvas/WebRTC)

**On timeout/failure:** Output raw coverage config + test file mapping without agent analysis. Note `Agent: None (timeout)` in header.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_TestQuality_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Test Quality Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_TestQuality_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** pytest, vitest, playwright (if available), Grep (built-in), Read (file analysis)
```

### 6b. Executive Summary

2-3 sentences covering:
- Total test count across stacks, test density, highest-risk quality finding
- Known pre-existing failures status
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Python tests | N | — | — |
| Vitest tests | N | — | — |
| Playwright tests | N | — | — |
| Rust tests | N | — | — |
| Total tests | N | — | — |
| Pre-existing failures | N | — | — |
| Flaky indicators found | N | — | — |
| Over-mocked tests (>5 patches) | N | — | — |
| Tests with sleep() | N | — | — |
| Directories with zero tests | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **TQ-** prefix (TQ-1, TQ-2, TQ-3... for Test Quality findings).

### 6e. Auto-Fixed Section

**Not applicable for test quality audit** — test changes require human review. If `--fix` was passed, note:
```markdown
## Auto-Fixed

N/A — Test quality audit is read-only. Test changes require human review.
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

### 6h. Test Inventory Summary

```markdown
## Test Inventory

| Stack | Directory | Test Count | Source Files | Density | Status |
|-------|-----------|-----------|-------------|---------|--------|
| Python | tests/test_routers/ | N | N | N.N | ? |
| Python | tests/test_services/ | N | N | N.N | ? |
| Vitest | src/**/__tests__/ | N | N | N.N | ? |
| Playwright | e2e/ | N | N | N.N | ? |
| Rust | your-rust-service/tests/ | N | N | N.N | ? |
| Rust | your-rust-crate/src/ | N | N | N.N | ? |
```

### 6i. Coverage Gap Matrix

Include the agent-generated matrix (or stub if `--quick`):
```markdown
## Coverage Gap Matrix

| Feature | Stack | Current Coverage | Risk | Blocker | Recommended Action |
|---------|-------|-----------------|------|---------|-------------------|
| Auth flow | Python | ? | ? | — | ? |
| Device connection | Python | ? | ? | Hardware | ? |
| Data pipeline | Python+Rust | ? | ? | — | ? |
| Canvas rendering | TypeScript | ? | ? | jsdom | ? |
| WebRTC | TypeScript | ? | ? | jsdom | ? |
```

### 6j. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. CRITICAL findings first (if any)
2. HIGH findings (pre-existing failures, zero-test directories, flaky tests on critical path)
3. Quick wins with highest test-quality-to-effort ratio

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
git add work/MMDDYY_TestQuality_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): test quality audit — N findings (X critical, Y high)

Tools: pytest, vitest, Grep, Read
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

Before completing, verify all items from shared.md Execution Checklist are met. Key test quality-specific items:

- [ ] All 6 phases ran (or 5 in quick mode — skip Phase 6 agent) — each phase produced findings or confirmed clean
- [ ] Phase 1: Tests counted by directory/stack, zero-test directories identified
- [ ] Phase 2: Slow tests flagged (>5s unit, >30s integration), sleep() patterns cataloged
- [ ] Phase 3: Flaky indicators found (time-based assertions, random without seed, Date.now() in fixtures, port binding)
- [ ] Phase 4: Shared state patterns assessed (session-scoped fixtures, global mocks, database writes without cleanup)
- [ ] Phase 5: Mock hygiene checked (patch at wrong module per codebase-quirks.md, over-mocking >5 patches, deep mock chains, auto-mock without factory)
- [ ] Phase 6: Coverage gaps identified by feature/stack, structural blockers noted (Canvas/WebRTC/jsdom), strategy recommended (agent in full mode)
- [ ] Severity mapping applied per phase-specific rules above
- [ ] Test quality-specific metrics populated (test counts per stack, flaky indicators, sleep count, over-mocked tests, zero-test directories)
- [ ] Findings numbered sequentially with TQ- prefix (TQ-1, TQ-2, ...)
- [ ] `--fix` noted as N/A (read-only audit)
- [ ] Delta filters to test file patterns and config files
- [ ] Test Inventory Summary and Coverage Gap Matrix included (or stubbed in quick mode)
- [ ] Known codebase quirks referenced: lazy import patch-at-source rule, Playwright Date.now() fixture quirk, 5 pre-existing Vitest failures
- [ ] Test coverage debt from tech-debt-patterns.md cross-referenced in Phase 6

---

## See Also

- `/audit-database` — Database integrity audit (schema, query safety, backups)
- `/audit-docker` — Docker infrastructure audit (container security, resource limits, network isolation)
- `/audit-resilience` — Error handling & resilience audit (error swallowing, timeouts, retry logic)
- `/audit-observability` — Logging & monitoring audit (structlog, alerts, dashboards)
- `/audit-python` — Python code quality audit (radon, vulture, ruff)
- `/audit-typescript` — TypeScript audit (circular deps, dead exports, oxlint)
- `/audit-rust` — Rust audit (clippy, unwrap audit, PyO3 bindings)
- `/eod-docs day1` — Day 1 Code Quality includes Regression Prevention (complementary)
