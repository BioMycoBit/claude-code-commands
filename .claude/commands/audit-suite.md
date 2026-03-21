# /audit-suite

Tier-based audit orchestrator. Launches all audits in a tier as **parallel background agents** with `--quick` and aggregates results into a summary table. No findings of its own — pure orchestration.

**Usage:** `/audit-suite tier1`, `/audit-suite tier2`, `/audit-suite tier3`, `/audit-suite tier4`, `/audit-suite all`

---

## Step 1: Parse Arguments

Parse `$ARGUMENTS` to determine which tier(s) to run:

| Argument | Behavior |
|----------|----------|
| `tier1` | Run Tier 1 audits (5 audits) |
| `tier2` | Run Tier 2 audits (8 audits) |
| `tier3` | Run Tier 3 audits (8 audits) |
| `tier4` | Run Tier 4 audits (5 audits) |
| `all` | Run all tiers sequentially (tier 1 parallel → wait → tier 2 parallel → ...) |
| (no args) | Show usage and tier summary, then stop |

**Additional flags (passed through to each audit):**

| Flag | Behavior |
|------|----------|
| `--full` | Run audits in full mode (default is `--quick`) |
| `--sarif` | Pass `--sarif` to each audit |
| `--since last` | Pass `--since last` to each audit |
| `--no-commit` | Skip the batch commit at the end (audits also skip individual commits) |

**Default mode is `--quick`** unless `--full` is specified. This keeps suite runs fast for routine checks.

Report to user:
```
## Audit Suite

**Tier:** [tier1 | tier2 | tier3 | tier4 | all]
**Mode:** Quick (default) | Full (--full)
**Flags:** [--sarif, --since last, or "none"]
**Audits:** [N] audits queued
```

If no arguments provided:
```
## Audit Suite — Usage

/audit-suite tier1          Run 5 high-priority audits (every sprint)
/audit-suite tier2          Run 8 code quality audits (biweekly)
/audit-suite tier3          Run 8 infrastructure audits (monthly)
/audit-suite tier4          Run 5 compliance audits (quarterly)
/audit-suite all            Run all 26 tiered audits (each tier parallel, tiers sequential)
/audit-suite tier1 --full   Run tier 1 in full mode (with agent analysis)
/audit-suite tier2 --sarif  Run tier 2 with SARIF output

See shared.md Tiered Scheduling for tier assignments.
```

---

## Step 2: Load Tier Definitions

Read `.claude/commands/audit/shared.md` and extract the Tiered Scheduling section for the requested tier(s).

### Tier Membership

| Tier | Audits |
|------|--------|
| 1 | security, dependency, secrets, standards, test-quality |
| 2 | python, typescript, rust, codesweep, resilience, deadcode, regression, performance |
| 3 | docker, observability, pipeline, privacy, modules, types, iac, database |
| 4 | sbom, licenses, accessibility, browser-compat, api-surface |

**On-demand audits** (dashboard, project-overview, system-walkthrough, techdebt) are NOT included in any tier. Run them individually.

---

## Step 3: Execute Audits

Launch **all audits in the tier as parallel background agents** using the Agent tool. Each agent runs one audit independently. This is critical for performance — a 5-audit tier completes in ~7 minutes parallel vs ~25 minutes sequential.

### Agent Launch

Build the flags string from Step 1:
- Default: `--quick --no-commit` (always pass `--no-commit` — suite handles the single batch commit)
- If `--full`: replace `--quick` with just `--no-commit`
- Append `--sarif` and/or `--since last` if specified

Launch **all audits in a single message** with multiple Agent tool calls:

```
For each audit in the tier, launch one Agent:
  description: "{audit-name} audit --quick"
  prompt: "Run /audit-{name} {flags} using the Skill tool. Invoke: Skill(skill: \"audit-{name}\", args: \"{flags}\"). Follow all instructions the skill provides. IMPORTANT: The --no-commit flag means you must NOT run git add or git commit — leave all generated files unstaged. The caller will batch-commit everything. When complete, return the completion report summary including: total findings count, Critical/High/Medium/Low breakdown, and the top 3 findings with file:line and severity."
  run_in_background: true
```

**All agents in one message** — this ensures parallel execution. Do NOT launch them one at a time.

After launching, report the status table to the user:

```markdown
| # | Audit | Status |
|---|-------|--------|
| 1 | {name} | Running... |
| 2 | {name} | Running... |
| ... | ... | ... |
```

### Collecting Results

As each agent completes (via background task notification), acknowledge it briefly:
- `**{audit} complete** — N findings (XC/YH/ZM/WL). [brief note]. N of M done...`

**Do not proceed to Step 4 until all agents have completed.**

### Error Handling

| Error | Action |
|-------|--------|
| Agent fails | Log failure in results table as FAIL, continue waiting for others |
| Agent times out | Log timeout in results table as TIMEOUT, continue waiting |
| Audit produces 0 findings | Log clean result as CLEAN |

