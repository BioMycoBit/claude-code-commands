# CLAUDE System Instructions

**Project:** [Your Project Name]
**Stack:** [Your tech stack summary]

---

## Operating Principles

1. **Data-driven debugging**: Logs → data → fix (no theories without data)
2. **One handoff at a time**: Use `/implement` → `/handoff-end` cycle
3. **Skills handle workflow**: `/sprint`, `/implement`, `/amplify`, `/handoff-end`
4. **Obvious over clever**: If output would confuse a non-coder, fix it
5. **Fail loud**: Crash with clear errors > silent failures

---

## Memory Protocol

Claude has persistent memory across conversations via `.claude/rules/` files:

| What | Where | Cap |
|------|-------|-----|
| Debugging patterns | `.claude/rules/debugging-patterns.md` | 200 lines |
| Codebase quirks | `.claude/rules/codebase-quirks.md` | 150 lines |
| Tech debt | `.claude/rules/tech-debt-patterns.md` | 100 lines |
| User preferences | `.claude/rules/user-preferences.md` | 50 lines |
| Cross-platform risks | `.claude/rules/cross-platform-risks.md` | 100 lines |

### Write Rules

1. **Append-only** — add new entries at bottom, never rewrite the file
2. **Date every entry** — `**Added:** YYYY-MM-DD`
3. **One insight per entry** — keep entries atomic
4. **Check before writing** — grep first, don't duplicate
5. **Stay under cap** — if at cap, evaluate oldest entries for removal

---

## Essential Commands

**Linting:**
```bash
# Add your project's lint commands here
# Example: ruff check . (Python), npm run lint (TypeScript)
```

**Testing:**
```bash
# Add your project's test commands here
```

---

## Workflow: Skill-Based Development

### Starting a Sprint
```
/sprint [focus topic]
```

### Beginning a Session
```
/implement @work/path/to/handoff.md
```

### Ending a Session
```
/handoff-end [completed] [next]
```

---

## Testing & Commit Protocol

1. Make changes → run linters
2. Wait for user to test
3. Fix issues if any
4. Get explicit approval
5. Commit

---

*Customize this file for your project. The slash commands read this file for context.*
