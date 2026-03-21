# /audit-docker

Docker infrastructure audit: base image security, container privileges, resource limits, health checks, network isolation, volume security, build hygiene, and Trivy container image vulnerability scanning across all 15 Docker services.

**Target:** `docker-compose.yml`, `your-service/Dockerfile`, `**/Dockerfile*`, `.dockerignore`

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
   - `AUTO_FIX` — `false` always (Docker audit is read-only, `--fix` is a no-op — report but do not modify any Docker configuration)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for Docker audit (read-only by design)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Docker_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 8 phases (or 7 if `--quick` — skip agent analysis in Phase 5), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Docker-Specific Delta Logic

In delta mode (`--since last`):
- If `docker-compose.yml` is **NOT** in CHANGED_FILES → carry forward ALL findings from prior (single file = all services)
- If `docker-compose.yml` **IS** in CHANGED_FILES → full re-scan required
- If `your-service/Dockerfile` changed → re-scan Phase 1 and Phase 7 only
- If `.dockerignore` changed → re-scan Phase 7 only

### Phase 1: Base Image Security

Scan all Dockerfiles and `docker-compose.yml` `image:` directives.

**Grep patterns (parallel):**

1. **Dockerfile FROM directives** — `^FROM\s+` in `**/Dockerfile*` files
2. **Unpinned tags** — `FROM.*:latest` or `FROM [^:]+\s` (no tag = implicit latest)
3. **docker-compose image tags** — `image:` in `docker-compose.yml`
4. **Multi-stage build usage** — count `FROM` directives per Dockerfile (>1 = multi-stage ✓)

**Analysis:**
- For each `image:` in docker-compose.yml, check if version is pinned (e.g., `redis:7.2-alpine` ✓, `redis:latest` ✗, `redis` ✗)
- For Dockerfiles, check `FROM` uses pinned digest or specific version tag
- Flag `latest` or absent tags

**Severity mapping:**
- `FROM` with no tag or `:latest` on public-facing service = MEDIUM
- `FROM` with no tag or `:latest` on internal service = LOW
- All images pinned = no finding

**LOC Impact:** ~1 line per finding (the `FROM` or `image:` line)
**Effort:** `trivial` (pin the version tag)

### Phase 2: Container Privileges

Scan `docker-compose.yml` for privilege escalation vectors.

**Grep patterns (parallel):**

1. **User directive** — `user:` in `docker-compose.yml` (presence = good)
2. **Privileged mode** — `privileged:\s*true` in `docker-compose.yml`
3. **Capability additions** — `cap_add:` in `docker-compose.yml`
4. **Security options** — `security_opt:` in `docker-compose.yml`
5. **Read-only root** — `read_only:\s*true` in `docker-compose.yml`
6. **Dockerfile USER** — `^USER\s+` in `**/Dockerfile*`

**Analysis:**
- List all 15 services; for each, note: has `user:` directive? has `privileged`? has `cap_add`?
- Services without `user:` run as root by default inside the container
- Cross-reference with port exposure: root + exposed port = higher severity

**Severity mapping:**
- Running as root (`no user:`) + exposed to host network or published port = HIGH
- `privileged: true` on any service = HIGH
- `cap_add` with dangerous capabilities (`SYS_ADMIN`, `NET_ADMIN`, `SYS_PTRACE`) = HIGH
- Running as root on internal-only service = MEDIUM
- Missing `read_only: true` on stateless service = LOW
- Missing `security_opt: ["no-new-privileges:true"]` = LOW

**LOC Impact:** ~1-5 lines per finding (service block)
**Effort:** Adding `user:` = `small`, removing `privileged` = `small`, adding `security_opt` = `trivial`

### Phase 3: Resource Limits

Scan `docker-compose.yml` for `deploy.resources.limits`.

**Grep patterns (parallel):**

1. **Memory limits** — `memory:` under `resources.limits` context in `docker-compose.yml`
2. **CPU limits** — `cpus:` under `resources.limits` context in `docker-compose.yml`
3. **Deploy section** — `deploy:` in `docker-compose.yml`
4. **Restart policy** — `restart:` in `docker-compose.yml`

