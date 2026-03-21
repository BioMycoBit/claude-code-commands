# /audit-api-surface

API surface mapping audit: endpoint-to-frontend tracing, orphan endpoint detection, WebSocket message flow mapping, deprecation candidates, and contract quality analysis (missing `response_model` in FastAPI, `unknown` types in generated TypeScript SDK). Includes Explore agent for consumer mapping and Plan agent for API maintenance planning.

**Target:** `api/routers/**/*.py`, `src/services/**/*.ts`, `api/websocket/**/*.py`, `src/generated/api/types.gen.ts`

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
   - `AUTO_FIX` — `false` always (`--fix` is a no-op — API changes require design decisions and consumer coordination)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for API surface audit (API changes require design decisions)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_API_Surface.md 2>/dev/null | sort -r | head -1
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

Run all phases (skip agents in Phases 2 and 4 if `--quick`), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### API Surface Delta Logic

In delta mode (`--since last`):
1. Load PRIOR_BASELINE for this audit (endpoint map, consumer counts)
2. CARRY FORWARD: Endpoint findings where router file NOT in CHANGED_FILES
3. RE-SCAN: Grep patterns below ONLY on changed router/service files
4. VERIFY: Spot-check 3-5 carried endpoint findings still valid
5. MERGE + MARK: NEW | CARRIED | CHANGED | RESOLVED

If MODE == "full": Run all patterns on entire codebase.

### Phase 1: Grep-Based Analysis

**Grep patterns (parallel):**

| Category | Pattern | Files | Tool |
|----------|---------|-------|------|
| REST endpoints | `@app.(get\|post\|put\|delete\|patch)` | `*.py` | Grep |
| Router endpoints | `@router.(get\|post\|put\|delete\|patch)` | `*.py` | Grep |
| WebSocket messages | Message type constants in code | `*.py`, `*.ts` | Grep |
| External calls | `fetch\|axios\|httpx` | `*.ts`, `*.tsx`, `*.py` | Grep |

**Analysis:**
1. Grep for FastAPI route decorators
2. List all WebSocket message types from CLAUDE.md and code
3. Document external API calls
4. Note any undocumented endpoints

### Phase 2: Explore Agent (if AGENT_MODE = true)

Launch Explore agent with `subagent_type=Explore`.

**Delta mode:** Use Agent Scope Narrowing from shared.md — pass prior endpoint map + CHANGED_FILES list instead of full exploration prompt.

**Full mode prompt:**
> "Map API surface and consumers in routers/, frontend services/, WebSocket handlers:
>
> 1. **Endpoint-to-Frontend Mapping** — Trace fetch/axios calls to REST endpoints, map WS handlers to senders
> 2. **Orphan Detection** — Endpoints never called by frontend (cross-reference routes vs frontend calls)
> 3. **WebSocket Message Flow** — Map message types: backend sender -> frontend subscriber
> 4. **Deprecation Candidates** — Old/unused endpoints (git blame, versioned paths)
>
> **Return:** API surface map with consumer counts per endpoint."

**Use agent output to:** Add consumer counts, mark orphan endpoints, group WS messages by flow.

**On timeout/failure:** Continue with grep patterns from Phase 1 without consumer mapping. Note `Agent: Plan only (Explore timeout)` in header if Plan succeeds later.

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

### Phase 3: Contract Quality Analysis

Checks that the backend's OpenAPI spec produces useful TypeScript types — not `unknown`.

**Why:** FastAPI endpoints without `response_model` emit empty response schemas in OpenAPI. The TypeScript SDK generator (`@hey-api/openapi-ts`) emits `200: unknown` for these. The generated types exist and import cleanly, so dead code and lint audits don't flag them — but they provide zero type safety.

#### Python Side: Missing `response_model`

**Pattern:** Grep FastAPI route decorators and check for `response_model=` presence.

**Grep patterns (parallel):**

| Search | Pattern | Files |
|--------|---------|-------|
| All route decorators | `@router\.(get\|post\|put\|delete\|patch)` | `api/routers/**/*.py` |
| Route decorators WITH response_model | `@router\.(get\|post\|put\|delete\|patch).*response_model` | `api/routers/**/*.py` |

