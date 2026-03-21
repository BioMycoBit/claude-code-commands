# Sprint End Skill

Close out a completed sprint with retrospective insights and archive to backup.

## Arguments

- `$ARGUMENTS` - Optional: Sprint folder name (e.g., `v1.5s_agents`). If not provided, auto-detect from current working folder or recent work.

---

## Step 1: Gather All Context (PARALLEL)

### Resolve Sprint Folder

If `$ARGUMENTS` provided → `work/$ARGUMENTS/`
Otherwise → the active sprint folder: `work/v*/` (there should be exactly one active sprint folder at a time).

### Do These in Parallel (single message, multiple tool calls)

1. **Read overview doc**: Glob for `work/<sprint>/00_*.md` — this has the Progress Tracker with all session statuses, dates, and deliverables. This is your primary source of truth.
2. **Git metrics** (single chained command). Derive `<tag>` from the sprint folder name — e.g., `v1.6h_codesweep_audit` → `v1.6h`:
   ```bash
   echo "=== COMMITS ===" && \
   git log --oneline --grep="<tag>" && \
   echo "=== COMMIT COUNT ===" && \
   git log --oneline --grep="<tag>" | wc -l && \
   echo "=== DATES ===" && \
   git log --format="%ad" --date=short --grep="<tag>" | sort -u
   ```
   **CRITICAL: Use `--grep` (commit message filtering), NOT `--` path filtering.** Path filtering (e.g., `-- "src/"`) matches every commit that ever touched that directory — potentially the entire project history. `--grep="v1.6h"` reliably matches only commits tagged with the sprint prefix per your commit convention.
   For files changed / lines added stats, extract from the overview doc's session notes (lines eliminated per session) — this is more accurate than git diff when commits from multiple sprints are interleaved on the same branch.
3. **Read memory files** that need updating: `memory/velocity.md` and `.claude/rules/tech-debt-patterns.md`

**DO NOT read individual handoff docs.** The overview's Progress Tracker already summarizes each session. Only read a specific handoff if the overview is missing critical info (rare).

---

## Step 2: Check for Blockers

Before proceeding, verify:

- **All sessions complete?** Check Progress Tracker for any non-✅ items. If incomplete sessions exist → warn user and ask whether to proceed.
- **Already archived?** Check `work/00_backup/<sprint>/` doesn't already exist.
- **Uncommitted changes?** Check `git status -- "work/<sprint>/"` for unstaged work. If found → warn user.

If any blocker → stop and ask. Otherwise proceed.

---

## Step 3: Update Overview Doc + Write Memory (PARALLEL)

Do both of these in parallel:

### A. Add Sprint Closure Section to Overview

Insert **immediately after the Progress Tracker** table (before Executive Summary). Update the sprint status from "IN PROGRESS" to "COMPLETE" in the header.

```markdown
---

## Sprint Closed: [DATE]

**Status:** COMPLETE
**Sessions:** [N] of [N] planned (+ [M] sidequests)
**Duration:** [start date] → [end date]

### Deliverables

- [Bulleted list extracted from Progress Tracker notes column]

### Key Metrics

| Metric | Value |
|--------|-------|
| Files changed | [from git diff --stat] |
| Lines added/removed | [+X / -Y from git diff --stat] |
| Commits | [count] |

### Retrospective

#### What Went Well
- [2-3 bullets — analyze from: clean handoff chain? all sessions complete? no rework?]

#### What Could Improve
- [1-2 bullets — be honest: scope creep? sessions that needed rework?]

#### Lessons Learned
- [1-2 actionable insights for future sprints]

### Tech Debt Incurred

| Item | Location | Cleanup Criteria |
|------|----------|------------------|
| [from handoff docs, or "None — clean sprint"] | | |

---

*Sprint archived to `work/00_backup/<sprint>/`*
```

### B. Update Memory Files