**Analysis:**
- For each of the 15 services, determine if memory AND CPU limits are set
- Read `deploy:` blocks to find `resources.limits.memory` and `resources.limits.cpus`
- Services without limits can consume unbounded host resources (OOM risk to other services)
- Note restart policy: `unless-stopped` or `always` without health check = restart loops possible

**Severity mapping:**
- Public-facing service (caddy, livekit, umami) without memory limit = HIGH
- Any service without memory limit = MEDIUM
- Any service without CPU limit = LOW
- `restart: always` without `healthcheck:` = MEDIUM (restart loop risk)

**LOC Impact:** ~3-8 lines per finding (adding deploy block)
**Effort:** Adding resource limits = `small`, tuning values = `medium`

### Phase 4: Health Checks

Scan `docker-compose.yml` for `healthcheck:` directives.

**Grep patterns:**

1. **Healthcheck directive** — `healthcheck:` in `docker-compose.yml`
2. **Healthcheck test** — `test:` under healthcheck context
3. **Healthcheck interval** — `interval:` under healthcheck context
4. **Depends_on condition** — `condition:\s*service_healthy` in `docker-compose.yml`

**Analysis:**
- List all 15 services; for each, note: has `healthcheck:`? what test command? what interval?
- Services depended on by others (`depends_on: condition: service_healthy`) MUST have health checks
- Services with `restart: always` but no health check will restart even when unhealthy (restart loops)

**Severity mapping:**
- Critical service (redis, relay, mongodb) without health check = HIGH
- Public-facing service (caddy, umami, uptime-kuma) without health check = HIGH
- Internal service without health check = MEDIUM
- Health check with interval >60s on critical service = LOW
- Health check present and reasonable = no finding

**LOC Impact:** ~5-10 lines per finding (adding healthcheck block)
**Effort:** Adding health check = `small` (most services have simple HTTP/TCP checks)

### Phase 5: Network Isolation (agent-driven)

**Grep patterns (always run):**

1. **Network definitions** — `networks:` (top-level) in `docker-compose.yml`
2. **Service network membership** — `networks:` (service-level) in `docker-compose.yml`
3. **Host network mode** — `network_mode:\s*host` in `docker-compose.yml`
4. **Published ports** — `ports:` in `docker-compose.yml`
5. **Internal network flag** — `internal:\s*true` in `docker-compose.yml`

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze the Docker network topology from docker-compose.yml:
>
> **Findings:** [Insert network grep results — network definitions, service memberships, host mode, ports]
>
> **Analyze:**
> 1. **Network Segmentation** — Are services grouped by trust level (public/internal/data)?
> 2. **Attack Surface** — Which services are reachable from the host? From each other?
> 3. **Lateral Movement** — If one container is compromised, what can it reach?
> 4. **Port Exposure** — Are any ports exposed that shouldn't be (e.g., database ports on 0.0.0.0)?
>
> **Return:** Per-service access matrix table + segmentation recommendations."

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- `network_mode: host` on any service = HIGH (full host network access)
- Database port (27017, 5432, 6379) bound to 0.0.0.0 = HIGH
- All services on single flat network (no segmentation) = MEDIUM
- Internal service with published port that could be internal-only = MEDIUM
- Missing `internal: true` on data network = LOW

**LOC Impact:** ~5-15 lines per finding (network config restructuring)
**Effort:** Network segmentation = `large`, restricting port binding = `small`, adding `internal: true` = `trivial`

**On timeout/failure:** Output raw network grep results without agent analysis. Note `Agent: None (timeout)` in header.

### Phase 6: Volume Security

Scan `docker-compose.yml` for volume mounts.

**Grep patterns (parallel):**

1. **Volume mounts** — `volumes:` in `docker-compose.yml`
2. **Docker socket** — `docker.sock` in `docker-compose.yml`
3. **Read-write on sensitive paths** — `:rw` in volume mounts (or absence of `:ro`)
4. **Host path mounts** — volumes starting with `/` or `./` (bind mounts vs named volumes)
5. **Named volumes** — top-level `volumes:` definitions

