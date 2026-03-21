# /audit-pipeline

Real-time data pipeline audit: latency budgets, sample rate validation, buffer management, backpressure handling, clock synchronization, data loss detection, and timer resolution across the full pipeline (Source → Backend → Relay → Frontend).

**Target:** `devices/`, `api/websocket/`, `api/services/`, `your-rust-service/src/`, `your-rust-crate/src/`, `your-rust-crate/src/`, `src/services/`

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
   - `AUTO_FIX` — `false` always (Pipeline audit is read-only, `--fix` is a no-op — report but do not modify any pipeline code)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for Pipeline audit (read-only by design)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Pipeline_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 7 phases (or 5 if `--quick` — skip agent analysis in Phases 1 and 4), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Pipeline-Specific Delta Logic

In delta mode (`--since last`):
- Identify CHANGED_FILES via `git diff --name-only` against prior date
- If no pipeline files (devices/, api/websocket/, api/services/, your-rust-service/, your-rust-crate/, your-rust-crate/, src/services/) changed → carry forward ALL findings from prior
- If pipeline files changed → re-scan only phases whose scan targets include changed files
- Phase mapping for selective re-scan:
  - `devices/` changed → Phase 1 (latency), Phase 2 (sample rate), Phase 7 (timer)
  - `api/websocket/` changed → Phase 1 (latency), Phase 3 (buffer), Phase 4 (backpressure), Phase 5 (clock sync)
  - `api/services/` changed → Phase 3 (buffer), Phase 6 (data loss)
  - `your-rust-service/src/` changed → Phase 1 (latency), Phase 4 (backpressure), Phase 5 (clock sync)
  - `your-rust-crate/src/` changed → Phase 1 (latency), Phase 3 (buffer)
  - `your-rust-crate/src/` changed → Phase 3 (buffer), Phase 6 (data loss)
  - `src/services/` changed → Phase 1 (latency), Phase 4 (backpressure)

### Known Codebase Quirks (read before scanning)

Reference these during analysis — they are confirmed issues documented in `.claude/rules/`:
- **Windows `asyncio.sleep` has ~15ms resolution — check for adaptive tick compensation in simulation/demo code
- **Check for messages dropped before relay/broker is ready — may need retry logic
- **Check for hardcoded data shape assumptions (e.g., fixed channel counts, sample sizes)
- **Check for parameter type mismatches in database queries
- **Use monotonic clocks (`perf_counter_ns`, `Instant`) for latency measurement, not wall clocks

---

### Phase 1: Latency Budget (agent-driven)

Trace pipeline stages from device to display, identify where latency accumulates.

**Grep patterns (parallel):**

1. **perf_counter usage** — `perf_counter` in `backend/`
2. **time.time for latency** — `time\.time\(\)` in `backend/` (should be `perf_counter_ns`, not `time.time`)
3. **Instant/Duration in Rust** — `Instant::now\(\)|\.elapsed\(\)` in `your-rust-service/src/` and `your-rust-crate/src/`
4. **performance.now in frontend** — `performance\.now\(\)` in `src/`
5. **Sleep/delay calls** — `asyncio\.sleep|tokio::time::sleep|setTimeout` across all targets
6. **Latency logging** — `latency|duration_ms|elapsed` across all targets
7. **Pipeline stage timestamps** — `timestamp|ts_ns|created_at` in `api/websocket/`

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze the real-time data pipeline latency budget:
>
> **Context:** Identify the project.s data pipeline stages and throughput requirements from CLAUDE.md. Typical pattern: Source → Backend → Relay → Frontend.
>
> **Findings:** [Insert latency grep results — perf_counter usage, sleep calls, timestamp patterns]
>
> **Analyze:**
> 1. **Stage-by-Stage Budget** — What is the expected latency at each stage? Where is it measured vs assumed?
> 2. **Accumulation Points** — Where do multiple small delays compound (serial awaits, buffering, queue depth)?
> 3. **Measurement Gaps** — Which pipeline stages have NO latency measurement?
> 4. **Coding Standard Violations** — Is `time.time()` used where `perf_counter_ns()` should be?
>
> **Return:** Per-stage latency budget table (stage, expected ms, measured?, measurement method, gap) + accumulation risk assessment."

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- No latency measurement across entire pipeline stage = HIGH
- `time.time()` used for latency measurement instead of `perf_counter_ns()` = MEDIUM
- Sleep/delay in hot path without justification = MEDIUM
- Missing latency logging at stage boundary = LOW
- Latency measured but not logged/reported = LOW

