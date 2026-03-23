# Configuration Guide

How to adapt these commands for your project. Most commands work out of the box — this guide covers the places where project-specific paths or tools matter.

---

## 1. CLAUDE.md — Your Project Instructions

The commands read `CLAUDE.md` in your project root for context. This file tells Claude about your project structure, tech stack, lint commands, and conventions.

**Start from the template:**
```bash
cp examples/CLAUDE.md.example.md CLAUDE.md
```

**Key sections to customize:**

| Section | What to Add |
|---------|-------------|
| Project / Stack | Your project name and tech stack |
| Essential Commands | Your lint, test, and build commands |
| File Structure | Where your source code lives |
| Testing & Commit Protocol | Your team's commit workflow |

The `/implement` skill reads the Essential Commands section to know which linters to run. If this section is empty, it falls back to auto-detection (checking for `package.json`, `pyproject.toml`, `Makefile`, etc.).

---

## 2. Target Lines in Audit Commands

Each audit command has a `**Target:**` line near the top that tells Claude which directories to scan. These use placeholder paths like `backend/`, `src/`, `your-rust-service/`.

**Example — `/audit-python` (line 5):**
```markdown
**Target:** Python source directories (e.g., `src/backend/`, `app/`, `backend/`)
```

**To customize:** Edit the Target line in each audit command you use. Claude reads this line to scope its file searches.

**Common adaptations:**

| Project Type | Python Target | TypeScript Target | Rust Target |
|-------------|---------------|-------------------|-------------|
| Django | `myapp/`, `myapp/api/` | — | — |
| FastAPI + React | `backend/`, `api/` | `src/`, `frontend/src/` | — |
| Full stack + Rust | `backend/` | `src/` | `crates/*/src/` |
| Monorepo | `packages/api/src/` | `packages/web/src/` | `packages/engine/src/` |

You don't need to edit every audit — only the ones relevant to your stack. A Python-only project can ignore `/audit-typescript` and `/audit-rust`.

---

## 3. Lint Commands

The `/implement` command runs linters after making changes. It reads your CLAUDE.md for the specific commands, but falls back to common patterns:

```markdown
# In your CLAUDE.md, under Essential Commands:

**Linting:**
```bash
ruff check .                    # Python
npm run lint                    # TypeScript/React
cargo clippy -- -W warnings     # Rust
```
```

The `/audit-standards` command also runs linters directly. Its Phase 1 and Phase 2 sections reference specific commands — update these if your linters differ:

```markdown
# In audit-standards.md, Phase 1:
ruff check . --statistics 2>&1    # Change to: flake8 --statistics 2>&1
```

---

## 4. Work Directory

Sprint plans, handoff documents, and audit reports are stored in `work/` at your project root:

```
work/
├── v1.0_auth/                    # Sprint folder
│   ├── 00_auth_overview_v4.md    # Sprint overview
│   ├── 01_login_handoff.md       # Session 1 handoff
│   └── 02_tokens_handoff.md      # Session 2 handoff
├── 032026_Python_Audit.md        # Audit report
└── 032026_Security_Audit.md      # Audit report
```

This directory is created automatically by `/sprint`. Add `work/` to `.gitignore` if you don't want sprint artifacts in version control, or keep them for project history.

---

## 5. Semgrep Rules

The `.semgrep/` directory contains 4 custom rule files used by audit commands:

| File | Used By | Patterns |
|------|---------|----------|
| `security.yml` | `/audit-security` | SQL injection, XSS, command injection, secrets exposure |
| `resilience.yml` | `/audit-resilience` | Error swallowing, broad except, empty catch, missing timeouts |
| `standards.yml` | `/audit-standards` | Naming conventions, debug output, type annotation issues |
| `privacy.yml` | `/audit-privacy` | PII field detection, PII logging, localStorage PII |

