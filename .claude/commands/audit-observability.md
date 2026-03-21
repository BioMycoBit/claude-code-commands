# /audit-observability

Observability audit: structured logging consistency, correlation IDs, alert coverage, dashboard completeness, and PII in logs across all 15 Docker services and 3 language stacks (Python, TypeScript, Rust).

**Target:** `backend/`, `src/`, `your-rust-service/src/`, `docker-compose.yml`, Prometheus/Grafana config

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
   - `AUTO_FIX` — `false` always (observability audit is read-only, `--fix` is a no-op — report but do not modify any logging or monitoring configuration)
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for observability audit (read-only by design)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Observability_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 5 phases (or 3 if `--quick` — skip agent analysis in Phase 3 and Phase 4), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Observability-Specific Delta Logic

In delta mode (`--since last`):
- Filter CHANGED_FILES to `.py`, `.ts`, `.tsx`, `.rs`, `.yml`, `.yaml`, `.json` extensions
- Re-scan only changed files for Phases 1-2 and Phase 5 (grep-based)
- Agent phases (3, 4) always run full scope in full mode (alert/dashboard gaps don't localize to single files)
- Carry forward findings from unchanged files
- If `docker-compose.yml` changed → re-scan Phases 3-4 (monitoring config may have changed)
- If Prometheus/Grafana config files changed → re-scan Phases 3-4

### Phase 1: Structured Logging Consistency

Scan Python, TypeScript, and Rust for logging patterns and consistency.

**Grep patterns (parallel):**

1. **structlog usage (Python)** — `structlog|get_logger|bind_contextvars` in `**/*.py`
2. **stdlib logging (Python)** — `import logging|logging\.getLogger` in `**/*.py`
3. **print statements (Python)** — `\bprint\(` in `**/*.py` (excluding test files)
4. **Log level usage (Python)** — `logger\.(debug|info|warning|error|critical)\(` in `**/*.py`
5. **structlog event= keys (Python)** — `event=` in `**/*.py`
6. **console.log (TypeScript)** — `console\.(log|warn|error|info|debug)\(` in `src/**/*.{ts,tsx}`
7. **Structured frontend logging** — `logger\.|log\.(debug|info|warn|error)` in `src/**/*.{ts,tsx}`
8. **tracing crate (Rust)** — `tracing::|info!|warn!|error!|debug!|trace!` in `your-rust-service/src/**/*.rs`
9. **println/eprintln (Rust)** — `println!|eprintln!` in `your-rust-service/src/**/*.rs`

**Analysis:**
- All Python backend services should use structlog, not stdlib `logging` or `print()`
- TypeScript should use structured logging, not raw `console.log` in production paths
- Rust relay should use `tracing` crate, not `println!`/`eprintln!`
- Check for consistent `event=` key naming convention in structlog calls
- Check log level appropriateness: business logic at INFO, debugging at DEBUG, failures at ERROR

**Severity mapping:**
- `print()` in production hot path (WS handlers, device callbacks, signal processing) = HIGH
- `println!`/`eprintln!` in Rust relay hot path = HIGH
- Inconsistent logging libraries within same service (mixed structlog + stdlib) = MEDIUM
- `console.log` in production frontend code (not behind debug flag) = MEDIUM
- Inappropriate log level (debug info at ERROR, errors at INFO) = MEDIUM
- Minor `print()` in startup/config code = LOW
- Consistent structlog usage = no finding (good pattern)

**LOC Impact:** ~1-3 lines per finding (replace print/console.log with structured call)
**Effort:** `trivial` (replace print with logger) to `small` (add structlog to service without it)

### Phase 2: Correlation IDs

Scan for request/session/trace IDs that enable end-to-end tracing across the stack.

**Grep patterns (parallel):**

1. **Request ID (Python)** — `request_id|req_id|trace_id|correlation_id|x-request-id` in `**/*.py`
2. **Session ID in logs (Python)** — `session_id.*log|log.*session_id|bind.*session` in `**/*.py`
3. **Device ID in logs (Python)** — `device_id.*log|log.*device_id` in `**/*.py`
4. **Middleware correlation (Python)** — `middleware|Middleware` in `**/*.py`
5. **Frontend trace ID (TypeScript)** — `traceId|trace_id|requestId|request_id|correlationId` in `src/**/*.{ts,tsx}`
6. **WS message ID (TypeScript)** — `messageId|message_id|msg_id` in `src/**/*.{ts,tsx}`
7. **Relay trace (Rust)** — `trace_id|request_id|session_id|correlation` in `your-rust-service/src/**/*.rs`
8. **Span context (Rust)** — `span|instrument|tracing::span` in `your-rust-service/src/**/*.rs`

**Analysis:**
- Can a single request be traced from frontend → backend → relay → device?
- Are session IDs included in log context via `bind_contextvars` or equivalent?
- Are WebSocket messages tagged with IDs that correlate across services?
- Is there middleware that injects correlation IDs on incoming requests?

**Severity mapping:**
- No correlation ID mechanism at all (cannot trace requests across services) = HIGH
- Correlation ID exists but not propagated to all services = MEDIUM
- Session ID in logs but no request-level tracing = MEDIUM
- Missing correlation in WS messages (harder to debug real-time pipeline) = MEDIUM
- Full end-to-end tracing implemented = no finding (good pattern)

**LOC Impact:** ~5-20 lines per finding (add middleware, propagate IDs)
**Effort:** `small` (add ID to existing log calls) to `medium` (implement correlation middleware)

### Phase 3: Alert Coverage (agent-driven)

Scan Prometheus alert rules and compare against available metrics.

**Grep patterns (always run):**

1. **Prometheus config** — `prometheus` in `docker-compose.yml`
2. **Alert rules files** — Glob for `*.rules`, `*.rules.yml`, `alert*.yml` in repo
3. **Prometheus scrape targets** — `scrape_configs|targets|metrics_path` in any `.yml` file under project root
4. **Application metrics (Python)** — `prometheus_client|Counter|Gauge|Histogram|Summary` in `**/*.py`
5. **Metrics endpoints** — `/metrics|metrics_port` in `**/*.py`
6. **Grafana alerting** — `alert|threshold|notification` in any Grafana JSON dashboard files

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze Prometheus alert coverage for this project:
>
> **Findings:** [Insert alert rule grep results, scrape targets, application metrics]
>
> **Analyze:**
> 1. **Metric Inventory** — What metrics are exposed by each service? (backend, relay, Redis, databases, message brokers)
> 2. **Alert Coverage** — Which failure modes have alert rules? Which are unmonitored?
> 3. **Missing Alerts** — Identify critical failure modes with no alert: service down, high latency, error rate spike, disk full, memory exhaustion, connection pool exhaustion
> 4. **Alert Quality** — Are thresholds reasonable? Are there runbook links? Is there alert fatigue risk (too many low-severity alerts)?
>
> **Return:** Coverage matrix (service × failure mode × has alert? × recommended threshold)"

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- No alert rules at all (Prometheus running but no alerts configured) = CRITICAL
- Critical service (relay, Redis, backend) has no down/error alert = HIGH
- Alert rules exist but missing key failure modes (disk, memory, error rate) = HIGH
- Alert thresholds unreasonable (e.g., CPU alert at 99% = useless) = MEDIUM
- Missing alert for non-critical service = MEDIUM
- Alert without runbook or description = LOW
- Comprehensive alert coverage = no finding (good pattern)

**LOC Impact:** ~5-15 lines per finding (add alert rule)
**Effort:** `small` (add single alert rule) to `medium` (design alert strategy for service)

**On timeout/failure:** Output raw metric/alert grep results without agent analysis. Note `Agent: None (timeout)` in header.

### Phase 4: Dashboard Completeness (agent-driven)

Scan Grafana configuration for dashboard coverage of all 15 Docker services.

**Grep patterns (always run):**

1. **Grafana config** — `grafana` in `docker-compose.yml`
2. **Dashboard JSON files** — Glob for `*.json` in any `grafana/` or `dashboards/` directory
3. **Dashboard provisioning** — `provisioning|datasources|dashboards` in Grafana config directories
4. **Panel targets** — `"expr":|"query":` in dashboard JSON files (Prometheus queries)
5. **Container metrics** — `container_|docker_` in any dashboard or alert config

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze Grafana dashboard completeness for this project's 15 Docker services:
>
> **Findings:** [Insert dashboard grep results, panel queries, container metric usage]
>
> **15 services to verify coverage for:**
> rust_relay, caddy, redis, umami, umami-db, uptime-kuma, coturn, livekit, mongodb, glances, infisical, infisical-db, infisical-redis, prometheus, grafana
>
> **Analyze:**
> 1. **Service Coverage** — Which of the 15 services have dashboard panels? Which are invisible?
> 2. **Metric Depth** — For monitored services, are panels showing: uptime, CPU, memory, error rate, request rate?
> 3. **Missing Dashboards** — Which services need new dashboards? Which need additional panels?
> 4. **Dashboard Organization** — Are dashboards logically grouped? Is there an overview/home dashboard?
>
> **Return:** Coverage matrix (service × has dashboard × metrics shown × gaps)"

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- No Grafana dashboards at all (Grafana running but empty) = HIGH
- Critical service (relay, Redis, backend) has no dashboard = HIGH
- Dashboard exists but missing key panels (no error rate, no latency) = MEDIUM
- Non-critical service (umami, glances) missing dashboard = LOW
- No overview/home dashboard linking to all services = LOW
- Comprehensive dashboard coverage = no finding (good pattern)

**LOC Impact:** ~10-50 lines per finding (add dashboard panel JSON)
**Effort:** `small` (add panel to existing dashboard) to `medium` (create new service dashboard)

**On timeout/failure:** Output raw dashboard grep results without agent analysis. Note `Agent: None (timeout)` in header.

### Phase 5: PII in Logs

Scan log statements for personally identifiable information patterns.

**Grep patterns (parallel):**

1. **Email in logs (Python)** — `log.*email|email.*log|logger.*email` in `**/*.py`
2. **IP address in logs (Python)** — `log.*ip_addr|log.*remote_addr|log.*client_ip|log.*X-Forwarded` in `**/*.py`
3. **User ID in logs (Python)** — `log.*user_id|logger.*user_id|log.*username` in `**/*.py`
4. **Device ID in logs (Python)** — `log.*device_id|log.*mac_addr|log.*bluetooth` in `**/*.py`
5. **Token/secret in logs (Python)** — `log.*token|log.*password|log.*secret|log.*api_key` in `**/*.py`
6. **PII in frontend logs (TypeScript)** — `console\..*(email|password|token|userId|user_id)` in `src/**/*.{ts,tsx}`
7. **PII in relay logs (Rust)** — `(info!|warn!|error!|debug!).*(email|token|password|ip_addr|user_id)` in `your-rust-service/src/**/*.rs`
8. **Sensitive domain data in logs** — search for domain-specific field names (e.g., health data, financial data, PII) in `**/*.py`

**Analysis:**
- PII (email, IP, user ID) should be masked or excluded from logs
- Tokens and secrets must NEVER appear in log output
- Device MAC addresses are PII under GDPR
- Special category data (Art. 9) — health, biometric, genetic data — should not appear in plaintext logs
- Cross-reference with `/audit-security` PII findings

**Severity mapping:**
- Token, password, or API key logged in plaintext = CRITICAL
- Email or IP address logged without masking = HIGH
- Biometric data values logged in plaintext (GDPR Art. 9) = HIGH
- User ID logged in high-volume path (enables tracking via log access) = MEDIUM
- Device ID (MAC address) logged without masking = MEDIUM
- PII in debug-level logs only (not in production log level) = LOW
- All PII properly masked or excluded = no finding (good pattern)

**LOC Impact:** ~1-3 lines per finding (mask or remove PII from log call)
**Effort:** `trivial` (remove PII field from log) to `small` (add masking utility)

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Observability_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Observability Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Observability_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Read (file analysis) [list any additional tools used]
```

### 6b. Executive Summary

2-3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., no alert rules, PII in logs, missing correlation IDs)
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
| Services using structlog | N/15 | — | — |
| Services with correlation IDs | N | — | — |
| Prometheus alert rules | N | — | — |
| Grafana dashboards | N/15 services | — | — |
| PII patterns in logs | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **O-** prefix (O-1, O-2, O-3... for Observability findings).

### 6e. Auto-Fixed Section

**Not applicable for observability audit** — this audit is read-only by design. If `--fix` was passed, note:
```markdown
## Auto-Fixed

N/A — Observability audit is read-only. Use findings to manually improve logging and monitoring.
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

### 6h. Alert Coverage Matrix

Include the agent-generated matrix (or stub if `--quick`):
```markdown
## Alert Coverage Matrix

| Service | Down Alert | Error Rate | Latency | Resource | Status |
|---------|-----------|------------|---------|----------|--------|
| rust_relay | ? | ? | ? | ? | ? |
| redis | ? | ? | ? | ? | ? |
| caddy | ? | ? | ? | ? | ? |
| ... | ... | ... | ... | ... | ... |
```

### 6i. Dashboard Coverage Matrix

Include the agent-generated matrix (or stub if `--quick`):
```markdown
## Dashboard Coverage Matrix

| Service | Has Dashboard | CPU | Memory | Error Rate | Custom Metrics | Status |
|---------|--------------|-----|--------|------------|----------------|--------|
| rust_relay | ? | ? | ? | ? | ? | ? |
| redis | ? | ? | ? | ? | ? | ? |
| ... | ... | ... | ... | ... | ... | ... |
```

### 6j. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. CRITICAL findings first (PII/secrets in logs, no alert rules)
2. HIGH findings (missing correlation IDs, no dashboards for critical services, unmasked PII)
3. Quick wins with highest observability-to-effort ratio

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
git add work/MMDDYY_Observability_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): observability audit — N findings (X critical, Y high)

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