**Analysis:**
- List all volume mounts per service
- Flag `/var/run/docker.sock` mount (container escape vector)
- Flag `:rw` (or no mode, which defaults to `:rw`) on sensitive paths (config files, certs, logs)
- Prefer named volumes over bind mounts for data persistence (portability + security)

**Severity mapping:**
- `/var/run/docker.sock` mounted = CRITICAL (container escape, full Docker API access)
- Secrets/certs directory mounted `:rw` = HIGH
- Config files mounted `:rw` where `:ro` would suffice = MEDIUM
- Bind mount (`./path`) where named volume is better = LOW
- All volumes properly scoped = no finding

**LOC Impact:** ~1-2 lines per finding (change `:rw` to `:ro`, remove socket mount)
**Effort:** `:rw` → `:ro` = `trivial`, removing socket mount = `small` (may need alternative), named volume migration = `medium`

### Phase 7: Build Hygiene

Scan Dockerfiles and `.dockerignore` for build context issues.

**Grep patterns (parallel):**

1. **Dockerignore exists** — check for `.dockerignore` file presence
2. **Env files excluded** — `.env` in `.dockerignore`
3. **Key files excluded** — `*.pem`, `*.key` in `.dockerignore`
4. **Node modules excluded** — `node_modules` in `.dockerignore`
5. **Venv excluded** — `.venv`, `venv` in `.dockerignore`
6. **Git excluded** — `.git` in `.dockerignore`
7. **Multi-stage build** — count `FROM` lines per Dockerfile (>1 = multi-stage ✓)
8. **COPY vs ADD** — `^ADD\s+` in Dockerfiles (prefer COPY unless extracting tar)

**Analysis:**
- Check `.dockerignore` covers: `.env`, `*.pem`, `*.key`, `node_modules`, `.venv`, `.git`, `work/`, `*.log`
- Check Dockerfiles use `COPY` not `ADD` (unless extracting archive)
- Note if multi-stage builds are used (reduces image size, excludes build tools from runtime)

**Severity mapping:**
- `.env` or `*.pem`/`*.key` NOT in `.dockerignore` = HIGH (secrets leak into image layer)
- No `.dockerignore` file at all = HIGH
- Missing `node_modules` or `.venv` in `.dockerignore` = MEDIUM (bloated image)
- Missing `.git` in `.dockerignore` = MEDIUM (history leak + bloat)
- Using `ADD` instead of `COPY` = LOW
- Large image without multi-stage build = LOW

**LOC Impact:** ~1-3 lines per finding
**Effort:** Adding `.dockerignore` entries = `trivial`, converting to multi-stage build = `large`

### Phase 8: Trivy Container Image Scanning

Scan Docker images for OS-level and application vulnerabilities using Trivy.

**Tool check:**

```bash
trivy --version 2>&1
```

If Trivy is not installed:
```
Tool not installed: trivy
Install: curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
Alternative: brew install trivy (Mac) | sudo apt-get install trivy (Debian/Ubuntu)
Docker: docker run --rm aquasec/trivy image <image_name>
```
Note in output under Tools and skip Phase 8 entirely.

**8a. Scan Built Images:**

Scan locally-built images (from Dockerfiles in the repo):

```bash
# Scan the rust relay image (the primary custom-built image)
trivy image --severity HIGH,CRITICAL --format json your-image:latest 2>&1
```

If the image doesn't exist locally, note it and skip:
```
Image your-image:latest not found locally. Build with: docker-compose build rust_relay
```

**8b. Scan Base Images:**

Extract base images from Dockerfiles and docker-compose.yml, then scan:

```bash
# Example: scan base images referenced in FROM directives
trivy image --severity HIGH,CRITICAL --format json alpine:3.19 2>&1
trivy image --severity HIGH,CRITICAL --format json python:3.14-slim 2>&1
```

Parse FROM directives from Phase 1 results. Scan up to 5 unique base images. Skip images that require authentication or are unavailable.