**Never stop the suite because one audit fails.** Log the failure and continue.

---

## Step 4: Aggregate Results

After all audits complete, produce the summary.

### Summary Table

```markdown
## Audit Suite Results

**Tier:** [tier]
**Date:** MMDDYY
**Mode:** Quick | Full
**Duration:** [total wall time]

### Results

| # | Audit | Findings | C | H | M | L | Duration | Status |
|---|-------|----------|---|---|---|---|----------|--------|
| 1 | security | 12 | 0 | 3 | 5 | 4 | 45s | OK |
| 2 | dependency | 3 | 1 | 1 | 1 | 0 | 30s | OK |
| 3 | secrets | 0 | 0 | 0 | 0 | 0 | 15s | CLEAN |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
| **Total** | **N audits** | **N** | **N** | **N** | **N** | **N** | **Nm Ns** | |

### Status Legend
- **OK** — Audit completed with findings
- **CLEAN** — Audit completed with 0 findings
- **FAIL** — Audit failed (tool missing, error)
- **TIMEOUT** — Audit timed out

### Top Findings (across all audits)

1. [audit-name] [file:line] — [issue] (CRITICAL)
2. [audit-name] [file:line] — [issue] (HIGH)
3. [audit-name] [file:line] — [issue] (HIGH)

### Health Score

Compute a simple health score per domain:

| Domain | Audits Run | Critical | High | Score |
|--------|-----------|----------|------|-------|
| Security | N | N | N | [A-F] |
| Code Quality | N | N | N | [A-F] |
| Reliability | N | N | N | [A-F] |
| Infrastructure | N | N | N | [A-F] |
| Compliance | N | N | N | [A-F] |

**Scoring:** A = 0 critical + 0 high, B = 0 critical + ≤3 high, C = 0 critical + 4+ high, D = 1 critical, F = 2+ critical

### Audit Documents

List all generated audit documents:
- work/MMDDYY_Security_Audit.md
- work/MMDDYY_Dependency_Audit.md
- ...
```

---

## Step 5: Batch Commit

**Skip if `--no-commit` was passed to the suite itself.**

After all audits complete and the summary is shown, commit all audit artifacts in a single commit.

### What to stage

```bash
# Stage all audit documents generated by the tier
git add work/MMDDYY_*_Audit.md
git add work/MMDDYY_*_Audit.sarif          # if --sarif
git add work/MMDDYY_CodeSweep.md            # codesweep uses different naming
git add work/MMDDYY_Regression_Prevention.md # regression uses different naming
git add docs/eod-docs/                       # promoted priors
git add .claude/rules/tech-debt-patterns.md  # if any audit updated it
git add tools/codesweep/                     # codesweep raw reports
```

Use `git status` first to identify exactly which untracked/modified files the audits produced, then stage only those. Do not stage unrelated changes.

### Commit message

```bash
git commit -m "$(cat <<'EOF'
docs(audit): tier [N] suite — [total] findings across [M] audits ([C]C/[H]H/[M]M/[L]L)

Audits: [comma-separated list of audit names]
Mode: Quick | Full
Health: Code Quality [grade], Reliability [grade]

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Step 6: Capacity Planning

Track execution times for capacity planning. Append timing data to the summary:

```markdown
### Execution Times

| Audit | Duration | Mode |
|-------|----------|------|
| security | 2m 15s | quick |
| dependency | 1m 30s | quick |
| ... | ... | ... |
| **Total** | **Nm Ns** | |

**Estimated full suite time:** [total quick time × 2.5 for full mode estimate]
```

---

## Domain Mapping

Maps audits to health score domains:

| Domain | Audits |
|--------|--------|
| Security | security, secrets, dependency |
| Code Quality | python, typescript, rust, standards, types, codesweep, deadcode |
| Reliability | resilience, regression, test-quality, pipeline, performance |
| Infrastructure | docker, observability, iac, database, modules |
| Compliance | sbom, licenses, accessibility, browser-compat, api-surface, privacy |

---

## Notes

- **No findings of its own** — audit-suite is pure orchestration
- **No finding prefix** — it has no findings to prefix
- **No archival** — it doesn't produce audit documents (individual audits do)
- **No tech debt integration** — individual audits handle their own
- **Single batch commit** — individual audits run with `--no-commit`; the suite stages and commits all artifacts in one commit after all audits complete (Step 5)
- Suite results are shown to user, not persisted to a file
- Run `dashboard` separately after a suite run for cross-audit correlation

---

## See Also

- `.claude/commands/audit/shared.md` — Tiered scheduling definitions
- `/audit-dashboard` — Cross-audit correlation and health scoring
- `/tmux` — For parallel sprint execution (audit-suite uses parallel agents within each tier)