Before completing, verify all items from shared.md Execution Checklist are met. Key observability-specific items:

- [ ] All 5 phases ran (or 3 in quick mode — skip Phases 3 and 4) — each phase produced findings or confirmed clean
- [ ] Phase 1: All Python files checked for structlog vs print/logging, all TS files for console.log, all Rust for println
- [ ] Phase 2: Correlation ID mechanism assessed across all 3 stacks (Python, TypeScript, Rust)
- [ ] Phase 3: Prometheus alert rules inventoried, coverage gaps identified (agent in full mode)
- [ ] Phase 4: Grafana dashboards inventoried vs all 15 Docker services (agent in full mode)
- [ ] Phase 5: All log statements scanned for PII patterns (email, IP, token, sensitive domain data)
- [ ] Severity mapping applied per phase-specific rules above
- [ ] Observability-specific metrics populated (structlog adoption, correlation IDs, alert count, dashboard coverage, PII count)
- [ ] Findings numbered sequentially with O- prefix (O-1, O-2, ...)
- [ ] `--fix` noted as N/A (read-only audit)
- [ ] Delta filters to relevant file extensions (.py, .ts, .tsx, .rs, .yml, .yaml, .json)
- [ ] Alert Coverage Matrix and Dashboard Coverage Matrix included (or stubbed in quick mode)
- [ ] Cross-stack scope verified: Python + TypeScript + Rust + Docker/Prometheus/Grafana config all scanned

---

## See Also

- `/audit-docker` — Docker infrastructure audit (container security, resource limits, network isolation)
- `/audit-resilience` — Error handling & resilience audit (error swallowing, timeouts, retry logic)
- `/audit-python` — Python code quality audit (radon, vulture, ruff)
- `/audit-typescript` — TypeScript audit (circular deps, dead exports, oxlint)
- `/audit-rust` — Rust audit (clippy, unwrap audit, PyO3 bindings)
- `/audit-security` — OWASP security audit (overlaps with PII-in-logs checks)
- `/eod-docs day2` — Day 2 System Health includes Bottleneck Audit (complementary)