**LOC Impact:** ~1-5 lines per finding (adding measurement, replacing timing function)
**Effort:** Adding `perf_counter_ns` measurement = `trivial`, restructuring serial awaits = `medium`, adding stage boundary instrumentation = `small`

**On timeout/failure:** Output raw grep results without agent analysis. Note `Agent: None (timeout)` in header.

---

### Phase 2: Sample Rate Validation

Verify that expected sample rates for each data source are validated at each pipeline stage.

**Grep patterns (parallel):**

1. **Sample rate constants** — `SAMPLE_RATE|sample_rate|Hz|frequency` in source directories
2. **Data rate config** — `expected_rate|data_rate|sample_rate` in source directories
3. **Rate validation** — `rate|frequency|samples_per_second` in `api/`
4. **Dropped sample detection** — `drop|missed|gap|lost|skip` in `devices/`
5. **Sample counting** — `sample_count|packet_count|sequence` in `backend/`
6. **Adaptive tick** — `adaptive|tick|compensation|calibrat` in `devices/`
7. **Rust sample handling** — `sample|packet|frame` in `your-rust-crate/src/`

**Analysis:**
- For each data source type, verify:
  - Is the expected rate documented/configured?
  - Is the actual received rate measured at the backend?
  - Are rate deviations detected and logged?
  - Is the rate propagated to downstream consumers?
- Check if simulated/demo data sources produce correct sample rates (especially on Windows with ~15ms timer resolution)
- Check for sequence number or gap detection between samples

**Severity mapping:**
- No sample rate validation at any stage = HIGH
- Expected rate hardcoded but never verified at runtime = MEDIUM
- Rate validated at source but not at consumer = MEDIUM
- Dropped sample detection missing entirely = MEDIUM
- Demo device rate not matching real device rate = LOW
- Rate logged but deviation threshold not defined = LOW

**LOC Impact:** ~5-20 lines per finding (rate validation, gap detection)
**Effort:** Adding rate validation = `small`, implementing gap detection = `medium`, fixing demo rate on Windows = `small` (already done via adaptive tick)

---

### Phase 3: Buffer Management

Scan for buffer sizes, queue depths, and overflow behavior across the pipeline.

**Grep patterns (parallel):**

1. **Queue/buffer creation** — `Queue|deque|maxlen|buffer_size|capacity` in `backend/`
2. **Rust buffer/channel** — `channel|mpsc|buffer|VecDeque|capacity|bounded` in `your-rust-service/src/` and `your-rust-crate/src/`
3. **WebSocket send queue** — `send|queue|pending|backlog` in `api/websocket/`
4. **Frontend buffer** — `buffer|queue|array|push|shift|splice` in `src/services/`
5. **Database batch size** — `batch|flush|write_buffer|BATCH_SIZE` in `your-rust-crate/src/`
6. **Overflow handling** — `overflow|full|discard|drop_oldest|maxsize` in `backend/`
7. **Memory limit patterns** — `MAX_|LIMIT_|max_size|threshold` in `api/`

**Analysis:**
- For each buffer/queue found, document: location, size/capacity, what happens when full
- Unbounded queues (no maxlen/capacity) in the hot path = memory leak risk at high throughput
- Check if database batch write accumulates unboundedly before flush
- Check if Rust relay channels are bounded or unbounded
- Verify frontend doesn't accumulate unlimited data points for visualization

**Severity mapping:**
- Unbounded queue in high-frequency hot path (samples accumulate forever) = HIGH
- Buffer overflow behavior is crash/panic (not graceful) = HIGH
- No buffer between pipeline stages (synchronous blocking) = MEDIUM
- Bounded buffer but no overflow logging/metrics = MEDIUM
- Buffer size not tunable (hardcoded magic number) = LOW
- Buffer exists and is properly bounded = no finding

