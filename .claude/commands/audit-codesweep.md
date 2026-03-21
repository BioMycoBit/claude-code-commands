# Codesweep Skill

Scan the codebase for copy/paste duplication using jscpd, save analysis report to `work/MMDDYY_CodeSweep.md`, and output a console summary.

## Usage

```
/audit-codesweep
/audit-codesweep --strict
/audit-codesweep --weak
/audit-codesweep --since last
/audit-codesweep --since last --strict
/audit-codesweep --sprint
/audit-codesweep --sprint v1.6f_codesweep
/audit-codesweep --sprint --since last
```

**Arguments:**
- No arguments — scan with mild mode (default), no sprint association
- `--strict` — strict detection mode (character-for-character match)
- `--mild` — mild detection mode (ignore blank lines, default)
- `--weak` — weak detection mode (ignore blank lines AND comments)
- `--since last` — compare current scan against the most recent archived baseline, show deltas (new/resolved/unchanged clones)
- `--sprint` — after report generation, chain into `/sprint` with codesweep findings as input for refactoring-focused sprint planning. Optionally followed by a `v*_*` folder shorthand (e.g., `v1.6f_codesweep`); if omitted, auto-increments from the latest sprint version

## Output

Creates:
1. `tools/codesweep/jscpd-report.json` — raw jscpd JSON report
2. `work/MMDDYY_CodeSweep.md` — scored, prioritized cleanup report with refactoring tickets (date-stamped, same convention as eod-docs audits)
3. Console summary — total duplication %, clone count, top 3 offenders with scores

---

## Step 1: Parse Arguments

### 1a: Detect Mode Flags & Comparison Flag

Check arguments for mode flags:
- `--strict` → set MODE to `strict`
- `--mild` → set MODE to `mild`
- `--weak` → set MODE to `weak`
- No flag → default to `mild`

Check arguments for comparison flag:
- `--since last` → set COMPARE to `true`
- No flag → set COMPARE to `false`

Check arguments for sprint chain flag:
- `--sprint` → set SPRINT_CHAIN to `true`
- No flag → set SPRINT_CHAIN to `false`

Remove all detected flags from arguments after parsing.

### 1b: Detect Sprint Folder Shorthand (only when --sprint is set)

**Skip this step entirely if SPRINT_CHAIN is `false`.**

If `--sprint` is set, check if remaining arguments (after all flags removed) contain a `v{version}_{name}` pattern (e.g., `v1.6f_codesweep`).

**If shorthand detected:**
1. Extract the full identifier (e.g., `v1.6f_codesweep`)
2. Set **SPRINT_FOLDER** = `work/{identifier}` (e.g., `work/v1.6f_codesweep`)

**If no shorthand detected (--sprint without folder):**
- Auto-increment in Step 7a (deferred until sprint chain runs)

Report mode to user:
```
Mode: mild
```

---

## Step 2.5: Archive Prior Report

Before running the scan, archive the current `jscpd-report.json` (if it exists) so future `--since last` runs have a baseline to compare against.

```bash
# Only archive if the file exists and has content
mkdir -p tools/codesweep/jscpd_reports
if [ -s tools/codesweep/jscpd-report.json ]; then
  ARCHIVE_DATE=$(date +%Y%m%d)
  cp tools/codesweep/jscpd-report.json "tools/codesweep/jscpd_reports/jscpd-report-${ARCHIVE_DATE}.json"
fi
```

**Notes:**
- Archive naming: `tools/codesweep/jscpd_reports/jscpd-report-{YYYYMMDD}.json`
- If an archive with today's date already exists, overwrite it (same-day re-runs produce same date)
- This runs unconditionally (not just when `--since last` is set) so baselines accumulate for future comparisons

---

## Step 3: Run jscpd Scan

Run jscpd with the project config, overriding mode if specified:

```bash
mkdir -p tools/codesweep/jscpd_reports
npx jscpd --config .jscpd.json \
  --mode ${MODE} \
  --reporters json,console \
  --output tools/codesweep \
  <frontend-dir>/ your-rust-service/ your-rust-crate/ your-cli/
```

**Important notes:**
- Use `npx jscpd` (v4.0.8 confirmed available)
- The `--output tools/codesweep` flag saves `jscpd-report.json` to `tools/codesweep/`
- The `--config .jscpd.json` flag loads project-specific ignores and settings
- The source paths at the end are the directories to scan
- If a source directory doesn't exist, jscpd silently skips it

**Exit code 1 is expected** when `threshold: 0` — jscpd reports "too many duplicates" as an error. This is normal. The scan succeeded if `tools/codesweep/jscpd-report.json` was created. Only treat it as a real failure if the JSON file is missing.

**If jscpd truly fails** (no JSON produced): Report the error and stop. Do not proceed to Step 4.

---

