# Claude Code Commands

A collection of slash commands for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that turn AI-assisted development into a structured, repeatable workflow. Instead of ad-hoc prompting, you get a complete system: sprint planning, implementation with approval gates, 26 code audits, and persistent project memory.

Built over 50+ sprints on a real production codebase, then extracted and generalized for any project.

---

## Table of Contents

- [Quick Start](#quick-start)
- [What Are Slash Commands?](#what-are-slash-commands)
- [The Workflow](#the-workflow)
- [Skill Catalog](#skill-catalog)
- [Audit Suite](#audit-suite)
- [Configuration](#configuration)
- [Memory Protocol](#memory-protocol)
- [Examples](#examples)
- [License](#license)

---

## Quick Start

```bash
# 1. Clone this repo
git clone https://github.com/yourusername/claude-code-commands.git

# 2. Copy commands into your project
cp -r claude-code-commands/.claude/commands/ your-project/.claude/commands/

# 3. Copy semgrep rules (used by audit commands)
cp -r claude-code-commands/.semgrep/ your-project/.semgrep/

# 4. Start Claude Code in your project
cd your-project
claude

# 5. Type a slash command
/sprint my first feature
```

Claude Code auto-discovers commands in `.claude/commands/`. No configuration needed to start — just copy the files.

---

## What Are Slash Commands?

Slash commands are markdown files that Claude Code reads as instructions. Instead of explaining how to plan a sprint every time, you write the recipe once in a `.md` file, put it in `.claude/commands/`, and type `/sprint` to run it. Claude reads the recipe and follows each step.

**Example:** When you type `/sprint auth system`, Claude Code:
1. Reads `.claude/commands/sprint.md`
2. Researches your git history, existing docs, and codebase
3. Generates a sprint plan with numbered sessions
4. Creates the first handoff document for implementation

Each command in this repo encodes a specific workflow pattern — tested across 50+ sprints — so Claude follows a consistent, high-quality process every time.

---

## The Workflow

The core workflow is a loop of four commands:

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   /sprint ──▶ /implement ──▶ /handoff-end ──▶ repeat    │
│      │              │              │                     │
│   Creates        Executes       Closes session,         │
│   sprint plan    one session    creates next handoff     │
│      │              │              │                     │
│      │         GEAUX gate      Updates overview          │
│      │        (approval)       doc tracker               │
│      │              │              │                     │
│      └──────────────┴──────────────┘                     │
│                     │                                    │
│              /sprint-end                                 │
│           (closes the sprint)                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### How It Works

1. **`/sprint [topic]`** — Analyzes your codebase and generates a sprint plan with numbered sessions (handoff documents). Each session has a clear goal, blast radius, success criteria, and stop conditions.

2. **`/implement @work/path/to/handoff.md`** — Loads the handoff document, researches the blast radius files, presents a GO/NO-GO gate for your approval, then executes the implementation. Runs linters and a slop check (AI anti-pattern detector) before finishing.

3. **`/handoff-end`** — Closes the current session, updates the sprint overview's progress tracker, creates the next handoff document, and commits.

4. **`/sprint-end`** — Closes the entire sprint with a retrospective: what went well, what didn't, velocity metrics.

### Supporting Commands

| Command | When to Use |
|---------|-------------|
| `/sidequest` | Handle an interruption without losing context on main work |
| `/park` | Pause a session that's blocked — saves state for later |
| `/amplify 4 doc.md` | Refine any document through 4 progressive passes |
| `/debug-bt` | Generate test scripts to verify your implementation works |
| `/slop-check` | Detect AI-generated fluff: placeholders, unnecessary abstractions, scope creep |

### The Audit Loop

Audits run independently from the sprint cycle. When findings accumulate:

```
/audit-suite tier1  ──▶  /sprint-from-findings  ──▶  /sprint (remediation)
     │                         │
  Runs 5 audits            Extracts top findings
  in parallel              into a sprint plan
```

---

## Skill Catalog

### Workflow Commands (12)

| Command | Purpose |
|---------|---------|
| `/sprint` | Generate a sprint plan with architectural analysis, progress tracker, and first handoff |
| `/implement` | Execute a handoff document with approval gate, lint, slop check, and test generation |
| `/handoff-end` | Close a session, update the overview tracker, create next handoff, commit |
| `/sprint-end` | Close a sprint with retrospective insights and archive |
| `/amplify` | Iterative document refinement through 4 progressive passes |
| `/sidequest` | Branch a sub-task from a parked session with hierarchical numbering |
| `/park` | Temporarily set aside a blocked session with return context |
| `/debug-bt` | Generate test scripts for verifying implementations |
| `/slop-check` | Scan changed files for AI anti-patterns that linters miss |
| `/random-sprint` | Collect ad-hoc items interactively, then delegate to `/sprint` |
| `/pick-findings` | Circle back to audit findings that were initially skipped |
| `/sprint-from-findings` | Scan audit documents and build a remediation sprint from top findings |

### Audit Commands (27)

| Command | What It Checks |
|---------|---------------|
| `/audit-suite` | **Orchestrator** — runs all audits in a tier as parallel agents |
| `/audit-security` | SQL injection, XSS, command injection, secrets, TLS, debug mode |
| `/audit-secrets` | Git history scanning, .env hygiene, hardcoded credentials |
| `/audit-dependency` | Version pinning, major version gaps, outdated packages (Python/NPM/Rust) |
| `/audit-standards` | Linting trends, naming conventions, type annotations, exception handling |
| `/audit-test-quality` | Test inventory, flaky indicators, mock hygiene, coverage gaps |
| `/audit-python` | Cyclomatic complexity, maintainability, dead code, hotspot correlation |
| `/audit-typescript` | Circular dependencies, dead code, type duplication, component health |
| `/audit-rust` | Clippy pedantic, unwrap audit, dead code, unused PyO3 bindings |
| `/audit-codesweep` | Copy/paste duplication detection via jscpd |
| `/audit-resilience` | Error swallowing, missing timeouts, retry gaps, resource cleanup |
| `/audit-deadcode` | Unused exports, orphan files, uncalled functions |
| `/audit-regression` | Danger zone mapping, swallowed errors, test gaps for high-churn files |
| `/audit-performance` | Hot paths, N+1 queries, sequential awaits, unbounded growth |
| `/audit-docker` | Base image security, container privileges, resource limits, Trivy scanning |
| `/audit-observability` | Structured logging, correlation IDs, alert coverage, PII in logs |
| `/audit-pipeline` | Latency budgets, sample rates, buffer management, backpressure |
| `/audit-privacy` | GDPR/PII lifecycle, data retention, consent tracking, cookie usage |
| `/audit-modules` | Module boundaries, dependency enforcement (tach/knip) |
| `/audit-types` | Python type coverage (ty/pyright), TypeScript strict checking (tsc) |
| `/audit-iac` | Infrastructure config validation (Caddy, Prometheus, Grafana, docker-compose) |
| `/audit-database` | Schema consistency, query safety, connection patterns, migration safety |
| `/audit-sbom` | SBOM generation (CycloneDX/SPDX), vulnerability scanning via grype |
| `/audit-licenses` | GPL/AGPL/copyleft detection across Python, NPM, and Rust dependencies |
| `/audit-accessibility` | WCAG 2.1 AA: alt text, ARIA, keyboard handlers, focus styles, axe-core |
| `/audit-browser-compat` | CSS prefixes, modern JS APIs, polyfill checks (Chrome 90+, Firefox 88+, Safari 14+) |
| `/audit-api-surface` | Endpoint-to-frontend tracing, orphan endpoints, WebSocket message flow mapping |

### On-Demand Audits (4, not in tiers)

| Command | Purpose |
|---------|---------|
| `/audit-dashboard` | Unified health dashboard — reads all recent audits, computes composite scores |
| `/audit-project-overview` | Project pulse check — version status, recent work, sprint priorities |
| `/audit-system-walkthrough` | End-to-end data flow tracing through all system layers |
| `/audit-techdebt` | TODO/FIXME markers, type-safety gaps, large files, backward-compat code |

---

## Audit Suite

The 26 tiered audits are organized by priority:

| Tier | Frequency | Audits | Focus |
|------|-----------|--------|-------|
| **Tier 1** | Every sprint | security, dependency, secrets, standards, test-quality | Critical safety |
| **Tier 2** | Biweekly | python, typescript, rust, codesweep, resilience, deadcode, regression, performance | Code quality |
| **Tier 3** | Monthly | docker, observability, pipeline, privacy, modules, types, iac, database | Infrastructure |
| **Tier 4** | Quarterly | sbom, licenses, accessibility, browser-compat, api-surface | Compliance |

### Running Audits

```bash
# Run a single audit
/audit-security

# Run a full tier (parallel — all audits launch simultaneously)
/audit-suite tier1

# Run with options
/audit-suite tier1 --full          # Deep analysis with Plan agent (slower, more context)
/audit-suite tier2 --sarif         # Machine-readable SARIF output alongside markdown
/audit-python --since last         # Show delta vs last audit (new/resolved/carried)
/audit-standards --fix             # Auto-fix what's safe (ruff --fix, etc.)

# Build a remediation sprint from findings
/sprint-from-findings
```

### What Audits Produce

Each audit generates a markdown report in `work/` with:
- **Executive summary** — 2-3 sentences on what was found
- **Metrics dashboard** — key metrics with trend arrows vs prior audit
- **Findings table** — severity (Critical/High/Medium/Low), file, line, effort estimate
- **Delta tracking** — new, resolved, carried, and changed findings vs prior run
- **Tech debt integration** — Critical/High findings auto-append to `.claude/rules/tech-debt-patterns.md`

Audits use [Semgrep](https://semgrep.dev/) (4 custom rule files in `.semgrep/`) for AST-aware pattern matching, plus built-in grep for text patterns.

---

## Configuration

These commands work out of the box for basic use. For project-specific behavior (custom lint commands, target directories, tech stack), see **[CONFIGURATION.md](CONFIGURATION.md)**.

Key configuration points:
- **CLAUDE.md** — Your project instructions file that commands read for context
- **`.claude/rules/`** — Persistent knowledge base files (debugging patterns, tech debt, quirks)
- **Lint commands** — Configure in CLAUDE.md; `/implement` reads them for the lint step
- **Semgrep rules** — Customize `.semgrep/*.yml` for your codebase patterns

The `examples/` directory includes starter templates for all of these.

---

## Memory Protocol

The commands use a file-based knowledge system in `.claude/rules/` that persists across conversations. Claude reads these files during `/implement` research and updates them during audits and sprint closures.

### Knowledge Files

| File | Purpose | Cap |
|------|---------|-----|
| `debugging-patterns.md` | Known failure modes: symptom → fix | 200 lines |
| `codebase-quirks.md` | Import patterns, gotchas, workarounds | 150 lines |
| `tech-debt-patterns.md` | Audit findings awaiting remediation | 100 lines |
| `user-preferences.md` | Communication style, workflow preferences | 50 lines |
| `cross-platform-risks.md` | Platform-specific anti-patterns and alternatives | 100 lines |

### How It Works

1. **Audits** write findings to `tech-debt-patterns.md` (Critical/High severity)
2. **`/implement`** reads all rules files before writing code — avoiding known failure modes
3. **`/handoff-end`** and **`/park`** append new debugging patterns discovered during implementation
4. **Resolved** items get struck through and eventually pruned

Each entry follows a consistent format:
```markdown
### Short Description
**Added:** YYYY-MM-DD | **Source:** /audit-python
One-line description of the issue.
**Location:** path/to/file.py
**Resolve:** Specific action needed to fix.
```

Starter templates with examples are in `examples/rules/`.

---

## Examples

The `examples/` directory contains starter templates:

| File | Purpose |
|------|---------|
| `CLAUDE.md.example` | Minimal project instructions file — copy to your project root as `CLAUDE.md` |
| `rules/debugging-patterns.md.example` | Debugging pattern template with example entries |
| `rules/codebase-quirks.md.example` | Codebase quirks template with example entries |
| `rules/tech-debt-patterns.md.example` | Tech debt tracking template with example entries |
| `rules/user-preferences.md.example` | User preferences template with example entries |
| `rules/cross-platform-risks.md.example` | Cross-platform risk registry with anti-pattern table |

### Setup for a New Project

```bash
# 1. Copy commands and semgrep rules
cp -r .claude/commands/ /path/to/your-project/.claude/commands/
cp -r .semgrep/ /path/to/your-project/.semgrep/

# 2. Create your CLAUDE.md (use the example as a starting point)
cp examples/CLAUDE.md.example /path/to/your-project/CLAUDE.md
# Edit CLAUDE.md with your project name, stack, lint commands, etc.

# 3. Create rules directory with starter templates
mkdir -p /path/to/your-project/.claude/rules/
for f in examples/rules/*.example; do
  cp "$f" "/path/to/your-project/.claude/rules/$(basename "$f" .example)"
done
# Clear the example entries and start fresh

# 4. Create work directory (where sprints and audits live)
mkdir -p /path/to/your-project/work/
```

---

## License

MIT — see [LICENSE](LICENSE).