**LOC Impact:** ~3-10 lines per finding (adding maxlen, overflow handler)
**Effort:** Adding bounds to queue = `trivial`, implementing overflow strategy = `small`, restructuring unbounded accumulation = `medium`

---

### Phase 4: Backpressure (agent-driven)

Analyze what happens when a consumer is slower than a producer at each pipeline stage.

**Grep patterns (parallel):**

1. **Slow consumer patterns** — `slow|lag|behind|stall|blocked` in all pipeline targets
2. **WebSocket send failure** — `send.*error|send.*fail|ConnectionClosed|close.*code` in `api/websocket/`
3. **Async queue full** — `QueueFull|queue\.full|put_nowait|try_send` in `backend/`
4. **Rust channel full** — `try_send|TrySendError|Err.*full|would block` in `your-rust-service/src/`
5. **Flow control** — `flow_control|pause|resume|throttle|rate_limit` in all pipeline targets
6. **Client disconnect handling** — `disconnect|remove.*client|cleanup` in `api/websocket/`
7. **Frontend frame skip** — `requestAnimationFrame|skip.*frame|throttle|debounce` in `src/services/`

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze backpressure handling in the data pipeline:
>
> **Context:** High-frequency data flows from source through backend and relay to N consuming clients. If any consumer is slow (e.g., browser tab in background, network congestion), what happens upstream?
>
> **Findings:** [Insert backpressure grep results — queue full handling, send failures, disconnect cleanup]
>
> **Analyze:**
> 1. **Producer-Consumer Mismatch** — At each stage, what happens when the downstream can't keep up?
> 2. **Cascade Risk** — Can a slow client cause the relay to back up, which causes the backend to back up, which stalls device reading?
> 3. **Drop Strategy** — Is data dropped (latest? oldest?), queued (bounded? unbounded?), or does the producer block?
> 4. **Client Isolation** — Can one slow WebSocket client affect other clients' data delivery?
>
> **Return:** Per-stage backpressure behavior table (stage, producer, consumer, when slow: behavior, isolation: yes/no) + cascade risk assessment."

**Skip agent if `--quick` mode.**

**Severity mapping:**
- Slow consumer blocks producer (cascade stall to device level) = CRITICAL
- No client isolation — one slow WS client degrades all clients = HIGH
- Unbounded queue between stages (memory leak under backpressure) = HIGH
- Send failure silently drops data with no logging = MEDIUM
- Backpressure exists but no metrics/alerting = MEDIUM
- Producer blocks on full queue but with timeout = LOW
- Proper backpressure with drop-oldest and logging = no finding

**LOC Impact:** ~5-30 lines per finding (isolation, drop strategy, metrics)
**Effort:** Adding client isolation = `medium`, implementing drop-oldest = `small`, adding backpressure metrics = `small`, restructuring blocking producer = `large`

**On timeout/failure:** Output raw grep results without agent analysis. Note `Agent: None (timeout)` in header.

---

### Phase 5: Clock Synchronization

Assess `clock_sync_ping`/`pong` implementation and cross-device time alignment.

**Grep patterns (parallel):**

1. **Clock sync messages** — `clock_sync|sync_ping|sync_pong` in all pipeline targets
2. **Time offset calculation** — `offset|skew|drift|ntp` in `backend/`
3. **Relay readiness check** — `relay_client|is_ready|connected.*relay` in `api/websocket/`
4. **Retry logic for sync** — `retry|attempt|max_retries` near clock sync in `backend/`
5. **Timestamp alignment** — `align|synchronize|reference_time` in `api/services/`
6. **Frontend clock handling** — `serverTime|timeOffset|clockDiff` in `src/`

**Analysis:**
- Verify `clock_sync_ping`/`pong` round-trip is implemented end-to-end (backend → relay → backend)
- Check for the known issue: Check for messages dropped before relay/broker is ready — may need retry logic — has the retry logic (3s delay, 5 retries) been implemented?
- Assess multi-device time alignment: when multiple data sources stream simultaneously, are their timestamps aligned to a common reference?
- Check if clock drift compensation exists for long-running sessions (hours)