## Step 3.5: Analyze & Score

Run the analyzer script. This handles parsing, scoring, grouping, report generation, and optional delta comparison in one execution.

```bash
# Without comparison:
your-cli codesweep analyze --mode ${MODE}

# With comparison (--since last):
PRIOR_REPORT=$(ls -1 tools/codesweep/jscpd_reports/jscpd-report-*.json 2>/dev/null | sort -V | tail -1)
if [ -n "$PRIOR_REPORT" ]; then
  your-cli codesweep analyze --mode ${MODE} --compare "$PRIOR_REPORT"
else
  echo "No prior baseline found — skipping comparison."
  your-cli codesweep analyze --mode ${MODE}
fi
```

**What the script does (Steps 3.5–3.8 combined):**

1. **Parses** `tools/codesweep/jscpd-report.json` — extracts stats, formats, duplicates
2. **Normalizes** paths (strips absolute prefix), groups by file pair (alphabetical order)
3. **Scores** each group: `(lines × 2) + (frequency × 10) + locality_bonus + risk_weight`
4. **Generates** suggested actions per group (extract helper, shared module, test fixture, etc.)
5. **Writes** `tools/codesweep/codesweep-scored.json` — structured data for programmatic use
6. **Writes** `work/MMDDYY_CodeSweep.md` — markdown with executive summary, ranked groups, top 15 refactoring tickets
7. **Compares** against prior baseline (if `--compare` provided) — classifies NEW/RESOLVED/UNCHANGED groups, appends delta section to report
8. **Prints** JSON summary to stdout — use this for Step 4 console output and Step 5.5 memory

**Stdout JSON structure:**
```json
{
  "statistics": { "lines", "sources", "clones", "duplicatedLines", "percentage", ... },
  "formats": { "python": { "sources", "clones", "duplicatedLines", "percentage" }, ... },
  "group_count": 67,
  "top3": [ { "fileA", "fileB", "score", "total_lines", "clone_count", "action" }, ... ],
  "mode": "mild",
  "delta": { "trend_arrow", "trend_word", "net_change", "new_count", "resolved_count", ... }
}
```

**If the script fails** (non-zero exit, no JSON output): Report the error and stop.

**Scoring formula details** (implemented in your codesweep scoring module):
- **locality_bonus:** `5` if same parent directory, `0` otherwise
- **risk_weight:** `10` (devices/, api/, services/), `5` (relay, signals, stream), `1` (everything else)
- **Actions:** self-duplication → "Extract shared helper", same dir → "Extract to shared module", cross-dir → "Extract to common lib", tests → "Consider test fixture", generated → "Skip"

---

## Step 4: Console Summary

Use the JSON summary printed to stdout by `analyze.py` (Step 3.5). Extract all values from that JSON — do NOT re-read the raw jscpd report.

Output format:
```
## Codesweep Results

**Duplication:** {percentage}% ({duplicatedLines} lines, {clones} clones across {group_count} file pairs)
**Sources scanned:** {sources} files
**Mode:** {MODE}

### Format Breakdown
| Format | Sources | Clones | Lines | % |
|--------|---------|--------|-------|---|
| Python | X | Y | Z | N% |
| TypeScript | X | Y | Z | N% |

### Top Offenders (by score)
| # | Score | File A | File B | Lines | Clones | Action |
|---|-------|--------|--------|-------|--------|--------|
| 1 | 127 | path/a.py | path/b.py | 48 | 4 | Extract to shared module |
| 2 | 85 | path/c.py | path/d.py | 13 | 2 | Extract shared helper |
| 3 | 42 | path/e.ts | path/f.ts | 8 | 1 | Consider test fixture |

**Reports saved:** tools/codesweep/jscpd-report.json, work/MMDDYY_CodeSweep.md
```

**When COMPARE is `true` and a prior baseline was found**, append this section to the console output:

```
### Delta (vs {prior_report_date})
**Trend:** {TREND_ARROW} {trend_word}
**Duplication:** {prior_percentage}% → {current_percentage}% ({percentage_delta:+.2f}%)

| Status | Count |
|--------|-------|
| NEW | {new_count} |
| RESOLVED | {resolved_count} |
| UNCHANGED | {unchanged_count} |
| Net change | {net_change:+d} |
```

**When COMPARE is `true` but no prior baseline was found**, show:
```
### Delta
No prior baseline found — run `/audit-codesweep` again after changes to compare.
```

---

## Step 5: Commit

Commit config (if new), the analysis report, and memory updates. Raw data in `tools/codesweep/` is gitignored — it regenerates on each run.