**Output table:**

| Metric | Count |
|--------|-------|
| Total REST endpoints | N |
| With `response_model` (typed) | N |
| Without `response_model` (untyped) | N |
| **Coverage** | **N%** |

List untyped endpoints by router file. Severity: HIGH if coverage < 80%, MEDIUM if < 95%, LOW if >= 95%.

**Known exceptions (skip-list):** Endpoints that intentionally lack `response_model`:
- `/metrics` — Prometheus text format, not JSON
- `/export` — CSV file download, not JSON
- Webhook endpoints — server-to-server, no frontend consumer

#### TypeScript Side: `unknown` in Generated Types

**Pattern:** Count `unknown` response types in the generated SDK.

**Grep patterns:**

| Search | Pattern | Files |
|--------|---------|-------|
| Unknown responses | `200: unknown` | `src/generated/api/types.gen.ts` |
| Total response types | `200:` | `src/generated/api/types.gen.ts` |

**Output table:**

| Metric | Count |
|--------|-------|
| Total endpoint response types | N |
| Concrete types | N |
| `unknown` types | N |
| **Type coverage** | **N%** |

Severity: HIGH if unknown > 10%, MEDIUM if > 0% (excluding skip-list), LOW if 0%.

**Delta mode:** Compare typed/untyped counts vs prior. Flag regressions (new untyped endpoints).

**Cross-reference:** Untyped Python endpoints should match `unknown` TypeScript types. If they don't match, either the generated types are stale or the skip-list needs updating.

### Phase 4: Plan Agent (if AGENT_MODE = true)

Launch Plan agent with `subagent_type=Plan` AFTER grep patterns and contract quality analysis complete.

**Delta mode:** Use Agent Scope Narrowing from shared.md — pass prior maintenance plan + CHANGED_FILES list instead of full exploration prompt.

**Full mode prompt:**
> "Create API maintenance plan:
>
> **Findings:** [Insert grep results + Explore agent output + contract quality results here]
>
> **Analyze:**
> 1. **Documentation Gaps** — Which endpoints lack docs? Prioritize public endpoints
> 2. **Contract Quality** — Untyped endpoints (missing `response_model`) by router, prioritized by frontend consumer count. Endpoints with frontend consumers but `unknown` types = HIGH priority
> 3. **Deprecation Timeline** — Immediate removal (0 consumers), soft deprecation (1-2), keep (core)
> 4. **Versioning** — Primary version, migration paths for old versions
> 5. **Breaking Change Impact** — Order by consumer count (high -> low)
>
> **Return:** Documentation priority table (Endpoint | Method | Consumers | Priority), contract quality table (Router | Typed | Untyped | Coverage%), phased deprecation plan, breaking changes with migration paths."

**Use agent output to:** Add documentation priorities, contract quality remediation priorities, phased deprecation plan, and breaking change migration paths.

**On timeout/failure:** Output raw grep findings with Explore context if available. Note `Agent: Explore only (Plan timeout)` in header.

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_API_Surface.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# API Surface Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_API_Surface.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Explore + Plan | Explore only (Plan timeout) | Plan only (Explore timeout) | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Read (file analysis)
```

### 6b. Executive Summary

2-3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., orphan endpoints, low contract quality coverage)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| REST endpoints | N | — | — |
| WebSocket message types | N | — | — |
| Orphan endpoints (0 consumers) | N | — | — |
| Endpoints with `response_model` | N | — | — |
| Contract quality coverage | N% | — | — |
| `unknown` TypeScript types | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **AS-** prefix (AS-1, AS-2, AS-3... for API Surface findings).

### 6e. API Surface Map

```markdown
## API Surface Map

### REST Endpoints

| Router | Method | Path | Consumers | response_model | Status |
|--------|--------|------|-----------|----------------|--------|
| auth | POST | /api/auth/login | 2 | LoginResponse | ... |

### WebSocket Message Types