**Severity mapping:**
- No clock synchronization mechanism at all = HIGH
- `clock_sync_ping` sent but no relay readiness check (known bug) = HIGH
- Clock sync implemented but no drift compensation for long sessions = MEDIUM
- Single-device sync works but multi-device alignment missing = MEDIUM
- Sync mechanism exists but offset not applied to stored data = LOW
- Clock sync present with retry and drift compensation = no finding

**LOC Impact:** ~5-20 lines per finding (retry logic, drift compensation)
**Effort:** Adding relay readiness retry = `small` (pattern exists), implementing drift compensation = `medium`, multi-device alignment = `large`

---

### Phase 6: Data Loss Detection

Scan for dropped/discarded sample counters, gap detection, and pipeline health metrics.

**Grep patterns (parallel):**

1. **Drop/discard counters** — `drop.*count|discard.*count|lost.*count|missed.*count` in `backend/`
2. **Prometheus pipeline metrics** — `prometheus|metric|counter|gauge|histogram` near pipeline/sample/buffer keywords in `backend/`
3. **Gap detection** — `gap|discontinuity|sequence.*number|seq_num` in `backend/`
4. **Error counting** — `error_count|fail_count|exception_count` in `api/`
5. **Rust pipeline metrics** — `metrics|counter|stats|telemetry` in `your-rust-service/src/` and `your-rust-crate/src/`
6. **Database write failures** — `write.*fail|insert.*error|batch.*error` in `your-rust-crate/src/`
7. **Frontend data gaps** — `gap|missing|stale|timeout` in `src/services/`

**Analysis:**
- At each pipeline stage, determine: are samples counted in AND counted out?
- Calculate theoretical throughput: 512 samples/sec × N seconds = expected count. Is actual count tracked?
- Check if database writer logs successful vs failed batch inserts
- Check if Prometheus metrics exist for: samples received, samples processed, samples dropped, pipeline latency
- Verify frontend detects data staleness (no new data for >N seconds)

**Severity mapping:**
- No sample counting at any pipeline stage = HIGH
- Data silently dropped with no counter or log = HIGH
- Samples counted at source but not at destination (can't detect loss) = MEDIUM
- Gap detection missing between non-contiguous samples = MEDIUM
- Prometheus metrics exist but no alert rules for pipeline health = LOW
- Drop counters exist but thresholds/alerts not configured = LOW
- Full pipeline health metrics with alerting = no finding

**LOC Impact:** ~5-15 lines per finding (counters, metrics, gap detection)
**Effort:** Adding counters = `trivial`, implementing gap detection = `small`, adding Prometheus metrics = `small`, adding alert rules = `medium`

---

### Phase 7: Timer Resolution

Assess timer resolution issues, especially Windows 15ms limitation vs 1.95ms sample period.

**Grep patterns (parallel):**

1. **asyncio.sleep in device loops** — `asyncio\.sleep` in `devices/`
2. **Sleep values** — `sleep\([0-9.]+\)` in `devices/`
3. **Platform detection** — `platform\.system|sys\.platform|IS_WINDOWS|IS_LINUX` in `devices/`
4. **Adaptive tick patterns** — `adaptive|tick|compensation|samples_per_tick|calibrat` in `devices/`
5. **perf_counter_ns usage** — `perf_counter_ns` in `backend/`
6. **Timer resolution comments** — `15ms|timer.*resolution|windows.*sleep|ProactorEventLoop` in `backend/`
7. **Tokio timer in Rust** — `tokio::time|interval|Duration::from_millis` in `your-rust-service/src/`

**Analysis:**
- Check if demo device simulation accounts for Windows 15ms timer resolution (should use adaptive tick compensation — see `your demo/simulation modules`)
- Verify real device drivers don't depend on `asyncio.sleep` precision for sample timing (BLE delivers on its own clock)
- Check if Rust relay timers are appropriate for sub-millisecond work
- Verify `perf_counter_ns` is used consistently (coding standard) and `time.time()` is not used for latency measurement
- Cross-reference with `.claude/rules/cross-platform-risks.md` — asyncio sleep row

**Severity mapping:**
- `asyncio.sleep` < 15ms in hot loop without adaptive compensation = HIGH (broken on Windows)
- `time.time()` used for latency measurement (should be `perf_counter_ns`) = MEDIUM
- No platform detection in timing-critical code = MEDIUM
- Adaptive tick exists but only for some data source types = MEDIUM
- Timer resolution documented but not compensated = LOW
- Adaptive tick compensation present and tested = no finding

**LOC Impact:** ~5-20 lines per finding (platform check, adaptive tick, timing fix)
**Effort:** Adding platform detection = `trivial`, implementing adaptive tick = `small`, replacing `time.time` = `trivial`

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Pipeline_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Pipeline Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Pipeline_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Read (file analysis) [list any additional tools used]
```

### 6b. Executive Summary

2–3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., unbounded queue in hot path, no backpressure isolation)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Pipeline stages scanned | N/5 | — | — |
| Stages with latency measurement | N/5 | — | — |
| Stages with sample counting | N/5 | — | — |
| Bounded buffers | N/total | — | — |
| Backpressure handling present | N/stages | — | — |
| Clock sync implemented | yes/no | — | — |
| Adaptive tick compensation | N devices | — | — |

Pipeline stages: Device, Python Backend, Rust Relay, Rust Signal Processing, React Frontend.

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **PL-** prefix (PL-1, PL-2, PL-3... for Pipeline findings).

### 6e. Auto-Fixed Section

**Not applicable for Pipeline audit** — this audit is read-only by design. If `--fix` was passed, note:
```markdown
## Auto-Fixed

N/A — Pipeline audit is read-only. Use findings to manually address pipeline issues.
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

### 6h. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. CRITICAL findings first (cascade stall, unbounded queues in hot path)
2. HIGH findings (no sample counting, no backpressure isolation, clock sync bug)
3. Quick wins with highest reliability-to-effort ratio

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
git add work/MMDDYY_Pipeline_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): pipeline audit — N findings (X critical, Y high)

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