```bash
git add .jscpd.json 2>/dev/null
# Also add the analysis report and memory/velocity.md if updated
git add work/MMDDYY_CodeSweep.md 2>/dev/null
git add memory/velocity.md 2>/dev/null
git commit -m "chore(codesweep): scan results — {percentage}% duplication ({clones} clones)

Mode: {MODE}
Sources: {sources} files
Duplicated lines: {duplicatedLines}
{If COMPARE: "Delta: +{new_count} new, -{resolved_count} resolved, net {net_change:+d} {TREND_ARROW}"}

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Step 5.5: Memory Persistence

After committing, append a codesweep entry to `memory/velocity.md`.

**If `memory/velocity.md` does not exist**, create it with a header first:
```markdown
# Velocity Metrics

Automated metrics from sprints and tools. Append-only, date every entry.

**Cap:** 150 lines | **Write:** /audit-codesweep, /sprint-end | **Prune:** Oldest first

---
```

**Append this entry** (2-3 lines, compact):

```markdown
### Codesweep — {YYYY-MM-DD}
**Duplication:** {percentage}% ({clones} clones, {group_count} groups) | **Trend:** {TREND_ARROW or "baseline"} | **Top:** {top_offender_fileA} ↔ {top_offender_fileB} ({top_score} pts)
```

**Notes:**
- First run has no trend — use "baseline" instead of an arrow
- When COMPARE is `true`: use the computed TREND_ARROW (↑/↓/→)
- Keep entries compact — each run adds exactly 2 lines (heading + metrics)
- The 150-line cap allows ~70 entries before pruning is needed

---

## Step 6: Output Summary

Final output to user:

```
## Codesweep Complete

**Raw report:** tools/codesweep/jscpd-report.json
**Analysis report:** work/MMDDYY_CodeSweep.md
**Mode:** {MODE}

| Metric | Value |
|--------|-------|
| Duplication | {percentage}% |
| Clone groups | {group_count} file pairs |
| Individual clones | {clones} |
| Duplicated lines | {duplicatedLines} |
| Sources scanned | {sources} |

### Top Offender
{fileA} ↔ {fileB} — score {score}, {lines} lines, {count} clones
Action: {suggested_action}
```

**When COMPARE is `true` and baseline found**, also show:
```
### Trend: {TREND_ARROW} {trend_word}
**Duplication:** {prior_percentage}% → {current_percentage}% ({percentage_delta:+.2f}%)
**Delta:** +{new_count} new, -{resolved_count} resolved (net {net_change:+d})

Next: Review work/MMDDYY_CodeSweep.md for delta details and refactoring tickets.
```

**Otherwise:**
```
Next: Review work/MMDDYY_CodeSweep.md for refactoring tickets, then `/audit-codesweep --since last` after cleanup to measure improvement.
```

---

## Step 7: Sprint Chain

**Only runs when SPRINT_CHAIN is `true` (`--sprint` flag set).**

After the report is committed and the output summary is displayed, automatically chain into `/sprint` with codesweep findings as context for refactoring-focused sprint planning.

### 7a: Determine Sprint Folder

**If SPRINT_FOLDER was set in Step 1b** (user provided explicit shorthand like `v1.6f_codesweep`), use it.

**If SPRINT_FOLDER is not set** (user ran `--sprint` without a folder shorthand), auto-increment:

```bash
# Find latest sprint version
LATEST=$(ls -d work/v* work/00_backup/v* 2>/dev/null | sed 's|.*/||' | sort -Vu | tail -1)
# Example: v1.6e_codesweep

# Increment the letter suffix: e → f, f → g, etc.
# If latest is v1.6e_codesweep → new folder is v1.6f_codesweep
# Extract version prefix (v1.6), letter (e), bump letter
```

Set **SPRINT_FOLDER** = `work/v{version}{next_letter}_codesweep` (always use `_codesweep` as the name for codesweep-driven sprints).

Report to user:
```
Sprint chain: creating work/v1.6f_codesweep
```

### 7c: Extract Sprint Context

From the `analyze.py` stdout JSON and `codesweep-scored.json`, extract:

1. **Top 10 offenders** — file pairs with highest scores
2. **Total duplication metrics** — percentage, clone count, duplicated lines
3. **Refactoring tickets** — the top 15 tickets from the report
4. **Trend data** (if `--since last` was also set) — net change, trend arrow

### 7d: Build Sprint Focus

Construct the sprint focus string:

```
Refactoring sprint: reduce {percentage}% code duplication ({clones} clones across {group_count} file pairs).

Top targets:
1. {fileA} ↔ {fileB} — score {score}, {lines} lines — {action}
2. {fileA} ↔ {fileB} — score {score}, {lines} lines — {action}
3. {fileA} ↔ {fileB} — score {score}, {lines} lines — {action}
...up to 10

