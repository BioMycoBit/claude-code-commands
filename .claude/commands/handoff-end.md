# Handoff End Skill

End-of-session documentation workflow: update overview, create next handoff, commit.

## Usage

```
/handoff-end
```

---

## Step 1: Identify Sessions (CONTEXT FIRST)

**Priority order:**

### 1a: Conversation Context (USE THIS)
If you just ran `/implement`, you already know:
- Completed session = the one you just implemented
- Next session = what you recommended ("Ready for 12b")

**Do not run git commands or file searches if you have this context.**

### 1b: Handoff Doc Footer
Every handoff doc ends with: `*Previous: X. Next: Y.*`

### 1c: Ask User
If still unclear:
```
Which session just completed?
What's the next session?
```

---

## Step 2: Identify Docs to Update

**If you need to search for the overview doc, use case-insensitive patterns (Linux is case-sensitive):**

```bash
# Search for both "overview" and "Overview" variants
Glob("**/*[Oo]verview*.md", path="work/[sprint_folder]")

# Or search for all .md files in the sprint folder and filter
Glob("**/*.md", path="work/[sprint_folder]")
```

**Ask user if not obvious from context:**

```
Which docs should I update?
- Overview doc (e.g., work/v1_users/00_overview.md)?
- Sprint/planning doc (e.g., work/010526_WeekSprint.md)?
```

**Only read files you don't already have in context.**

---

## Step 3: Update Overview Document

Edit the overview doc:

### Progress Tracker Table
```markdown
| [Completed] - [Title] | ✅ COMPLETE | [Today's Date] | [Brief description] |
| [Next] - [Title] | ⏳ NEXT | - | [Brief description] |
```

### Next Action Section
```markdown
## Next Action

**Start Session:** `[next_session]_[title].md`
```

### Phase Map / Version (if applicable)
- Mark phase complete if this was last session in phase
- Increment version if phase complete

---

## Step 4: Create Next Handoff

Create `work/<sprint>/[next_session]_[title].md`:

```markdown
# Session [ID]: [Title]

## SINGLE PURPOSE GOAL
[One sentence from sprint plan]

## BLAST RADIUS
Files I WILL touch:
- [file1]
- [file2]

Files I MUST NOT touch:
- [protected files]

## CONTEXT
[Background from sprint plan, previous session]

## IMPLEMENTATION

### 1. [First Step]
[Details or code]

### 2. [Second Step]
[Details]

## SUCCESS CRITERIA
- [ ] [Criterion 1]
- [ ] [Criterion 2]
- [ ] Linters pass

## STOP CONDITIONS
- [Scope creep item 1]
- [Scope creep item 2]

## TECH DEBT INCURRED
- [ ] None
<!-- Fill if shortcuts taken: inline styles, wrapper divs, TODO comments, etc. -->

## EFFORT
~[X] hour(s)

## DEPENDENCIES
- [Previous session] complete ✅

---

*Session [ID] of <sprint>. Previous: [completed]. Next: [following].*
```

---

## Step 5: Update Memory (only if something new was learned)

**Skip this step entirely if the session was routine with no new discoveries.**

If debugging uncovered a new failure pattern → append to `.claude/rules/debugging-patterns.md`.
If a codebase quirk was discovered → append to `.claude/rules/codebase-quirks.md`.

**Do NOT write per-session velocity entries** — `/sprint-end` handles sprint-level velocity tracking.

---

## Step 6: Commit (Scoped to This Session Only)

**CRITICAL:** With parallel sprints, other modified files may be staged or in the working tree. Only commit files from THIS session.

### 6a: Collect Session Files

Build the explicit file list from conversation context:

1. **Implementation files** — every file in the handoff's Blast Radius that was actually modified (check `git diff --name-only` against blast radius)
2. **Documentation files** — the overview doc updated in Step 3, the new handoff created in Step 4
3. **Memory files** — only if updated in Step 5 (e.g., `debugging-patterns.md`, `codebase-quirks.md`)

### 6b: Commit with --only

Use `git commit --only` to commit **exactly** the session files, ignoring anything else in the staging area:

```bash
# List the files explicitly — never use "git add ."
git add [overview doc] [new handoff] [implementation file 1] [implementation file 2] ...
# Only add memory files if they were actually changed:
# git add .claude/rules/debugging-patterns.md .claude/rules/codebase-quirks.md

git commit --only [overview doc] [new handoff] [implementation file 1] [implementation file 2] ... \
  -m "docs(<sprint>): mark [completed] complete + add [next] handoff

- Mark Session [completed] complete
- Add [next] handoff document

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

**Rules:**
- **Never `git add .`** — always name files explicitly
- **Always use `--only`** — commits exactly those files regardless of staging area state
- **Verify before commit** — run `git diff --cached --name-only` to confirm only session files are staged

---

## Step 7: Summary

```
## Handoff Complete

**Completed:** Session [ID] - [Title]
**Next:** Session [next_ID] - [Title]

### Updated
- [overview doc]
- [sprint doc]

### Created
- [next handoff]

Ready for next session? Run:
`/implement @work/<sprint>/[next_session].md`
```

---

## Key Principles

1. **Context first** - Use what you know from the conversation
2. **Ask if unsure** - Don't guess doc names, ask user
3. **Skip redundant reads** - If you already have a file in context, don't re-read it
4. **No hardcoded paths** - Doc naming varies by project/sprint

---

## See Also

**Workflow chain:** `/sprint` → `/implement` → `/handoff-end` → `/sprint-end`

- `/implement` — Execute the handoff document this skill creates
- `/sprint-end` — Close the entire sprint after all sessions complete
- `/park` — Temporarily set aside a session instead of closing it