**8c. Parse Trivy Output:**

For JSON output, extract:
- `Results[].Vulnerabilities[]` — each with `.VulnerabilityID`, `.Severity`, `.PkgName`, `.InstalledVersion`, `.FixedVersion`
- Count CRITICAL and HIGH vulnerabilities
- Note if `.FixedVersion` exists (fixable vs unfixable)

**Severity mapping:**

- CRITICAL CVE with fix available = CRITICAL (update base image or package)
- HIGH CVE with fix available = HIGH
- CRITICAL CVE without fix = HIGH (awareness, no immediate action)
- HIGH CVE without fix = MEDIUM (awareness)
- MEDIUM/LOW CVEs = INFO (count only, don't list individually)

**Output table:**

```markdown
### Trivy Image Scan

| Image | Critical | High | Medium | Low | Fixable |
|-------|----------|------|--------|-----|---------|
| your-image:latest | N | N | N | N | N/N |
| alpine:3.19 | N | N | N | N | N/N |
```

**LOC Impact:** ~1 line per finding (update base image tag or install package update)
**Effort:** Updating base image = `small`, patching specific package = `medium`

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Docker_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Docker Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Docker_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Read (file analysis), Trivy (container image scanning) [note if Trivy unavailable]
```

### 6b. Executive Summary

2–3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., docker.sock mount, running as root)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Services scanned | N/15 | — | — |
| Services with health checks | N/15 | — | — |
| Services with resource limits | N/15 | — | — |
| Services running as root | N/15 | — | — |
| Docker socket mounts | N | — | — |
| Trivy image CVEs (critical) | N | — | — |
| Trivy image CVEs (high) | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **D-** prefix (D-1, D-2, D-3... for Docker findings).

### 6e. Auto-Fixed Section

**Not applicable for Docker audit** — this audit is read-only by design. If `--fix` was passed, note:
```markdown
## Auto-Fixed

N/A — Docker audit is read-only. Use findings to manually update docker-compose.yml and Dockerfiles.
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
1. CRITICAL findings first (docker.sock, secrets in image)
2. HIGH findings (root + exposed, no resource limits on public service, no health check on critical service)
3. Quick wins with highest security-to-effort ratio

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
git add work/MMDDYY_Docker_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): docker audit — N findings (X critical, Y high)

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

Before completing, verify all items from shared.md Execution Checklist are met. Key Docker-specific items:

- [ ] All 8 phases ran (or 7 in quick mode) — each phase produced findings or confirmed clean
- [ ] Phase 1: Every `image:` and `FROM` directive checked for pinned versions
- [ ] Phase 2: All 15 services checked for `user:` directive, `privileged`, `cap_add`
- [ ] Phase 3: All 15 services checked for `deploy.resources.limits` (memory + CPU)
- [ ] Phase 4: All 15 services checked for `healthcheck:` presence and adequacy
- [ ] Phase 5: Network topology mapped, segmentation assessed (agent in full mode)
- [ ] Phase 6: All volume mounts checked, docker.sock flagged if present
- [ ] Phase 7: `.dockerignore` checked for secrets exclusion, build patterns reviewed
- [ ] Phase 8: Trivy image scan ran (or install command noted if unavailable)
- [ ] Severity mapping applied per phase-specific rules above
- [ ] Docker-specific metrics populated (services with health checks, resource limits, root)
- [ ] Findings numbered sequentially with D- prefix (D-1, D-2, ...)
- [ ] `--fix` noted as N/A (read-only audit)
- [ ] Delta uses docker-compose.yml single-file logic (changed = full rescan, unchanged = carry all)

---

## See Also

- `/audit-python` — Python code quality audit (radon, vulture, ruff)
- `/audit-typescript` — TypeScript audit (circular deps, dead exports, oxlint)
- `/audit-rust` — Rust audit (clippy, unwrap audit, PyO3 bindings)
- `/audit-security` — OWASP security audit (overlaps with privilege/network checks)
- `/eod-docs day2` — Day 2 System Health includes Bottleneck Audit (complementary)