Full report: work/MMDDYY_CodeSweep.md
```

### 7e: Invoke Sprint

Invoke `/sprint {SPRINT_FOLDER_SHORTHAND}` with the constructed focus. For example: `/sprint v1.6f_codesweep Refactoring sprint: reduce ...`

The sprint skill will:
1. Use the folder shorthand to create `work/v1.6f_codesweep/`
2. Use the focus string as its research context
3. Auto-discover the latest `work/*_CodeSweep.md` via Step 2e (see sprint.md)
4. Generate a refactoring-focused sprint plan with sessions targeting the highest-score clone groups
5. Create the overview and first handoff as usual

### 7f: Post-Chain Summary

After `/sprint` completes, output:

```
## Sprint Chain Complete

**Codesweep** → **Sprint** chain finished.

- Codesweep report: work/MMDDYY_CodeSweep.md
- Sprint overview: work/{folder}/00_{topic}_overview_v4.md
- First handoff: work/{folder}/01_{session}_handoff.md

Refactoring sessions planned based on {group_count} clone groups ({percentage}% duplication).
Ready for GO/NO-GO on Session 01?
```

---

## Error Handling

| Error | Action |
|-------|--------|
| jscpd not available via npx | Report error, suggest `npm install -g jscpd` |
| jscpd exit code 1 | Expected — threshold 0 triggers "too many duplicates". Check if JSON exists; if yes, proceed normally |
| jscpd scan produces empty report | Report "no duplication found" as success |
| analyze.py fails (non-zero exit) | Report the error and stop — do not proceed to Step 4 |
| tools/codesweep/ directory doesn't exist | Create it (and jscpd_reports/) before running scan |
| Invalid mode flag | Default to mild, warn user |
| `--since last` but no archived reports | Warn and skip comparison — run scan normally, archive for next time |
| Prior report JSON is malformed | Warn "corrupt baseline" and skip comparison |
| memory/velocity.md at cap (150 lines) | Prune oldest entries before appending |
| `--sprint` but codesweep scan fails | Do not chain — report the scan failure and stop |
| `--sprint` but `/sprint` fails | Report the sprint failure — codesweep report is still valid and committed |
| `--sprint` with 0 clones found | Skip sprint chain — nothing to refactor. Show "No duplication found — sprint chain skipped" |
| `--sprint` without folder and no prior sprints found | Error — cannot auto-increment without a baseline version. User must provide explicit folder shorthand |

---

## Execution Notes

- **Default mode is mild** — catches real duplication without false positives from formatting
- **Raw data goes to `tools/codesweep/`** (gitignored), **analysis report goes to `work/MMDDYY_CodeSweep.md`** (committed, same convention as eod-docs audits)
- **Analyzer:** `your-cli codesweep analyze` handles Steps 3.5–3.8 (parse, score, group, report, compare)
- **Source directories scanned:** `<frontend-dir>/`, `your-rust-service/`, `your-rust-crate/`, your CLI tool directory
- **Ignores configured in `.jscpd.json`:** work/, node_modules/, venv/, tools/, .claude/, etc.
- **Scoring formula:** `(lines × 2) + (frequency × 10) + locality_bonus + risk_weight` — see `analyze.py` for weight definitions
- **Report generation:** `analyze.py` writes `work/MMDDYY_CodeSweep.md` — top 15 groups get full refactoring tickets
- **Archive naming:** `tools/codesweep/jscpd_reports/jscpd-report-{YYYYMMDD}.json` — same-day re-runs overwrite
- **Delta comparison key:** normalized `(fileA, fileB)` pair — same key used in `analyze.py` grouping
- **Trend arrows:** ↓ = improving, ↑ = regressing, → = stable (based on net clone group change + duplication %)
- **Memory persistence:** `memory/velocity.md` gets a compact 2-line entry per run (150-line cap)
- **Codesweep is sprint-agnostic** — the scan itself has no sprint folder association. Sprint folders only matter when `--sprint` chains into `/sprint`
- **Sprint chain:** `--sprint` triggers Step 7 — determines folder (explicit or auto-increment), then chains into `/sprint` with top offenders as focus. Sprint auto-discovers the latest `work/*_CodeSweep.md` via Step 2e
- **Sprint folder auto-increment:** When `--sprint` is used without an explicit folder, find latest `v*_*` folder, increment the letter suffix (e.g., `v1.6e` → `v1.6f`), always suffix with `_codesweep`
- **Sprint + since last:** Both flags can be combined — scan, compare, then plan a refactoring sprint based on current state
- **eod-docs integration:** `/eod-docs` Day 1 runs `/audit-codesweep` as standalone audit #5 (delegated, same as `/audit-python` etc.)

---

## See Also

- `/audit-python` — Python code quality audit (linting, complexity, dead code — complementary to duplication)
- `/eod-docs day1` — Day 1 Code Hygiene runs `/audit-codesweep` as standalone audit #5
- `/sprint` — Chain into sprint planning with `--sprint` flag
