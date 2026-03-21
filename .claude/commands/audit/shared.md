# Audit Shared Content

Shared patterns referenced by all `/audit-*` commands (`/audit-python`, `/audit-typescript`, `/audit-rust`).
Loaded via `Read` tool — this is not a slash command.

---

## Flag Parsing

All `/audit-*` commands accept these flags:

| Flag | Behavior |
|------|----------|
| (no flags) | Full audit with agent analysis |
| `--quick` | Tool-only, skip agent analysis (faster, less context) |
| `--since last` | Compare against most recent prior audit, show delta |
| `--sprint` | After audit, chain into `/sprint` with findings as input |
| `--sprint v1.X_name` | Same, but with explicit sprint folder |
| `--fix` | Auto-fix what's safe (ruff --fix, clippy --fix, etc.) |
| `--sarif` | Also produce SARIF v2.1.0 output (machine-readable) alongside Markdown |
| `--no-commit` | Skip the git commit step (used by `/audit-suite` for batch commit) |

### Parse Rules

1. Strip all flags from `$ARGUMENTS`
2. Set boolean variables:
   - `AGENT_MODE` — `true` unless `--quick` is set
   - `COMPARE` — `true` if `--since last` is present
   - `SPRINT_CHAIN` — `true` if `--sprint` is present
   - `AUTO_FIX` — `true` if `--fix` is present
   - `SARIF_OUTPUT` — `true` if `--sarif` is present
   - `SKIP_COMMIT` — `true` if `--no-commit` is present
3. If `--sprint` has a trailing `v*_*` pattern (e.g., `v1.7a_cleanup`), extract as `SPRINT_FOLDER`
4. Remove all detected flags from arguments after parsing

Report to user:
```
Mode: Full | Quick
Flags: --since last, --fix, --sarif, --sprint v1.X_name  (list active flags, or "none")
```

---

## Tool Rules

**Semgrep for AST-aware pattern scans.** Use **Semgrep** (via Bash tool) for pattern
scans that benefit from AST awareness — injection detection, error swallowing,
PII field detection, debug output, type annotation misuse. Semgrep eliminates false
positives from comments, strings, and test fixtures that grep can't distinguish.

```bash
# Run Semgrep with a specific rule file, JSON output for parsing
semgrep --config .semgrep/<ruleset>.yml --json <target_dirs> 2>/dev/null
```

**Semgrep JSON output parsing:**
- `results[]` — array of findings, each with:
  - `check_id` — rule ID (e.g., `python-sql-injection-fstring`)
  - `path` — file path
  - `start.line` / `end.line` — line numbers
  - `extra.message` — human-readable description
  - `extra.metadata` — rule metadata (owasp, phase, category, audit, etc.)
  - `extra.severity` — `ERROR`, `WARNING`, or `INFO`
