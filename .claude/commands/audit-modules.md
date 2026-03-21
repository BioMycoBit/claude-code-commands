# /audit-modules

Module boundary audit: Python module dependency enforcement via tach, TypeScript unused exports/files/dependencies via knip, and architecture boundary validation across both ecosystems.

**Target:** `backend/` (Python — tach), `<frontend-dir>/` (TypeScript — knip)

---

## Scope Boundary

**This audit covers:** Module boundary enforcement — ensuring layers don't import across boundaries (e.g., routers don't import device internals), detecting unused exports/files/dependencies, and validating architecture constraints.

**NOT covered here (see /audit-deadcode):** General dead code detection — unused functions, unreachable code, orphan files by grep pattern. This audit uses dedicated tools (tach, knip) for structural analysis; `/audit-deadcode` uses grep patterns and vulture.

**NOT covered here (see /audit-dependency):** Dependency version management, CVE scanning, outdated packages. This audit checks if dependencies are *used*; `/audit-dependency` checks if they're *current and secure*.

**NOT covered here (see /audit-python, /audit-typescript):** Code quality, linting, complexity. This audit checks architecture boundaries; language audits check code correctness.

---

## Step 1: Load Shared Infrastructure

1. Read `.claude/commands/audit/shared.md`
2. All output format, severity levels, delta tracking, archival, tech debt integration, commit format, sprint chain, error handling, and completion report rules come from shared.md — do NOT deviate

---

## Step 2: Parse Flags

Parse `$ARGUMENTS` per shared Flag Parsing rules:

1. Strip flags from arguments, set booleans:
   - `AGENT_MODE` — `true` unless `--quick` is set
   - `COMPARE` — `true` if `--since last`
   - `SPRINT_CHAIN` — `true` if `--sprint`
   - `AUTO_FIX` — `true` if `--fix` (runs `knip --fix` for unused TS exports)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix runs knip --fix (unused TS exports only). Python boundary violations require manual refactoring.
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Modules_Audit.md 2>/dev/null | sort -r | head -1
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

Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Phase 0: Configuration Check

**tach configuration:**

Check if `tach.toml` exists in `backend/`:
```
Glob("tach.toml", path="<backend-dir>")
Glob("tach.toml", path=".")
```

If no `tach.toml` found, note:
```
No tach.toml found. tach requires configuration to define module boundaries.
To initialize: cd <backend-dir> && tach mod
This generates tach.toml with discovered module structure. Review and commit.
```
Skip Phase 1 tool run, proceed to Phase 1 grep-based analysis.

**knip configuration:**

Check if knip config exists:
```
Glob("knip.json", path="<frontend-dir>")
Glob("knip.jsonc", path="<frontend-dir>")
```

Also check for `knip` key in `package.json`:
```
Grep("\"knip\"", path="package.json")
```

If no config found, note:
```
No knip configuration found. knip works without config but may produce noise.
To initialize: cd <frontend-dir> && npx knip --include-entry-files
Review output, then add knip.json for project-specific settings.
```

### Phase 1: Python Module Boundaries (tach)

Run tach on the Python backend:

```bash
cd <backend-dir> && tach check 2>&1
echo "Exit code: $?"
```

If tach is not installed:
```
Tool not installed: tach
Install: uv tool install tach | pip install tach
```
Note in output under Tools and proceed to grep-based boundary analysis below.

