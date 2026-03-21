# Random Sprint Skill

Collect ad-hoc items interactively, then delegate to `/sprint` with the full item list as focus context.

## Usage

```
/random-sprint v1.u
/random-sprint v1.7_cleanup
```

**Argument:** A version/folder shorthand (e.g., `v1.u`). The suffix `_random` is appended automatically → `v1.u_random`.

---

## Step 1: Parse Version Prefix

Extract the version prefix from the arguments.

- **Input:** `v1.u` → **Sprint identifier:** `v1.u_random`
- **Input:** `v1.7_cleanup` → **Sprint identifier:** `v1.7_cleanup_random`
- **No input:** Ask the user for a version prefix.

---

## Step 2: Enter Collection Mode

Present the collection prompt:

```
## Random Sprint Collector

**Sprint:** {identifier}

I'll collect items for this sprint one at a time. Give me each item and I'll add it to the list. Say **done** when finished.

**Item 1:**
```

---

## Step 3: Collect Items

For each item the user provides:

1. Add it to the internal list
2. Echo back the running list so far
3. Prompt for the next item

**Format after each entry:**

```
**Collected so far:**
  [1] {first item}
  [2] {second item}

**Item {N+1}:** (or say **done**)
```

**Exit condition:** User says "done", "Done", "DONE", or any case variation.

**Minimum:** At least 1 item required. If user says "done" with 0 items, prompt: "Need at least one item. What's the first one?"

---

## Step 4: Confirm & Delegate

Once the user says "done", present the final list for confirmation:

```
## Sprint Items Collected

**Sprint:** {identifier}
**Items:** {count}

  [1] {item 1}
  [2] {item 2}
  [3] {item 3}
  ...

Launching `/sprint {identifier}` now.
```

Then invoke `/sprint {identifier}` with the following focus topic:

```
The user has selected the following items for this sprint. Each item represents a
candidate task or session. Use these as the primary input for session decomposition
and sprint planning.

Sprint items:
  [1] {item 1}
  [2] {item 2}
  [3] {item 3}
  ...

Organize these into logical sessions. Items may be grouped, split, or reordered
based on dependencies and effort. Every item must appear in the final plan —
either as its own session, grouped into a session with related items, or explicitly
deferred with rationale.
```

---

## Error Handling

| Error | Action |
|-------|--------|
| No version prefix | Ask user for one |
| Zero items at "done" | Re-prompt for at least 1 |
| User says "cancel" or "abort" | Exit cleanly, no sprint created |

---

## Example Execution

```
User: /random-sprint v1.u

Claude:
## Random Sprint Collector

**Sprint:** v1.u_random

I'll collect items for this sprint one at a time. Give me each item
and I'll add it to the list. Say **done** when finished.

**Item 1:**

User: Fix the login redirect bug

Claude:
**Collected so far:**
  [1] Fix the login redirect bug

**Item 2:** (or say **done**)

User: Update dependency versions

Claude:
**Collected so far:**
  [1] Fix the login redirect bug
  [2] Update dependency versions

**Item 3:** (or say **done**)

User: Refactor message_handlers.py into subdirectory

Claude:
**Collected so far:**
  [1] Fix the login redirect bug
  [2] Update dependency versions
  [3] Refactor message_handlers.py into subdirectory

**Item 4:** (or say **done**)

User: done

Claude:
## Sprint Items Collected

**Sprint:** v1.u_random
**Items:** 3

  [1] Fix the login redirect bug
  [2] Update dependency versions
  [3] Refactor message_handlers.py into subdirectory

Launching `/sprint v1.u_random` now.

[Invokes /sprint v1.u_random with formatted item list as focus]
```
