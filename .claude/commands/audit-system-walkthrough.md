# /audit-system-walkthrough

System walkthrough audit: end-to-end data flow tracing through all layers (device connection, signal processing, backend WebSocket, relay, frontend WS, UI rendering). Includes Explore agent for automated flow tracing and a manual file reading table covering 7 stages.

**Target:** `devices/*.py`, `your-rust-crate/src/*.rs`, `api/websocket/*.py`, `your-rust-service/src/main.rs`, `src/services/WebSocket*.ts`, `src/components/*.tsx`

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
   - `AUTO_FIX` — `false` always (`--fix` is a no-op — architecture changes require design decisions)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for system walkthrough audit (architecture changes require design decisions)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_System_Walkthrough.md 2>/dev/null | sort -r | head -1
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

Run all phases (skip agent in Phase 2 if `--quick`), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### System Walkthrough Delta Logic

In delta mode (`--since last`):
1. Load PRIOR_BASELINE for this audit (flow descriptions)
2. If <10 files changed: CARRY FORWARD prior flow, only re-trace paths through CHANGED_FILES
3. If >50% source files changed: full trace
4. VERIFY: Spot-check 3-5 carried flow descriptions still accurate
5. MERGE + MARK: NEW | CARRIED | CHANGED | RESOLVED

If MODE == "full": Run full trace on entire codebase.

### Phase 1: Manual File Reading

Read key files at each stage of the data flow pipeline:

| Stage | Files to Read |
|-------|---------------|
| Device connection | `devices/*.py` |
| Signal processing | `your-rust-crate/src/*.rs`, `rust_signal_adapter.py` |
| Backend WebSocket | `api/websocket/*.py` |
| Relay client | `api/websocket/relay_client.py` |
| Relay server | `your-rust-service/src/main.rs` |
| Frontend WS | `src/services/WebSocket*.ts` |
| UI rendering | `src/components/*.tsx` |

**Analysis per stage:**
1. Read key files at each stage
2. Document the data flow
3. Note any bottlenecks or complexity
4. Update if architecture has changed

### Phase 2: Agent Analysis (if AGENT_MODE = true)

Launch Explore agent with `subagent_type=Explore`.

**Delta mode:** Use Agent Scope Narrowing from shared.md — pass prior flow summary + CHANGED_FILES list. Agent: "Prior flow is [paste]. Only re-trace paths through these N changed files."

**Full mode prompt:**
> "Trace the complete data flow through the this project system:
>
> 1. **Device Layer**: How do external data sources connect to the system?
> 2. **Backend Layer**: How does signal processing transform raw data?
> 3. **Relay Layer**: How does the Rust relay coordinate state?
> 4. **Frontend Layer**: How does the frontend receive and render live data?
>
> Include:
> - Error handling paths
> - Edge cases (device disconnect, relay failure)
> - Alternative flows (demo device vs real device)
>
> Focus on: backend/, src/, your-rust-service/
>
> Return a structured flow diagram with file:line references."

**Use agent output to:**
- Populate the "Data Flow" section with traced paths
- Add error paths not visible from manual file reading
- Include file:line references for each flow stage

**On timeout/failure:** Continue with manual file reading from Phase 1. Note `Agent: None (timeout)` in header.

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_System_Walkthrough.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# System Walkthrough

**Date:** MMDDYY
**Prior:** work/MMDDYY_System_Walkthrough.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Explore | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Read (file analysis)
```

### 6b. Executive Summary

2-3 sentences covering:
- Data flow health: are all 7 stages connected and documented?
- Highest-impact finding (e.g., undocumented path, broken error handling chain)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Flow stages traced | N/7 | — | — |
| Error paths documented | N | — | — |
| Alternative flows documented | N | — | — |
| Undocumented paths found | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Data Flow Diagram

Document the traced data flow:

```markdown
## Data Flow

### Primary Flow: Device → UI

| Stage | Component | File | Entry Point | Output | Status |
|-------|-----------|------|-------------|--------|--------|
| 1. Device Connection | BiometricDevice | devices/*.py | connect() | raw samples | NEW/CARRIED/CHANGED |
| 2. Signal Processing | your-rust-crate | src/*.rs | process_data() | processed bands | ... |
| 3. Backend WS | WebSocket handler | api/websocket/*.py | broadcast() | WS message | ... |
| 4. Relay Client | relay_client.py | relay_client.py | send() | relay frame | ... |
| 5. Relay Server | Rust relay | main.rs | handle_ws() | broadcast | ... |
| 6. Frontend WS | WebSocketManager | services/WebSocket*.ts | onMessage() | store update | ... |
| 7. UI Rendering | Components | components/*.tsx | useStore() | visual output | ... |

### Error Handling Paths

[Document error propagation at each stage]

### Alternative Flows

- **Demo device** — [path differences]
- **Real device** — [path differences]
- **Alternative data sources — document path differences]
```

### 6e. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **SW-** prefix (SW-1, SW-2, SW-3... for System Walkthrough findings).

### 6f. Auto-Fixed Section

**Not applicable for system walkthrough audit** — architecture changes require design decisions.
```markdown
## Auto-Fixed

N/A — System walkthrough audit is read-only. Use findings to improve documentation and error handling.
```

### 6g. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6h. Historical Section

If prior existed, list resolved items per shared.md format.

### 6i. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. Broken or undocumented data flow paths
2. Missing error handling at stage boundaries
3. Architecture drift from documented design

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules — append new CRITICAL/HIGH findings to `tech-debt-patterns.md`, mark resolved entries.

### 7b. Git Commit

```bash
git add work/MMDDYY_System_Walkthrough.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): system walkthrough — N findings (X critical, Y high)

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
| CRITICAL | Data flow stage completely broken or disconnected |
| HIGH | Error handling missing at stage boundary (data silently lost) |
| HIGH | Undocumented alternative flow in production use |
| MEDIUM | Stage documented but implementation has drifted from docs |
| MEDIUM | Error path exists but doesn't propagate to user |
| LOW | Minor documentation gap (stage works but isn't fully traced) |
| INFO | Flow stage healthy and well-documented |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key system walkthrough-specific items:

- [ ] Phase 1: All 7 stages of manual file reading completed
- [ ] Phase 1: Device connection, signal processing, backend WS, relay client, relay server, frontend WS, UI rendering all traced
- [ ] Phase 2: Explore agent ran (full mode) or skipped (quick mode) with correct header
- [ ] Data Flow Diagram populated with file:line references
- [ ] Error handling paths documented per stage
- [ ] Alternative flows documented (demo vs real devices, browser-origin vs native)
- [ ] Delta mode: <10 files = carry forward, >50% = full trace
- [ ] Findings numbered with SW- prefix
- [ ] `--fix` noted as N/A (read-only audit)

---

## See Also

- `/audit-performance` — Performance + bottleneck analysis (overlaps: complexity, hot paths through the flow)
- `/audit-resilience` — Resilience audit (overlaps: error handling at boundaries)
- `/audit-observability` — Observability audit (overlaps: logging/metrics at each stage)