Parse tach output:
- Extract: source module, target module, import path, violation type
- Group by boundary violation (which layer imports what it shouldn't)
- Map to severity based on architectural risk

**Grep-based boundary analysis** (always runs, supplements tach):

Run all Grep calls in a single parallel message:

**Scan Group 1 — Router importing device internals:** 1 Grep call, path=`api/routers/`
Pattern: `from devices\.|import devices\.`
Routers should use device manager, not device internals = HIGH

**Scan Group 2 — Device importing router/API internals:** 1 Grep call, path=`devices/`
Pattern: `from api\.routers|from api\.middleware|import api\.routers`
Devices should be API-agnostic = HIGH

**Scan Group 3 — Circular imports (services ↔ routers):** 1 Grep call, path=`api/services/`
Pattern: `from api\.routers|import api\.routers`
Services should not import routers = MEDIUM

**Scan Group 4 — Config importing runtime modules:** 1 Grep call, path=`config/`
Pattern: `from api\.|from devices\.|import api\.|import devices\.`
Config should be leaf-level = MEDIUM

### Phase 2: TypeScript Module Analysis (knip)

Run knip on the frontend:

```bash
cd <frontend-dir> && npx knip 2>&1
echo "Exit code: $?"
```

If knip is not installed:
```
Tool not installed: knip
Install: cd <frontend-dir> && npm install -D knip
```
Note in output under Tools and proceed to grep-based analysis below.

Parse knip output — knip reports:
- **Unused files** — source files with no imports
- **Unused dependencies** — packages in package.json never imported
- **Unused devDependencies** — dev packages never referenced
- **Unused exports** — exported symbols with no consumers
- **Unlisted dependencies** — imports not in package.json

Group by category, map to severity:
- Unused dependencies in production = HIGH (bloats bundle)
- Unused files = MEDIUM (dead code)
- Unused exports = MEDIUM (API surface noise)
- Unused devDependencies = LOW (dev-only bloat)

**If `AUTO_FIX` is true:**

```bash
cd <frontend-dir> && npx knip --fix 2>&1
```

Record what was fixed in Auto-Fixed section. knip --fix removes unused exports only (safe operation).

**Grep-based TS boundary analysis** (always runs, supplements knip):

**Scan Group 5 — Component importing service internals:** 1 Grep call, path=`src/components/`
Pattern: `from ['"].*services/.*/(internal|private|_)`
Components should use service public API = HIGH

**Scan Group 6 — Barrel re-exports without consumers:** 1 Grep call, path=`src/`
Pattern: `export \* from|export \{ .+ \} from` in `index.ts` files
Cross-reference with import consumers to find dead re-exports = LOW

### Phase 3: Cross-Reference with Dead Code

Cross-reference knip's unused files/exports with `/audit-deadcode` findings (if a prior deadcode audit exists):

```bash
ls work/*_Deadcode_Audit.md work/*_DeadCode_Audit.md 2>/dev/null | sort -r | head -1
```

If found:
1. Read the deadcode audit's findings tables
2. Identify overlapping findings (same file flagged by both tools)
3. Note cross-references using prefix: `See also: DC-N in /audit-deadcode`
4. Bump severity if same file flagged by both audits (per shared.md Cross-Audit Correlation)

### Phase 4: Architecture Boundary Summary

Produce a module dependency matrix showing which layers import which:

```markdown
## Architecture Boundary Matrix

### Python Backend
| Source Layer | → api.routers | → api.services | → api.websocket | → devices | → config |
|-------------|---------------|----------------|-----------------|-----------|----------|
| api.routers | — | ✅ | ✅ | ❌ (via manager) | ✅ |
| api.services | ❌ | — | ✅ | ❌ (via manager) | ✅ |
| devices | ❌ | ❌ | ❌ | — | ✅ |
| config | ❌ | ❌ | ❌ | ❌ | — |

✅ = allowed import | ❌ = boundary violation if present | (via manager) = only through DeviceManager
```

Fill in actual import status: `✅ clean`, `⚠️ N violations`, `❌ boundary broken`.

### Phase 5: Plan Agent Analysis (if AGENT_MODE)

Launch **one** Plan agent with `subagent_type=Plan` AFTER Phases 1-4 complete.

**Prompt:**
> "Analyze module boundary violations and unused code for a multi-service application:
>
> **tach violations (Python):** [paste Phase 1 output]
> **knip results (TypeScript):** [paste Phase 2 output — first 50 items max]
> **Boundary grep results:** [paste Phase 1+2 grep findings]
> **Architecture matrix:** [paste Phase 4 matrix]
>
> **Analyze:**
> 1. **Identify dependency cycles** — which modules form circular dependency chains?
> 2. **Assess boundary health** — are layer separations clean or eroding?
> 3. **Prioritize cleanup** — which violations are highest risk for maintenance?
> 4. **Recommend boundary rules** — what tach.toml / knip.json config would enforce clean architecture?
> 5. **Estimate effort** — group violations into cleanup sprints
>
> **Return:** Architecture health assessment with prioritized violation list and recommended tool configuration."

**On timeout/failure:** Continue with tool output only. Note `Agent: None (timeout)` in header.

### Phase 6: Verification (shared.md Phase 5.5)

For each CRITICAL/HIGH finding from Phases 1-4:
1. **Read the file** at the reported line number
2. **Confirm the finding is real** — not a re-export, type-only import, or intentional cross-boundary access with documented reason
3. **Assign confidence:** HIGH, MEDIUM, LOW
4. **Drop LOW confidence findings**

Skip this step if `--quick` is set.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Modules_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Modules Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Modules_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** tach (Python module boundaries), knip (TS unused exports/files/deps), Grep (boundary patterns)
```

### 6b. Executive Summary

2-3 sentences covering:
- tach boundary violation count and most violated boundary
- knip unused items count (files, deps, exports)
- Architecture health summary
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Python boundary violations (tach) | N | — | — |
| Python boundary violations (grep) | N | — | — |
| TS unused files (knip) | N | — | — |
| TS unused dependencies (knip) | N | — | — |
| TS unused exports (knip) | N | — | — |
| TS unlisted dependencies (knip) | N | — | — |
| Cross-audit overlaps (deadcode) | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **MD-** prefix (MD-1, MD-2, MD-3...).

### 6e. Auto-Fixed Section

If `AUTO_FIX` was true and `knip --fix` ran:
```markdown
## Auto-Fixed

| # | File | What Changed | Tool |
|---|------|-------------|------|
| 1 | src/components/Foo.tsx | Removed unused export `FooProps` | knip --fix |
```

If `AUTO_FIX` was false:
```markdown
## Auto-Fixed

N/A — Run with `--fix` to auto-remove unused TypeScript exports via `knip --fix`. Python boundary violations require manual refactoring.
```

### 6f. Architecture Boundary Matrix

Include the matrix from Phase 4 in the output document.

### 6g. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6h. Historical Section

Per shared.md format.

### 6i. Recommendations

Top 3 highest-impact recommendations. Prioritize:
1. Circular dependency chains (hardest to fix later)
2. Layer boundary violations (architectural erosion)
3. Unused dependencies (bundle bloat, attack surface)

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules.

### 7b. Git Commit

```bash
git add work/MMDDYY_Modules_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): modules audit — N findings (X critical, Y high)