**velocity.md** — Append sprint summary:
```markdown
### [Sprint Name]
**Closed:** [Date] | **Sessions:** [N] completed | **Duration:** [X] days
**Key deliverables:** [1-line summary]
```

**tech-debt-patterns.md** — Mark resolved items with `~~strikethrough~~ RESOLVED` and add any new debt discovered during sprint.

**MEMORY.md** — Only append if retrospective surfaced a genuinely new behavioral insight. Don't force it.

---

## Step 4: Archive Sprint + Commit

**CRITICAL: Archive path is `work/00_backup/<sprint>/` where `<sprint>` is the EXACT sprint folder name (e.g., `v1.6k_remediation`).** Do NOT nest under version prefixes like `work/00_backup/v1.2/v1.6k_remediation/`. Verify by checking existing archives: `ls -d work/00_backup/v1.6*/` — they are all flat at the top level.

```bash
# Create backup location — FLAT, directly under 00_backup/
mkdir -p work/00_backup/<sprint>

# Move sprint files (preserves git history)
git mv work/<sprint>/* work/00_backup/<sprint>/

# Remove empty folder
rmdir work/<sprint>

# Stage everything
git add work/00_backup/<sprint>/ memory/velocity.md .claude/rules/tech-debt-patterns.md

# Commit
git commit -m "$(cat <<'EOF'
docs(sprint): close <sprint> - [1-line summary]

Sessions: [N] completed
Deliverables: [top 3-4 items]
Archived to: work/00_backup/<sprint>/

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

If `memory/MEMORY.md` was updated, add it to the staging too.

---

## Step 5: Completion Report

Output to user:

```
## Sprint Closed: <sprint>

**Status:** COMPLETE
**Archived to:** work/00_backup/<sprint>/

### Summary
- [N] sessions completed [+ M sidequests]
- [Key deliverables, 3-5 bullets]

### Metrics
- [X] files changed, +[Y] / -[Z] lines, [N] commits

### Retrospective Highlights
- **Well:** [top item]
- **Improve:** [top item]
- **Learned:** [top insight]

### Next Steps
- [ ] [Follow-up items if any]
- [ ] Start next sprint with `/sprint [topic]`

### Commit
[hash] docs(sprint): close <sprint> - [summary]
```

---

## Error Handling

| Condition | Action |
|-----------|--------|
| Sprint folder not found | List `work/*/00_*overview*.md` and ask user to pick |
| Incomplete sessions | Warn + ask "Close anyway?" |
| Already archived | Error: "Already at `work/00_backup/<sprint>/`" |
| Uncommitted changes | Warn + ask "Commit first or proceed?" |

---

## Anti-Patterns (DON'T DO THIS)

- **Don't nest archives under version prefixes** — The archive path is `work/00_backup/<sprint_folder_name>/`, NOT `work/00_backup/v1.2/<sprint>/` or any other nested path. The `<sprint>` placeholder is the exact folder name from `work/` (e.g., `v1.6k_remediation`), not a sub-path. All archives live flat under `00_backup/`.
- **Don't use `git log -- "src/"` or other broad path filters** — This matches every commit that ever touched that directory, potentially returning 200+ commits from the entire project. Always use `--grep="<sprint-tag>"` to filter by commit message.
- **Don't read every handoff doc** — The overview Progress Tracker has the summary. Reading 7 handoff docs wastes context.
- **Don't use `--since="30 days ago"`** — Fragile. Use `--grep` instead.
- **Don't gather metrics and context in separate sequential steps** — Parallel tool calls.
- **Don't ask the user questions you can answer from the overview** — Session count, dates, deliverables are all in the Progress Tracker.

---

## See Also

**Workflow chain:** `/sprint` → `/implement` → `/handoff-end` → `/sprint-end`

- `/sprint` — Create the sprint plan that this skill closes
- `/report` — Git activity report (report covers daily/weekly activity, sprint-end covers sprint closure)
- `/backup` — Archive sprint folders into version subfolders (runs after sprint-end)