**To customize:** Add project-specific patterns to the relevant `.yml` file. Each file uses standard [Semgrep rule syntax](https://semgrep.dev/docs/writing-rules/rule-syntax/).

**Example — adding a project-specific security rule:**
```yaml
# In .semgrep/security.yml, add to the rules list:
- id: no-raw-sql-in-views
  pattern: |
    cursor.execute($SQL, ...)
  message: Use the ORM query builder instead of raw SQL in view functions
  severity: WARNING
  languages: [python]
  paths:
    include:
      - "*/views.py"
```

If you don't use Semgrep, the audit commands fall back to grep-based pattern matching. Install Semgrep for better accuracy: `pip install semgrep` or `brew install semgrep`.

---

## 6. Rules Files (Memory Protocol)

The `.claude/rules/` files are your project's persistent knowledge base. Start from the templates:

```bash
mkdir -p .claude/rules/
for f in examples/rules/*.example; do
  cp "$f" ".claude/rules/$(basename "$f" .example)"
done
```

**Clear the example entries** and let them fill organically as you work:

- **`debugging-patterns.md`** — Gets entries when you debug tricky issues
- **`codebase-quirks.md`** — Gets entries when you discover gotchas during implementation
- **`tech-debt-patterns.md`** — Gets entries from audits (Critical/High findings)
- **`user-preferences.md`** — Gets entries when you tell Claude how you like to work
- **`cross-platform-risks.md`** — Pre-populated with common patterns; add project-specific ones

Each file has a line cap (listed in the header). When a file hits its cap, evaluate the oldest entries for removal.

---

## 7. Adapting for Different Tech Stacks

### Python-Only Project (Django, Flask, FastAPI)

1. Keep: All workflow commands, `/audit-python`, `/audit-security`, `/audit-secrets`, `/audit-standards`, `/audit-test-quality`, `/audit-dependency`, `/audit-deadcode`, `/audit-performance`, `/audit-resilience`, `/audit-types`
2. Skip: `/audit-typescript`, `/audit-rust`, `/audit-browser-compat`, `/audit-accessibility`
3. Update Target lines to point at your Python source directories
4. Set lint command in CLAUDE.md (`ruff check .`, `flake8`, `mypy`, etc.)

### JavaScript/TypeScript Project (React, Next.js, Node)

1. Keep: All workflow commands, `/audit-typescript`, `/audit-security`, `/audit-secrets`, `/audit-standards`, `/audit-test-quality`, `/audit-dependency`, `/audit-deadcode`, `/audit-performance`, `/audit-resilience`, `/audit-types`, `/audit-accessibility`, `/audit-browser-compat`
2. Skip: `/audit-python`, `/audit-rust`
3. Update Target lines to point at your source directories
4. Set lint command in CLAUDE.md (`npm run lint`, `eslint .`, `biome check`, etc.)

### Rust Project

1. Keep: All workflow commands, `/audit-rust`, `/audit-security`, `/audit-secrets`, `/audit-dependency`, `/audit-performance`, `/audit-resilience`
2. Skip: `/audit-python`, `/audit-typescript`, `/audit-accessibility`, `/audit-browser-compat`
3. Update Target lines with your crate paths
4. Set lint command in CLAUDE.md (`cargo clippy`, `cargo test`, etc.)

### Removing Unused Commands

You can safely delete any `.claude/commands/audit-*.md` file you don't need. The workflow commands (`sprint.md`, `implement.md`, etc.) are stack-agnostic and should be kept.

If you remove audits, update the tier membership in `audit-suite.md` so `/audit-suite tier1` doesn't try to launch a deleted audit.

---

## Quick Reference

| What to Configure | Where |
|-------------------|-------|
| Project context | `CLAUDE.md` (root) |
| Lint commands | `CLAUDE.md` → Essential Commands section |
| Audit scan directories | `**Target:**` line in each `audit-*.md` |
| Semgrep patterns | `.semgrep/*.yml` |
| Knowledge base templates | `.claude/rules/*.md` (from `examples/rules/`) |
| Audit tier membership | `.claude/commands/audit-suite.md` |