Before completing, verify all items from shared.md Execution Checklist are met. Key Pipeline-specific items:

- [ ] All 7 phases ran (or 5 in quick mode) — each phase produced findings or confirmed clean
- [ ] Phase 1: Latency measurement checked at all 5 pipeline stages
- [ ] Phase 2: Sample rate validation checked for all configured data sources
- [ ] Phase 3: All buffers/queues identified with capacity and overflow behavior
- [ ] Phase 4: Backpressure behavior mapped at each stage boundary (agent in full mode)
- [ ] Phase 5: Clock sync implementation verified, relay readiness bug checked
- [ ] Phase 6: Sample counting and data loss detection at each stage
- [ ] Phase 7: Timer resolution issues on Windows assessed, adaptive tick verified
- [ ] Severity mapping applied per phase-specific rules above
- [ ] Pipeline-specific metrics populated (stages with measurement, bounded buffers, etc.)
- [ ] Findings numbered sequentially with PL- prefix (PL-1, PL-2, ...)
- [ ] `--fix` noted as N/A (read-only audit)
- [ ] Known project quirks referenced where applicable (timer resolution, clock sync, data shape assumptions)
- [ ] Scan targets cover all pipeline stages: devices/, api/websocket/, api/services/, your-rust-service/src/, your-rust-crate/src/, your-rust-crate/src/, src/services/
- [ ] Delta uses pipeline file mapping for selective re-scan

---

## See Also

- `/audit-docker` — Docker infrastructure audit (containers, health checks, resource limits)
- `/audit-resilience` — Error handling and resilience audit (timeouts, retries, degradation)
- `/audit-observability` — Observability audit (logging, correlation IDs, alerts, dashboards)
- `/audit-database` — Database audit (schema, query safety, backups)
- `/audit-test-quality` — Test quality audit (flaky tests, isolation, mock hygiene)
- `/audit-python` — Python code quality audit (radon, vulture, ruff)
- `/audit-rust` — Rust audit (clippy, unwrap audit, PyO3 bindings)
- `/eod-docs day2` — Day 2 System Health includes Bottleneck Audit (complementary)
