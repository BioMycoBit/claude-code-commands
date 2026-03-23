# User Preferences

Communication style, workflow preferences, and tool choices.
Discovered through interaction. Rarely changes.

**Cap:** 50 lines | **Write:** When user states preference

---

## Preferences

### Output Clarity
**Added:** YYYY-MM-DD
If output would make a non-coder squint, fix it proactively.
Technically correct but confusing = wrong. Be explicit, label edge cases.

### Always Generate Test Scripts After Implementation
**Added:** YYYY-MM-DD
After completing `/implement`, if the handoff has testable success criteria
(API endpoints, logging behavior, state changes), **immediately run `/debug-bt`**
— do NOT wait for the user to ask. The user expects proof of correctness.

### Commit Style
**Added:** YYYY-MM-DD
Concise commit messages. Use conventional commits (feat/fix/docs).
Co-Authored-By line required.

### Never Use Explore/Task for Known Paths
**Added:** YYYY-MM-DD
Skills live in `.claude/commands/`. Config lives in `.claude/rules/`.
If you know or can guess the path, use **Glob or Read directly** — never spawn an
Explore agent. Explore agents cost 20-50k tokens. A Glob costs ~200.
Use Explore only for genuinely open-ended codebase questions. Glob first, Explore never.

### Never Commit Before User Tests
**Added:** YYYY-MM-DD
After `/debug-bt` generates a test script, **STOP and wait for the user to run it**.
Do NOT commit. The test script is the gate — user runs it, reports results,
fixes any failures, THEN commits. This is a hard stop, not a suggestion.

### Abstraction-First, Never Bypass
**Added:** YYYY-MM-DD
Before implementing: check if an abstraction exists. Extend it — never bypass with
one-offs. Red flags: cloning internal handles, reaching through layers,
single-call-site patterns. Test: "Would a dev find this reading the API?" No = shortcut.

### Never Propose Placeholder/Fallback/Dummy Data (BANNED)
**Added:** YYYY-MM-DD
Missing state = show nothing or redirect. NEVER invent placeholder data to satisfy
the type checker. Empty array or loading state is correct. Before handling any edge
case, ask: "what is the correct behavior?" not "how do I make the types compile?"

---

*Add entries as you discover preferences. Keep entries atomic — one preference per block.*