Tools: tach, knip, Grep
Mode: Full | Quick
[Delta: +N new, -N resolved]  (only if --since last)
[Auto-fixed: N unused exports]  (only if --fix)

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
| CRITICAL | Circular dependency causing import errors or runtime failures | Router ↔ Service circular import crashing on startup |
| HIGH | Layer boundary violation in production code | Router importing device internals directly (bypasses DeviceManager) |
| HIGH | Unused production dependency (attack surface + bundle size) | `lodash` in dependencies but never imported |
| HIGH | Unlisted dependency used in production | Import from package not in package.json |
| MEDIUM | Service importing from router layer (wrong direction) | Service reading router-specific request types |
| MEDIUM | Unused source file (dead code) | Component file with zero imports |
| MEDIUM | Unused export in production code (API surface noise) | Exported function with no consumers |
| LOW | Unused devDependency | Test utility package never imported |
| LOW | Barrel re-export with no consumers | index.ts re-exporting symbol nobody imports |
| LOW | Config importing from runtime module (minor boundary) | Config reading from services |
| INFO | tach/knip configuration suggestion | Missing boundary rule, noisy default |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key modules-specific items:

- [ ] Phase 0: tach.toml and knip config checked (or init commands noted)
- [ ] Phase 1: tach check ran (or install command noted) AND grep boundary scans ran
- [ ] Phase 2: knip ran (or install command noted) AND grep boundary scans ran
- [ ] Phase 3: Cross-referenced with deadcode audit if prior exists
- [ ] Phase 4: Architecture boundary matrix produced
- [ ] Phase 5: Plan agent ran (if AGENT_MODE, or timeout noted)
- [ ] Phase 6: CRITICAL/HIGH findings verified via Read tool
- [ ] `--fix` runs `knip --fix` if set, or noted as N/A
- [ ] Findings numbered with MD- prefix
- [ ] Agent header field set correctly

---

## See Also

- `/audit-deadcode` — Dead code detection (grep patterns, vulture — structural overlap possible)
- `/audit-dependency` — Dependency version management (versions, CVEs — not usage)
- `/audit-python` — Python code quality (ruff, complexity — not boundaries)
- `/audit-typescript` — TypeScript code quality (oxlint, React — not exports/deps)
- `/audit-types` — Type checking (ty, tsc — not module structure)
