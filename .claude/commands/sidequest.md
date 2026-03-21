# Sidequest Skill

Create a sub-task handoff that branches from a parked session. Uses hierarchical numbering (02a, 02b) to track quick fixes that unblock the parent.

## Usage

```
/sidequest [parent_session] [description]
```

**Examples:**
```
/sidequest 02 "fix env loading on Mac"     # Creates 02a_
/sidequest 05 "add missing dependency"      # Creates 05a_
/sidequest                                  # Uses most recent parked session
```

---

## Naming Convention

**Pattern:** `{parent_number}{letter}_{short_name}_handoff.md`

| Parent | Sidequest | Name |
|--------|-----------|------|
| 02 | 02a | First sidequest from session 02 |
| 02 | 02b | Second sidequest from session 02 |
| 05 | 05a | First sidequest from session 05 |

This is called **hierarchical numbering** or **sub-numbering** - common in:
- JIRA subtasks (PROJ-123 → PROJ-123-1)
- RFC amendments (RFC 2a, 2b)
- Legal documents (Section 4.2.1)
- Semantic versioning patches

---

## Step 1: Identify Parent Session

**Priority order:**

### 1a: Arguments
If user provided: `/sidequest 02 "fix env loading"`

### 1b: Conversation Context
If you just ran `/park`, use that session number.

### 1c: Find Most Recent Parked
```bash
Grep("🅿️ PARKED", path="work/", glob="**/0*_*.md")
```

### 1d: Ask User
If unclear:
```
Which parked session should this sidequest branch from?
```

---

## Step 2: Determine Sidequest Letter

Check existing sidequests for this parent:

```bash
# If parent is 02, find existing 02a, 02b, etc.
Glob("02[a-z]_*_handoff.md", path="work/[sprint_folder]")
```

- No existing → use `a`
- 02a exists → use `b`
- 02a, 02b exist → use `c`

---

## Step 3: Research Known Patterns

Before scoping the sidequest, read known patterns that may explain the blocker or affect the fix:

- `.claude/rules/debugging-patterns.md` — common failure modes to check first
- `.claude/rules/codebase-quirks.md` — gotchas that may explain the issue or affect the fix

Use this knowledge to inform the Context and Tasks sections of the handoff.

---

## Step 4: Create Sidequest Handoff

Create `{num}{letter}_{short_name}_handoff.md`:

```markdown
# Session {num}{letter}: {Title} (Sidequest)

**Sprint:** {sprint_name}
**Status:** ⏳ NEXT
**Parent:** {parent_handoff} (PARKED)
**Type:** SIDEQUEST

---

## Goal

{One-line goal that unblocks parent}

---

## Context

{Why parent was parked, what this sidequest fixes}

---

## Tasks

- [ ] {Task 1}
- [ ] {Task 2}
- [ ] Verify fix
- [ ] (Optional) Prevent recurrence

---

## Files

| File | Change |
|------|--------|
| {file} | {change} |

---

## Success Criteria

- [ ] {Unblocking condition met}
- [ ] Parent session can resume

---

## Notes

- This unblocks Session {num} ({parent_title})
- After completion, return to parent or continue to next main session

---

*Sidequest from Session {num}. Quick fix, back to main quest.*
```

---

## Step 5: Update Overview Document

Add sidequest to Progress Tracker (indented or bold to show relationship):

```markdown
| 02 - P2P Session Tests | 🅿️ PARKED | 020326 | Code done, blocked |
| **02a - Fix Env Loading** | ⏳ NEXT | | SIDEQUEST - unblocks 02 |
| 03 - WebRTC Signaling | PENDING | | |
```

Update legend if needed:
```markdown
**Legend:** ✅ COMPLETE | ⏳ NEXT | PENDING | BLOCKED | 🅿️ PARKED | **##a** = SIDEQUEST
```

---

## Step 6: Commit

```bash
git add [sidequest_handoff] [overview_doc]

git commit -m "docs([sprint]): add sidequest [num][letter] - [description]

- Branch from parked Session [num]
- Goal: [what it unblocks]

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Step 7: Summary

```
## Sidequest Created

**Handoff:** [sidequest_handoff]
**Parent:** [parent_handoff] (PARKED)
**Goal:** [what it fixes]

### Files Created/Updated
- [sidequest_handoff] - New sidequest handoff
- [overview_doc] - Added to progress tracker

### To Start
```
/implement @[sidequest_handoff]
```

### After Completion
Either:
- Return to parent: `/implement @[parent_handoff]`
- Continue to next: `/implement @[next_main_session]`
```

---

## Sidequest vs Main Session

| Aspect | Main Session | Sidequest |
|--------|--------------|-----------|
| Numbering | 01, 02, 03 | 02a, 02b, 05a |
| Scope | Full feature/task | Quick fix/unblock |
| Duration | Hours to days | Minutes to hours |
| Trigger | Sprint plan | /park blocker |
| Completion | Mark COMPLETE | Return to parent |

---

## When to Use Sidequest

**Use sidequest when:**
- Parent is PARKED due to specific blocker
- Fix is small (< 1 hour)
- Fix is prerequisite for parent to continue

**Don't use sidequest when:**
- Issue is unrelated to parent
- Fix requires significant design work
- Better as its own main session

---

## Key Principles

1. **Quick and focused** - Sidequests should be completable in one session
2. **Unblocks parent** - Clear connection to why parent was parked
3. **Hierarchical naming** - Easy to see relationship (02 → 02a → 02b)
4. **Return path clear** - After sidequest, know where to go next
