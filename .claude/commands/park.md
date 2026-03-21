# Park Skill

Temporarily set aside a handoff with context for returning later. Updates both the handoff doc and overview.

## Usage

```
/park [handoff_doc] [reason]
```

**Examples:**
```
/park                                    # Park current handoff (from context)
/park 03_connection_states_handoff.md    # Park specific handoff
/park "waiting on PC2 testing"           # Park with reason
```

---

## Step 1: Identify Handoff Document

**Priority order:**

### 1a: Conversation Context (USE THIS)
If you just ran `/implement`, you already know the handoff doc.

### 1b: Arguments
If user provided: `/park 03_connection_states_handoff.md`

### 1c: Ask User
If unclear:
```
Which handoff should I park?
```

---

## Step 2: Identify Overview Document

Find the `00_*.md` file in the same folder as the handoff:

```bash
# If handoff is work/v1.5r_404/03_connection_states_handoff.md
Glob("00_*.md", path="work/v1.5r_404")
```

**Ask user if multiple found or none obvious.**

---

## Step 3: Update Handoff Document

Add/update status header at top of file (after title):

```markdown
# Session 03: Connection States

**Status:** 🅿️ PARKED
**Parked:** [Today's Date]
**Reason:** [User's reason or "Context switch"]
**Return Point:** [Where to resume - step number, line of code, or description]

---
```

**Return Point Detection:**
- If you were mid-implementation: note the step/file you were working on
- If not started: "Start from beginning"
- If blocked: note what's blocking

**Update footer:**
```markdown
---

*Session 03 of v1.5r_404 sprint. Previous: 02. Next: 04.*
*🅿️ Parked [Today's Date] - [reason]*
```

---

## Step 4: Update Overview Document

Edit the Progress Tracker table:

### Change Status
```markdown
| 03 - Connection States | 🅿️ PARKED | Feb 2, 2026 | [reason] |
```

### Update Next Action (if this was NEXT)
```markdown
## Next Action

**Parked:** Session 03 - Connection States
**Reason:** [reason]
**Resume with:** `/implement @work/v1.5r_404/03_connection_states_handoff.md`
```

---

## Step 4.5: Update Memory

### Debugging Pattern (if applicable)

If the session was parked due to a technical blocker, append to
`.claude/rules/debugging-patterns.md`:

```markdown
### [Short description of blocker]
**Added:** [Date]
[1-2 lines: what blocked and what's needed to unblock]
```

**Skip if the park reason is non-technical (scope change, priority shift, etc.)**

---

## Step 5: Commit

```bash
git add [handoff_doc] [overview_doc] .claude/rules/debugging-patterns.md

git commit -m "docs([sprint]): park session [ID] - [reason]

- Mark Session [ID] as PARKED
- Return point: [brief description]

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Step 6: Summary

```
## Session Parked

**Handoff:** [handoff_doc]
**Reason:** [reason]
**Return Point:** [where to resume]

### Updated
- [handoff_doc] - Added PARKED status
- [overview_doc] - Updated progress tracker

### To Resume
```
/implement @[handoff_doc]
```

The handoff doc contains your return point context.
```

---

## Status Legend

| Status | Meaning |
|--------|---------|
| ✅ COMPLETE | Done, success criteria met |
| ⏳ NEXT | Queued as next to work on |
| PENDING | Not started, not queued |
| BLOCKED | Cannot proceed (dependency) |
| 🅿️ PARKED | Intentionally paused, will return |

**PARKED vs BLOCKED:**
- PARKED = voluntary pause (context switch, lower priority, waiting for feedback)
- BLOCKED = involuntary stop (missing dependency, external blocker)

---

## Key Principles

1. **Capture return context** - Future you needs to know where to resume
2. **Update both docs** - Handoff AND overview stay in sync
3. **Reason matters** - "Why parked" helps prioritize later
4. **Easy resume** - `/implement` should just work when returning