- `errors[]` — parse errors (file couldn't be analyzed)

**Available rule files (.semgrep/):**

| File | Used By | Covers |
|------|---------|--------|
| `security.yml` | audit-security | SQL injection, XSS, command injection, secrets, TLS, debug mode, traceback, bare except |
| `resilience.yml` | audit-resilience | Error swallowing (Python+TS), broad except, empty catch, timeout coverage, Rust unwrap |
| `standards.yml` | audit-standards | Naming conventions, exception handling, debug output, type annotations |
| `privacy.yml` | audit-privacy | PII field detection, PII logging, localStorage PII |

**Only custom local rules** — do not use Semgrep registry rules (`--config r/...`).

**Tool version discovery:** Include `semgrep --version` output in audit header Tools line.

**Grep tool as fallback.** Use the built-in **Grep tool** directly for patterns that
Semgrep can't express: simple text searches, config string patterns, CORS checks,
contextual proximity checks (e.g., "is `timeout=` within 5 lines of this call?"),
and patterns requiring manual review. DO NOT delegate to Bash agents, Task agents,
or shell `rg`/`grep`. Grep tool is auto-approved (zero user prompts).

Issue multiple Grep/Semgrep calls in a **single parallel message** (all independent — no
dependencies between them). This applies to all language-specific audits that
inherit from this shared file.

**Explore/Task agents — known targets only.** Reserve Explore agents for genuinely
open-ended codebase exploration where you don't know where to look. If the search
target is a known file, pattern, or directory, use inline Grep/Glob/Read directly.
Explore agents cost 20-50k tokens per invocation; a Grep call costs ~200 tokens.

---

## Agent Header Conventions

Note in each audit document header:
- `Agent: Plan` (if Plan agent was used for deep analysis)
- `Agent: Explore` (if Explore agent was used for codebase scanning)
- `Agent: Explore + Plan` (if both agent types were used)
- `Agent: None (--quick)` (if `--quick` flag was set)
- `Agent: None (timeout)` (if agent failed or timed out)

---

## Output Format

Every audit document follows this structure:

```markdown
# [Language] Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_[Language]_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | Explore | None (--quick) | None (timeout)
**Tools:** [list of tools run with versions]

## Executive Summary

[2-3 sentences: what was found, severity distribution, trend vs prior]

## Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| [key metric 1] | X | Y | ↑/→/↓ |
| ... | ... | ... | ... |

## Findings

### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
| ... | ... | ... | ... | ... | ... | ... |

### High

[same table format]

### Medium

[same table format]

### Low / Informational

[same table format]

## Auto-Fixed (when --fix used)

| # | File | What Changed | Tool |
|---|------|-------------|------|
| ... | ... | ... | ... |

## Delta (when --since last used)

### New Findings (not in prior)
[table]

### Resolved (was in prior, now gone)
[table]

### Unchanged (still present)
[count only, not full table]

### Trend Summary
| Metric | Prior | Current | Change |
|--------|-------|---------|--------|
| Total findings | X | Y | +/-N |
| Critical | X | Y | +/-N |
| ... | ... | ... | ... |

## Historical (Resolved)

- [MMDDYY] Fixed: [brief description]

## Recommendations

1. [Highest impact, with estimated effort]
2. [Second priority]
3. [Third priority]
```

---

## SARIF Writer Specification

**Only runs when `SARIF_OUTPUT` is `true` (`--sarif` flag set).**

When `--sarif` is set, produce **both** Markdown (human-readable) and SARIF (machine-readable) from the same findings. The Markdown document is the primary output; the SARIF file is a secondary artifact.

### SARIF v2.1.0 Schema

Output file: `work/MMDDYY_[Language]_Audit.sarif`

**Structure:**

```json
{
  "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/main/sarif-2.1/schema/sarif-schema-2.1.0.json",
  "version": "2.1.0",
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "audit-[type]",
          "version": "1.0.0",
          "informationUri": "https://github.com/your-org/your-project",
          "rules": [ /* rule definitions */ ]
        }
      },
      "results": [ /* finding instances */ ],
      "invocations": [
        {
          "executionSuccessful": true,
          "commandLine": "/audit-[type] --sarif"
        }
      ]
    }
  ]
}
```

### Field Mapping

| Markdown Finding | SARIF Field | Example |
|------------------|-------------|---------|
| Finding prefix + number (e.g., `PY-3`) | `results[].ruleId` | `"PY-3"` |
| Severity (CRITICAL/HIGH/MEDIUM/LOW/INFO) | `results[].level` | See severity mapping below |
| File | `results[].locations[].physicalLocation.artifactLocation.uri` | `"src/backend/api/server.py"` |
| Line | `results[].locations[].physicalLocation.region.startLine` | `42` |
| Issue description | `results[].message.text` | `"Bare except hides errors"` |
| Tool | `results[].properties.tool` | `"ruff"` |
| LOC Impact | `results[].properties.locImpact` | `"~120 LOC"` |
| Effort | `results[].properties.effort` | `"small"` |
| Audit prefix | `runs[].tool.driver.rules[].id` | `"PY"` |

### Severity Mapping (Markdown → SARIF)

| Markdown Severity | SARIF `level` |
|-------------------|---------------|
| CRITICAL | `"error"` |
| HIGH | `"error"` |
| MEDIUM | `"warning"` |
| LOW | `"note"` |
| INFO | `"note"` |

### Rule Definitions

Each unique finding type becomes a rule in `tool.driver.rules[]`:

```json
{
  "id": "PY-3",
  "shortDescription": { "text": "Bare except clause" },
  "defaultConfiguration": { "level": "error" },
  "properties": {
    "category": "exceptions",
    "audit": "audit-python"
  }
}
```

### SARIF Output Rules

1. **Dual output** — always produce Markdown first, then SARIF from the same findings
2. **No findings = valid SARIF** — output `"results": []` with empty array
3. **Relative paths** — use repo-relative paths in `artifactLocation.uri` (no leading `/`)
4. **One run per audit** — each `/audit-*` invocation produces one SARIF run
5. **Categories** — set `runs[].tool.driver.rules[].properties.category` per audit type for GitHub Code Scanning filtering
6. **Commit the SARIF** — `git add work/MMDDYY_[Language]_Audit.sarif` alongside the Markdown

### VS Code SARIF Viewer

For local development, install the [SARIF Viewer](https://marketplace.visualstudio.com/items?itemName=MS-SarifVSCode.sarif-viewer) extension. Open any `.sarif` file from `work/` to see findings inline in the editor.

---

## Severity Levels

Consistent across all languages:

| Severity | Criteria |
|----------|----------|
| CRITICAL | Runtime crash risk, security vulnerability, data loss |
| HIGH | Bug-prone pattern, significant dead code (>200 LOC), cyclomatic complexity 30+, O(n²)+ in hot path |
| MEDIUM | Maintainability issue, moderate dead code (50-200 LOC), cyclomatic complexity 15-29, O(n) where O(1) available in hot path |
| LOW | Style issue, minor dead code (<50 LOC), cyclomatic complexity 11-14, suboptimal complexity on bounded/cold-path input |
| INFO | Suggestion, auto-fixable, cosmetic |

### LOC Impact & Effort Columns

Each finding includes:
- **LOC Impact** — estimated lines of code affected or removable (e.g., `~120 LOC`, `3 files`)
- **Effort** — estimated cleanup effort: `trivial` (<5 min), `small` (5-30 min), `medium` (30-120 min), `large` (>2 hours)

---

## Delta Tracking

### Finding Prior Audit

```bash
# Find most recent prior audit for this language
PRIOR=$(ls work/*_[Language]_Audit.md 2>/dev/null | sort -r | head -1)
```

If no prior found, set `Prior: None` in header and skip delta section.

### Comparison Process

1. Parse prior findings tables — extract `(file, line, issue)` tuples
2. Run current scan — produce current `(file, line, issue)` tuples
3. Classify each finding using progressive summarization annotations (below)
4. Produce delta section with counts and trend arrows

### Progressive Summarization (Canonical)

**This is the single canonical definition.** Both audit and EOD commands use these annotations.

Annotate every finding in the output document:

| Annotation | Meaning | Action |
|------------|---------|--------|
| `NEW` | In current scan but not in prior (file + issue match) | Full detail in Current Findings |
| `CARRIED` | Unchanged from prior, verified still present | Keep in Current Findings, verify exists in codebase |
| `CHANGED` | In prior but details differ (e.g., line count grew, severity shifted) | Update detail, note what changed |
| `RESOLVED` | In prior but no longer present | Move to Historical as: `- [MMDDYY] Fixed: [brief description]` |

**Historical section management:**
- Old historical items (>2 cycles old) → compress or remove
- Keep only last 2 cycles in Historical to prevent unbounded growth

### Trend Arrows

- `↑` (regressing) = total findings increased
- `→` (stable) = total findings unchanged
- `↓` (improving) = total findings decreased

---

## Archival

Before creating a new audit document, promote the prior to `docs/eod-docs/`:

```bash
PRIOR=$(ls work/*_[Language]_Audit.md 2>/dev/null | sort -r | head -1)
if [ -n "$PRIOR" ]; then
  git mv "$PRIOR" docs/eod-docs/
fi
```

**Notes:**
- Priors promote to `docs/eod-docs/` (flat directory, same filename)
- If a file with the same name already exists in `docs/eod-docs/`, the `git mv` overwrites it (newer prior replaces older)
- Uses `git mv` so history is preserved
- Archival runs before the new audit document is written

---

## Tech Debt Integration

After producing findings, update `.claude/rules/tech-debt-patterns.md`:

### New Findings (CRITICAL or HIGH)

1. Read `.claude/rules/tech-debt-patterns.md`
2. Check if the finding already has an entry (grep for file path + issue keywords)
3. If new, append:

```markdown
### [Short description]
**Added:** YYYY-MM-DD | **Source:** /audit-[language]
[1-2 lines describing the issue and resolution criteria]
**Location:** [file path]
**Resolve:** [specific action needed]
```

### Resolved Findings

If any existing tech-debt entries match findings that are now RESOLVED (confirmed gone from codebase):
1. Mark the entry with `~~strikethrough~~` and add `RESOLVED` + date
2. Or remove the entry entirely if the file has many entries

**Skip tech debt updates if:**
- No new CRITICAL/HIGH findings AND nothing resolved
- `tech-debt-patterns.md` is at its 100-line cap (evaluate oldest entries for removal before appending)

---

## Sprint Chain

**Only runs when `SPRINT_CHAIN` is `true` (`--sprint` flag set).**

### Determine Sprint Folder

**If `SPRINT_FOLDER` was set during flag parsing** (user provided `--sprint v1.X_name`), use it.

**If not set** (bare `--sprint`), auto-increment:
```bash
LATEST=$(ls -d work/v* work/00_backup/v* 2>/dev/null | sed 's|.*/||' | sort -Vu | tail -1)
# Extract version prefix (v1.7), letter (a), bump letter
# New folder: v{version}{next_letter}_audit_remediation
```

### Build Sprint Context

1. Extract all CRITICAL and HIGH findings from the audit
2. Build a focus string:

```
Audit remediation sprint: [language] audit found [N] critical + [M] high findings.

Top targets:
1. [file:line] — [issue] (CRITICAL) — Effort: [effort]
2. [file:line] — [issue] (HIGH) — Effort: [effort]
3. [file:line] — [issue] (HIGH) — Effort: [effort]
...up to 10

Full audit: work/MMDDYY_[Language]_Audit.md
```

3. Invoke `/sprint {SPRINT_FOLDER_SHORTHAND}` with the focus string
4. `/sprint` will create the overview and first handoff as usual

### Post-Chain Summary

After `/sprint` completes:
```
## Sprint Chain Complete

**Audit** -> **Sprint** chain finished.

- Audit report: work/MMDDYY_[Language]_Audit.md
- Sprint overview: work/{folder}/00_{topic}_overview_v4.md
- First handoff: work/{folder}/01_{session}_handoff.md

Remediation sessions planned for [N] critical + [M] high findings.
Ready for GO/NO-GO on Session 01?
```

---

## Commit Format

**Skip this section entirely if `SKIP_COMMIT` is true (`--no-commit` flag).** When skipped, leave files unstaged — the caller (e.g., `/audit-suite`) will handle staging and committing.

```bash
git add work/MMDDYY_[Language]_Audit.md
git add work/MMDDYY_[Language]_Audit.sarif  # if --sarif
git add docs/eod-docs/  # if priors promoted
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): [language] audit — [N] findings ([critical] critical, [high] high)

Tools: [tool list with versions]
Mode: Full | Quick
Delta: [+N new, -N resolved] (if --since last)
SARIF: work/MMDDYY_[Language]_Audit.sarif (if --sarif)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

**Notes:**
- Language in commit message is lowercase: `python`, `typescript`, `rust`
- Include delta line only when `--since last` was used
- Include auto-fix summary if `--fix` was used: `Auto-fixed: [N] issues`
- Use HEREDOC for multi-line commit message

---

## Completion Report

Output to user after commit:

```
## [Language] Audit Complete

**Date:** MMDDYY
**Mode:** Full | Quick
**Agent:** [type used]
**Tools:** [list with versions]

### Findings Summary
| Severity | Count | Delta |
|----------|-------|-------|
| Critical | N | +/-N |
| High | N | +/-N |
| Medium | N | +/-N |
| Low | N | +/-N |
| **Total** | **N** | **+/-N** |

### Top 3 Findings
1. [file:line] — [issue] (CRITICAL)
2. [file:line] — [issue] (HIGH)
3. [file:line] — [issue] (HIGH)

### Output
- work/MMDDYY_[Language]_Audit.md
- [promoted prior to docs/eod-docs/ if applicable]

[If --fix used: "Auto-fixed N issues. See Auto-Fixed section in audit doc."]
[If --sprint: "Chaining into /sprint with N critical/high findings..."]
```

**Delta column:**
- Show `+/-N` only when `--since last` was used
- Otherwise show `—` or omit the column

---

## Error Handling

| Error | Action |
|-------|--------|
| Tool not installed | Print install command, skip that tool's phase, note in output under Tools |
| Tool exits with error | Capture stderr, include in output, continue with other tools |
| No prior audit found | Skip delta section, set `Prior: None` in header |
| Agent timeout | Fall back to tool-only analysis, set `Agent: None (timeout)` |
| Prior already in `docs/eod-docs/` | `git mv` overwrites — newer prior replaces older |
| Tech debt file at cap | Evaluate oldest entries for removal before appending |
| `--fix` changes break lint | Re-run lint after fix, report both states (before/after) |
| `--sprint` but audit has 0 critical/high | Skip sprint chain — nothing to remediate. Show "No critical/high findings — sprint chain skipped" |
| `--sprint` but `/sprint` fails | Report sprint failure — audit report is still valid and committed |
| `--sprint` without folder and no prior sprints | Error — cannot auto-increment. User must provide explicit folder shorthand |
| Git commit fails | Show error, leave files staged, ask user to resolve |

---

## Execution Checklist

Before completing, verify:

- [ ] Flags parsed correctly (`--quick`, `--since last`, `--sprint`, `--fix`, `--sarif`)
- [ ] Prior audit found and read (or noted as "None")
- [ ] Prior audit promoted to `docs/eod-docs/` with `git mv` (if it existed)
- [ ] All language-specific tools run (or skipped with install note)
- [ ] Agent analysis run (unless `--quick` or timeout)
- [ ] Agent header field set correctly in document
- [ ] Findings classified by severity (CRITICAL/HIGH/MEDIUM/LOW/INFO)
- [ ] LOC Impact and Effort columns populated for each finding
- [ ] Delta section populated (if `--since last`)
- [ ] Auto-Fixed section populated (if `--fix`)
- [ ] Tech debt patterns updated (if new CRITICAL/HIGH or resolved entries)
- [ ] Progressive summarization applied (new/unchanged/resolved findings)
- [ ] Historical section includes resolved items from prior
- [ ] Git commit created with correct format (skipped if `--no-commit`)
- [ ] Completion report shown to user
- [ ] SARIF file produced with valid v2.1.0 structure (if `--sarif`)
- [ ] Sprint chain invoked (if `--sprint`)

---

## Verification Step (Phase 5.5)

After Phase 5 (analysis) and before assembling the findings document, verify each finding is real.

For each finding from Phase 5:
1. **Read the file** at the reported line number using the Read tool
2. **Confirm the finding is real** — not inside a comment, string literal, test fixture, or already-fixed code
3. **Check recency** — `git log -1 -- <file>` to see if the file was recently modified (finding may be stale)
4. **Assign confidence:**
   - `HIGH` — confirmed in source code, issue is real
   - `MEDIUM` — pattern match is plausible but context is ambiguous
   - `LOW` — likely false positive (in comment, string, test, or already fixed)
5. **Drop LOW confidence findings** — do not include in output

Skip this step if `--quick` is set.

**Why this matters:** Grep pattern matches produce false positives — hits inside comments, string literals, test fixtures, or code that was already refactored. The find→verify→rank pattern (from Claude Code Review) cuts noise before it reaches the user.

---

## Cross-Audit Correlation

When the same file appears in findings from multiple audits:

1. **Note the cross-reference** using finding prefixes: `See also: PF-3 in /audit-performance (same file)`
2. **Bump severity by one level** if 3+ audits flag the same file (e.g., MEDIUM → HIGH)
3. **Use finding prefixes** (see Finding Prefix Registry below) for unambiguous cross-references
4. **Do not deduplicate** — each audit reports its own perspective. Cross-references add context, not replace findings.

When writing findings, check if the file appeared in the most recent prior run of related audits. If you have access to prior audit documents in `work/`, scan them for the same file paths.

---

## Finding Prefix Registry

Every audit uses a unique 2-3 letter prefix for finding IDs. Use `PREFIX-N` format (e.g., `BN-3`, `PY-12`).

| Audit | Prefix |
|-------|--------|
| accessibility | AC- |
| api-surface | AS- |
| ~~bestpractices~~ | ~~BP-~~ (MERGED into standards, 2026-03-15) |
| ~~bottleneck~~ | ~~BN-~~ (MERGED into performance, 2026-03-15) |
| browser-compat | BC- |
| codesweep | CS- |
| ~~component-health~~ | ~~CH-~~ (MERGED into typescript Phase 5, 2026-03-15) |
| dashboard | DA- |
| database | DB- |
| deadcode | DC- |
| dependency | DEP- |
| ~~dependency-vuln~~ | ~~DV-~~ (MERGED into dependency, 2026-03-12) |
| docker | DK- |
| iac | IC- |
| licenses | LI- |
| standards (née linting) | LT- |
| observability | OB- |
| performance | PF- |
| pipeline | PL- |
| privacy | PR- |
| project-overview | PO- |
| python | PY- |
| regression | RG- |
| resilience | RE- |
| rust | RS- |
| security | SC- |
| system-walkthrough | SW- |
| techdebt | TD- |
| test-quality | TQ- |
| typescript | TS- |
| modules | MD- |
| sbom | SB- |
| secrets | SE- |
| types | TY- |

**Rules:**
- Prefixes are permanent — do not reassign
- New audits added in future sessions get the next available 2-3 letter prefix
- When merging audits, the surviving audit keeps its prefix; the merged audit's prefix is retired with a strikethrough note
- **Merge history:**
  - DV- (dependency-vuln) → merged into DEP- (dependency), 2026-03-12 (v2.0y_audits Session 03)
  - BP- (bestpractices) → merged into LT- (standards, née linting), 2026-03-15 (v2.1o_audits Session 01)
  - CH- (component-health) → merged into TS- (typescript Phase 5), 2026-03-15 (v2.1o_audits Session 01)
  - BN- (bottleneck) → merged into PF- (performance Phase 2), 2026-03-15 (v2.1o_audits Session 01)

---

## Overlap Cross-Reference Table

Intentional overlaps between audits. Each overlap exists because the two audits examine the same code from different perspectives. Do not deduplicate — document the overlap so future audit development doesn't accidentally create new ones.

| Audit A | Audit B | Shared Pattern | Resolution |
|---------|---------|---------------|------------|
| python P3 | deadcode | vulture | Intentional — different severity mapping |
| python P4 | standards | ruff | Standards adds trend tracking |
| python P7 | performance | algorithmic complexity | Shared grep patterns via shared.md |
| security A3 | secrets | credential patterns | Security checks code; secrets checks git history |
| security A9 | observability | PII logging | Security checks exposure; observability checks coverage |
| resilience P1 | regression | swallowed errors | Resilience checks patterns; regression checks history |

---

## Shared Grep Patterns

Patterns used by multiple audits. Reference by category name instead of duplicating regex across audit files.

### Dead Code Patterns
- Unused imports (Python): `^import .+` cross-referenced with usage count = 0
- Unused imports (TS): `^import .+` with no references beyond the import line
- Unused variables: `_[a-z]` prefix convention (Python), `^const .+ = .+` with zero references (TS)
- Unreachable code: `return\s` followed by statements at same indent level

### Error Handling Patterns
- Bare except (Python): `except\s*:` without specific exception type
- Swallowed errors (Python): `except.*:\s*pass$`
- Empty catch (TS): `catch\s*\(.*\)\s*\{\s*\}` — catch block with no body
- Silent error drop: `catch` block that doesn't log, re-throw, or return error state

### Performance Patterns
- Sequential awaits: `await .*\n\s*await ` outside `Promise.all` / `asyncio.gather` context
- List membership (Python): `if .+ in \[` — should be `set` for >3 elements
- String concat in loop: `\+=\s*["']` or `\+=\s*f["']` inside `for`/`while`
- Unbounded query: `SELECT .+ FROM` without `LIMIT` or pagination

### Type Safety Patterns
- Any type (TS): `:\s*any` — explicit any annotation
- Type ignore (Python): `# type:\s*ignore`
- Untyped function (Python): `def \w+\([^)]*\)\s*:` without `->` return annotation
- Type assertion (TS): `as any` — casting to any

### Security Patterns
- Hardcoded secrets: `(?i)(password|secret|api_key|token)\s*=\s*["'][^"']+["']`
- Unvalidated input: `request\.(args|form|json)\[` without validation
- SQL string formatting: `f".*SELECT.*\{` or `"SELECT.*" %` — potential injection
- Disabled TLS verify: `verify\s*=\s*False`

---

## Tiered Scheduling

Audits are grouped into tiers by frequency. Higher tiers run more often — they cover fast-moving, high-risk targets. Lower tiers cover slow-changing infrastructure and compliance concerns.

### Schedule Table

| Tier | Frequency | Audits | Rationale |
|------|-----------|--------|-----------|
| 1 | Every sprint | security, dependency, secrets, standards, test-quality | High-risk, fast-moving targets |
| 2 | Biweekly | python, typescript, rust, codesweep, resilience, deadcode, regression, performance | Code quality keeps pace with development |
| 3 | Monthly | docker, observability, pipeline, privacy, modules, types, iac, database | Infrastructure changes slowly |
| 4 | Quarterly | sbom, licenses, accessibility, browser-compat, api-surface | Compliance + low-velocity concerns |
| On-demand | As needed | dashboard, project-overview, system-walkthrough, techdebt | Meta/context — run when planning or reviewing |

### Tier Details

**Tier 1 — Every Sprint (5 audits)**
Run at the start of every sprint. Use `--quick` in CI (PR gates). Full mode for scheduled runs.
- `security` — OWASP vulnerability patterns, injection, XSS, secrets in code
- `dependency` — CVE scanning (pip-audit, cargo audit), outdated packages
- `secrets` — gitleaks scan for credentials in code and git history
- `standards` — ruff + oxlint rule compliance, naming conventions
- `test-quality` — test inventory, coverage gaps, flaky test indicators

**Tier 2 — Biweekly (8 audits)**
Run every other sprint. Keeps code quality aligned with development velocity.
- `python` — ruff linting, complexity, dead code (vulture), import analysis
- `typescript` — oxlint, React patterns, component health, bundle analysis
- `rust` — clippy, cargo audit, cargo deny, unsafe usage
- `codesweep` — code duplication analysis (jscpd)
- `resilience` — error handling, timeout patterns, retry logic
- `deadcode` — unused code detection across all languages
- `regression` — danger zone mapping, swallowed errors, test coverage gaps
- `performance` — algorithmic complexity, file churn, bottleneck detection

**Tier 3 — Monthly (8 audits)**
Run once per month. Infrastructure and configuration change slowly.
- `docker` — Dockerfile best practices, image vulnerabilities (Trivy), compose config
- `observability` — logging coverage, metrics endpoints, PII in logs
- `pipeline` — CI/CD configuration, build scripts, deployment readiness
- `privacy` — PII detection, consent handling, data retention
- `modules` — module boundaries (tach/knip), dependency graph health
- `types` — ty + pyright (Python), tsc --noEmit (TypeScript), type coverage
- `iac` — Infrastructure-as-Code validation (Caddy, Prometheus, LiveKit, docker-compose)
- `database` — schema validation, migration health, query patterns

**Tier 4 — Quarterly (5 audits)**
Run once per quarter. Compliance and low-velocity concerns.
- `sbom` — Software Bill of Materials generation (syft), vulnerability scan (grype)
- `licenses` — license compatibility analysis across all dependencies
- `accessibility` — WCAG compliance (jsx-a11y, axe-core), ARIA patterns
- `browser-compat` — cross-browser compatibility, API usage analysis
- `api-surface` — API contract validation, breaking change detection

**On-Demand (4 audits)**
Not scheduled — run when planning sprints, onboarding, or reviewing architecture.
- `dashboard` — aggregates recent audit results into composite health score
- `project-overview` — codebase summary for onboarding and planning
- `system-walkthrough` — end-to-end data flow documentation
- `techdebt` — tech debt inventory and prioritization

### Quick Mode Support

All audits support `--quick` via shared flag parsing. In quick mode:
- Agent analysis is skipped (tool-only, faster)
- Verification step (Phase 5.5) is skipped
- Useful for CI/PR gates and rapid spot checks

### Recommended Cadence

| Context | What to Run | Mode |
|---------|-------------|------|
| Every PR | Tier 1 (via `audit-pr.yml`) | `--quick` |
| Sprint start | Tier 1 full + Tier 2 `--quick` | Mixed |
| Biweekly review | Tier 2 full | Full |
| Monthly review | Tier 3 full + `dashboard` | Full |
| Quarterly review | Tier 4 full + `dashboard` | Full |
| Pre-release | All tiers `--quick` | `--quick` |
| Post-incident | `security` + `regression` + `resilience` | Full |

---

## Execution Notes

- **Default is full audit** — agents + tools. Use `--quick` for tool-only fast scans
- **Audit documents go to `work/`** — named `MMDDYY_[Language]_Audit.md`
- **Prior audits are promoted** to `docs/eod-docs/` before writing new ones
- **Severity is consistent** across all three languages — same thresholds, same table format
- **Delta comparison key:** `(file, issue)` pair — line numbers may shift between audits
- **Trend arrows:** `↓` = improving, `↑` = regressing, `→` = stable
- **Tech debt integration is automatic** — CRITICAL/HIGH findings append to `tech-debt-patterns.md`
- **Sprint chain:** `--sprint` triggers remediation sprint planning after audit completes
- **Memory persistence:** Each language-specific command handles its own velocity tracking
- **SARIF output:** `--sarif` produces `work/MMDDYY_[Language]_Audit.sarif` alongside Markdown. SARIF v2.1.0 for GitHub Code Scanning + VS Code SARIF Viewer
- **Flag combinations are valid:** `--quick --since last`, `--fix --since last`, `--sprint --since last --fix`, `--sarif --quick`, `--sarif --since last`
- **eod-docs integration:** `/eod-docs` Day 1 reads audit documents from `work/` for tech debt assessment
