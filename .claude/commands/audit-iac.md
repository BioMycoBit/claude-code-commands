# /audit-iac

Infrastructure-as-Code audit: Caddy config validation (TLS, headers, reverse proxy), Prometheus scrape config checks, Grafana provisioning validation, LiveKit config checks, and docker-compose.yml baseline drift detection.

**Target:** `Caddyfile`, `prometheus/prometheus.yml`, `grafana/provisioning/**/*.yml`, `livekit/livekit.yaml`, `docker-compose.yml`

---

## Scope Boundary

**This audit covers:** Validation of infrastructure configuration files — TLS settings, security headers, reverse proxy rules, monitoring targets, alerting rules, media server settings, and configuration drift from a known-good baseline. Maps to NIST SP 800-53 CM-2 (Baseline Configuration).

**NOT covered here:** Docker container security (see `/audit-docker`), application-level security (see `/audit-security`), secrets in config files (see `/audit-secrets`).

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
   - `AUTO_FIX` — `false` always (`--fix` is a no-op — infrastructure config changes require manual verification and testing)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for IAC audit (config changes require manual verification)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_IAC_Audit.md 2>/dev/null | sort -r | head -1
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

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) and **Read tool** for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep/Read calls in a **single parallel message** where patterns are independent.

### IAC Delta Logic

In delta mode (`--since last`):
- For each config file: if NOT in CHANGED_FILES → carry forward ALL findings from prior
- If config file IS in CHANGED_FILES → full re-scan of that file's phase
- `docker-compose.yml` drift detection always runs (compares against baseline)

### Phase 1: Caddy Config Validation

Read `Caddyfile` and validate:

**1a. TLS Configuration:**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| TLS enabled | `tls` directive present for each site block | HIGH if missing on public-facing |
| Auto HTTPS | Caddy defaults to auto HTTPS — check for `auto_https off` | CRITICAL if disabled |
| TLS version | `protocols` directive — minimum should be TLS 1.2 | HIGH if TLS 1.0/1.1 allowed |
| HSTS | `Strict-Transport-Security` header present | MEDIUM if missing |

**1b. Security Headers:**

| Header | Expected Value | Severity if Missing |
|--------|---------------|---------------------|
| `X-Content-Type-Options` | `nosniff` | MEDIUM |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | MEDIUM |
| `X-XSS-Protection` | `1; mode=block` | LOW |
| `Content-Security-Policy` | Present (any value) | MEDIUM |
| `Referrer-Policy` | `strict-origin-when-cross-origin` or stricter | LOW |
| `Permissions-Policy` | Present (any value) | LOW |

**1c. Reverse Proxy Rules:**

| Check | Pattern | Severity |
|-------|---------|----------|
| Upstream health checks | `health_uri` or `health_path` in `reverse_proxy` blocks | MEDIUM if missing |
| Timeout configuration | `transport` block with timeouts | LOW if missing |
| Load balancing policy | `lb_policy` for multi-upstream | LOW if missing (single upstream OK) |
| WebSocket upgrade | `@websocket` matcher or header upgrade handling | HIGH if WS endpoints lack upgrade |
| Rate limiting | `rate_limit` directive | LOW if missing (may be handled at app layer) |

**Grep patterns (parallel):**

1. `tls` in `Caddyfile`
2. `header` in `Caddyfile` (security headers)
3. `reverse_proxy` in `Caddyfile`
4. `auto_https` in `Caddyfile`
5. `Strict-Transport-Security` in `Caddyfile`

**LOC Impact:** ~1-5 lines per finding
**Effort:** Adding headers = `trivial`, TLS config = `small`, health checks = `small`

### Phase 2: Prometheus Configuration

Read `prometheus/prometheus.yml` and validate:

**2a. Scrape Targets:**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| All services covered | Each Docker service with metrics endpoint should have a scrape target | MEDIUM if missing |
| Scrape interval | `scrape_interval` — should be ≤30s for critical services | LOW if >60s |
| Scrape timeout | `scrape_timeout` < `scrape_interval` | MEDIUM if timeout >= interval |
| Job naming | Each `job_name` should be descriptive and unique | LOW |

**2b. Expected Scrape Targets:**

Cross-reference with docker-compose.yml services:

| Service | Expected Metrics Path | Required? |
|---------|-----------------------|-----------|
| rust_relay | `/metrics` on :8000 | Yes |
| caddy | `/metrics` (admin API) | Yes |
| grafana | `/metrics` on :3002 | No (self-monitors) |
| prometheus | `/metrics` on :9090 | No (self-scrapes) |
| livekit | `/metrics` on :7880 | Yes |
| node_exporter | `/metrics` on :9100 | Recommended |

**2c. Alerting Rules:**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Alert rules defined | `rule_files:` section present and non-empty | HIGH if no alerting |
| Alertmanager configured | `alerting.alertmanagers` section present | HIGH if alerts defined but no destination |
| Alert severity labels | Each rule has `severity` label | LOW |