| Type | Sender | Receiver | Consumers | Status |
|------|--------|----------|-----------|--------|
| sensor_data | backend | frontend | 3 | ... |
```

### 6f. Contract Quality Report

```markdown
## Contract Quality

### Python: response_model Coverage

| Router | Total | Typed | Untyped | Coverage |
|--------|-------|-------|---------|----------|
| auth | N | N | N | N% |

### TypeScript: Generated Type Quality

| Metric | Count |
|--------|-------|
| Total response types | N |
| Concrete types | N |
| `unknown` types | N |
| **Type coverage** | **N%** |

### Cross-Reference

[Note any mismatches between untyped Python endpoints and unknown TS types]
```

### 6g. Maintenance Plan (agent output)

If Plan agent ran successfully, include:

```markdown
## API Maintenance Plan

### Documentation Priorities

| Endpoint | Method | Consumers | Priority |
|----------|--------|-----------|----------|

### Deprecation Timeline

| Phase | Endpoints | Action | Timeline |
|-------|-----------|--------|----------|

### Breaking Changes

| Change | Consumer Count | Migration Path |
|--------|----------------|----------------|
```

If agent did not run: `## API Maintenance Plan\n\nN/A — Agent skipped (--quick) or timed out.`

### 6h. Auto-Fixed Section

**Not applicable for API surface audit** — API changes require design decisions.
```markdown
## Auto-Fixed

N/A — API surface audit is read-only. Use findings to prioritize API cleanup.
```

### 6i. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6j. Historical Section

If prior existed, list resolved items per shared.md format.

### 6k. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. Orphan endpoints (0 consumers — candidates for removal)
2. Low contract quality coverage (missing `response_model` on consumer-facing endpoints)
3. Undocumented WebSocket message flows

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules — append new CRITICAL/HIGH findings to `tech-debt-patterns.md`, mark resolved entries.

### 7b. Git Commit

```bash
git add work/MMDDYY_API_Surface.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): api surface audit — N findings (X critical, Y high)

Tools: Grep, Read
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

| Severity | Criteria |
|----------|----------|
| HIGH | Orphan endpoint with security implications (auth bypass, data leak) |
| HIGH | Contract quality < 80% (widespread `unknown` types) |
| MEDIUM | Orphan endpoint with no security risk (dead code) |
| MEDIUM | Missing `response_model` on endpoint with frontend consumers |
| MEDIUM | Undocumented WebSocket message type in active use |
| LOW | Missing `response_model` on internal/server-to-server endpoint |
| LOW | Endpoint documentation incomplete but functional |
| INFO | Well-documented endpoint with proper `response_model` |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key API surface-specific items:

- [ ] Phase 1: FastAPI route decorators scanned
- [ ] Phase 1: WebSocket message types mapped
- [ ] Phase 1: External API calls documented
- [ ] Phase 2: Explore agent ran (full mode) or skipped (quick mode) with correct header
- [ ] Phase 3: Contract quality — Python `response_model` coverage calculated
- [ ] Phase 3: Contract quality — TypeScript `unknown` types counted
- [ ] Phase 3: Cross-reference between Python untyped and TS unknown completed
- [ ] Phase 3: Known exceptions (skip-list) applied
- [ ] Phase 4: Plan agent ran (full mode) or skipped (quick mode) with correct header
- [ ] API Surface Map populated with consumer counts
- [ ] Contract Quality Report populated with coverage percentages
- [ ] Maintenance Plan populated (if agent ran) or marked N/A
- [ ] Findings numbered with AS- prefix
- [ ] `--fix` noted as N/A (read-only audit)

---

## See Also

- `/audit-typescript` — TypeScript audit (overlaps: type safety, dead exports)
- `/audit-python` — Python code quality (overlaps: FastAPI patterns)
- `/audit-deadcode` — Dead code audit (overlaps: orphan endpoints are dead code)
- `/audit-resilience` — Resilience audit (overlaps: error handling in API layer)
- `/audit-observability` — Observability audit (overlaps: endpoint monitoring)