**Grep patterns (parallel):**

1. `scrape_interval` in `prometheus/prometheus.yml`
2. `job_name` in `prometheus/prometheus.yml`
3. `rule_files` in `prometheus/prometheus.yml`
4. `alertmanagers` in `prometheus/prometheus.yml`

**LOC Impact:** ~3-10 lines per finding
**Effort:** Adding scrape target = `trivial`, alert rules = `medium`

### Phase 3: Grafana Provisioning

Read all files in `grafana/provisioning/`:

**3a. Datasources (`grafana/provisioning/datasources/`):**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Prometheus datasource | `type: prometheus` present | HIGH if missing |
| URL correct | `url:` points to correct Prometheus endpoint | MEDIUM if wrong |
| Default datasource | `isDefault: true` on one datasource | LOW |
| Access mode | `access: proxy` (server-side, preferred over `direct`) | LOW |

**3b. Dashboards (`grafana/provisioning/dashboards/`):**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Dashboard provisioning | At least one dashboard provider configured | MEDIUM if none |
| Path exists | `path:` points to existing directory | HIGH if path invalid |
| Update interval | `updateIntervalSeconds` set | LOW |

**3c. Alerting (`grafana/provisioning/alerting/`):**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Alert rules provisioned | Alert YAML files present | MEDIUM if none |
| Contact points defined | `contactPoints:` section present | HIGH if alerts but no contacts |
| Notification policies | `policies:` section present | MEDIUM if missing |

Read all provisioning files in parallel.

**LOC Impact:** ~3-10 lines per finding
**Effort:** Adding datasource = `trivial`, dashboards = `small`, alerting = `medium`

### Phase 4: LiveKit Configuration

Read `livekit/livekit.yaml` and validate:

**4a. Room Settings:**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Max participants | `room.max_participants` set | MEDIUM if unlimited |
| Empty timeout | `room.empty_timeout` set | LOW (resource cleanup) |
| Departure timeout | `room.departure_timeout` set | LOW |

**4b. Codec Configuration:**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Video codecs | `video.codecs` specified | LOW if using defaults |
| Audio codecs | `audio.codecs` specified | LOW if using defaults |

**4c. Security:**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| API key defined | `keys:` section with at least one key | CRITICAL if no keys |
| Key rotation | Multiple keys present (allows rotation) | LOW if single key |
| TURN server configured | `turn:` section present | MEDIUM if missing (NAT traversal) |
| TLS on TURN | `turn.tls_port` configured | MEDIUM if TURN without TLS |

**4d. Logging:**

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Log level | `logging.level` set | LOW |
| Structured logging | `logging.json: true` | LOW (recommended for production) |

**Grep patterns (parallel):**

1. `max_participants` in `livekit/livekit.yaml`
2. `keys:` in `livekit/livekit.yaml`
3. `turn:` in `livekit/livekit.yaml`
4. `logging` in `livekit/livekit.yaml`

**LOC Impact:** ~1-5 lines per finding
**Effort:** Config additions = `trivial` to `small`

### Phase 5: docker-compose.yml Baseline Drift

Compare current `docker-compose.yml` against baseline expectations:

**5a. Service Inventory:**

Verify all 15 expected services are present:

```
rust_relay, caddy, redis, umami, umami-db, uptime-kuma, coturn,
livekit, mongodb, glances, infisical, infisical-db, infisical-redis,
prometheus, grafana
```

| Check | Severity |
|-------|----------|
| Expected service missing | HIGH |
| Unexpected new service (not in baseline) | MEDIUM (review needed) |
| All 15 present | no finding |

**5b. Port Exposure Drift:**

For each service, verify published ports match expected values (from CLAUDE.md Docker Services table):

| Service | Expected Port(s) |
|---------|-------------------|
| rust_relay | 8000 |
| caddy | 80, 443 |
| redis | 6379 |
| umami | 3000 |
| umami-db | 5432 |
| uptime-kuma | 3001 |
| coturn | 3478 |
| livekit | 7880 |
| mongodb | 27017 |
| glances | 61208 |
| infisical | 8080 |
| infisical-db | 5433 |
| prometheus | 9090 |
| grafana | 3002 |

| Check | Severity |
|-------|----------|
| Port changed from baseline | MEDIUM (intentional or drift?) |
| New port exposed | MEDIUM |
| Port removed | LOW (verify service is internal-only) |

**5c. Image Version Drift:**

| Check | Severity |
|-------|----------|
| Image tag changed from prior audit | INFO (note for awareness) |
| Image tag `:latest` | MEDIUM (non-reproducible) |

Read `docker-compose.yml` and cross-reference with expected values.

**LOC Impact:** ~1-3 lines per finding
**Effort:** Port/service changes = `trivial` to `small`

### Phase 6: Cross-Config Consistency (agent-driven)

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze cross-config consistency across this project infrastructure files:
>
> **Configs:** Caddyfile, prometheus/prometheus.yml, grafana provisioning, livekit/livekit.yaml, docker-compose.yml
>
> **Analyze:**
> 1. **Service Discovery** — Does Prometheus scrape every service that Caddy reverse-proxies? Are all docker-compose services reachable by their config references?
> 2. **Port Consistency** — Do port numbers in Caddyfile reverse_proxy match docker-compose published ports? Do Prometheus targets match actual service ports?
> 3. **TLS Consistency** — Is TLS configured end-to-end (Caddy→service)? Is TURN TLS aligned with Caddy TLS?
> 4. **Missing Integration** — Any service in docker-compose that has no monitoring (Prometheus) AND no proxy (Caddy) AND no dashboard (Grafana)?
>
> **Return:** Cross-config consistency matrix + specific mismatches found."

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- Port mismatch between configs (Caddy upstream ≠ docker-compose port) = HIGH
- Service with no monitoring AND no proxy = MEDIUM
- TLS inconsistency = MEDIUM
- Minor naming inconsistency = LOW

**On timeout/failure:** Skip cross-config analysis. Note `Agent: None (timeout)` in header.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_IAC_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# IAC Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_IAC_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** Read (config files), Grep (pattern matching)
```

### 6b. Executive Summary

2–3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., TLS disabled, no alerting, missing scrape targets)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Config files scanned | N/5 | — | — |
| Security headers present | N/6 | — | — |
| Prometheus scrape targets | N | — | — |
| Services with monitoring | N/15 | — | — |
| Docker services (expected 15) | N | — | — |
| Port drift from baseline | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **IC-** prefix (IC-1, IC-2, IC-3...).

### 6e. Config Inventory

```markdown
## Config Inventory

| Config File | Exists | Last Modified | Phase |
|-------------|--------|---------------|-------|
| Caddyfile | Yes/No | YYYY-MM-DD | Phase 1 |
| prometheus/prometheus.yml | Yes/No | YYYY-MM-DD | Phase 2 |
| grafana/provisioning/**/*.yml | Yes/No | YYYY-MM-DD | Phase 3 |
| livekit/livekit.yaml | Yes/No | YYYY-MM-DD | Phase 4 |
| docker-compose.yml | Yes/No | YYYY-MM-DD | Phase 5 |
```

### 6f. Auto-Fixed Section

```markdown
## Auto-Fixed

N/A — IAC audit is read-only. Infrastructure config changes require manual verification and testing.
```

### 6g. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6h. Historical Section

Per shared.md format.

### 6i. Recommendations

Top 3 highest-impact recommendations. Prioritize:
1. CRITICAL findings (TLS disabled, no API keys, missing services)
2. HIGH findings (no alerting, port mismatches, missing scrape targets)
3. Cross-config consistency issues (port drift, missing monitoring)

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules — append new CRITICAL/HIGH findings to `tech-debt-patterns.md`, mark resolved entries.

### 7b. Git Commit

```bash
git add work/MMDDYY_IAC_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): iac audit — N findings (X critical, Y high)

Tools: Read, Grep
Mode: Full | Quick
[Delta: +N new, -N resolved]  (only if --since last)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### 7c. Completion Report

Output to user per shared.md Completion Report format.

### 7d. Sprint Chain

Only if `SPRINT_CHAIN` is true — per shared.md Sprint Chain rules.

---

## Severity Mapping

| Severity | Criteria | Example |
|----------|----------|---------|
| CRITICAL | Security bypass, authentication missing | `auto_https off`, LiveKit with no API keys |
| HIGH | Missing security controls, monitoring gaps | No alerting rules, missing scrape targets, expected service absent |
| MEDIUM | Configuration weakness, drift from baseline | Missing security headers, port changes, stale image tags |
| LOW | Best practice deviation, cosmetic | Missing log config, default timeouts, single API key |
| INFO | Informational, configuration note | Image version change, config file last modified date |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key IAC-specific items:

- [ ] Phase 1: Caddyfile checked for TLS, headers, reverse proxy rules
- [ ] Phase 2: Prometheus scrape targets validated against docker-compose services
- [ ] Phase 3: Grafana datasources, dashboards, and alerting provisioning checked
- [ ] Phase 4: LiveKit config checked for room settings, codecs, security, TURN
- [ ] Phase 5: docker-compose.yml service inventory and port exposure validated
- [ ] Phase 6: Cross-config consistency analyzed (agent in full mode)
- [ ] Config inventory table populated with existence and modification dates
- [ ] Findings numbered with IC- prefix
- [ ] `--fix` noted as N/A (config changes require manual verification)
- [ ] Baseline drift detection compared against CLAUDE.md Docker Services table

---

## See Also

- `/audit-docker` — Docker container security (privileges, resource limits, health checks)
- `/audit-security` — Application-level security audit
- `/audit-secrets` — Secret detection in config files and git history
- `/audit-observability` — Observability coverage audit (complementary to Prometheus/Grafana checks)
- `/audit-pipeline` — CI/CD pipeline audit
