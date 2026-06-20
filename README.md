# my-claude-code-framework

A batteries-included setup for [Claude Code](https://claude.com/claude-code): a curated library of
**subagents, slash commands, skills, hooks, MCP server configs, behavioral rules, CLAUDE.md templates,
and a dual-memory bank system** — ready to drop into any project or install globally.

It's a template repo: there's no application code. You copy the pieces you want into your own
`~/.claude/` (global) or a project's `.claude/` directory and Claude Code picks them up automatically.

---

## What's inside

| Component | Count | Location | Loaded |
|-----------|-------|----------|--------|
| **Subagents** | 110 | `.claude/agents/` | On demand (by task or name) |
| **Slash commands** | 51 | `.claude/commands/` | When you type `/category:name` |
| **Skills** | 40 | `.claude/skills/` | On demand (auto-matched to request) |
| **Plugins** | 108 | `.claude/plugins/` | When installed |
| **Hooks** | 20 scripts | `.claude/hooks/` | Automatically on tool/session events |
| **MCP server configs** | 14 bundles | `.claude/mcp-configs/` | When wired into `.mcp.json` |
| **Context profiles** | 5 | `.claude/contexts/` | When applied |
| **Behavioral rules** | `core-rules.md` | `.claude/rules/` | Always |
| **CLAUDE.md templates** | 3 + migration guide | `.claude/` | Reference |

---

## Quick start

### Option A — Install globally (recommended for trying it out)

The installer copies commands, hooks, rules, templates, and a reference MCP config into `~/.claude/`,
so they're available across all your projects.

```bash
git clone https://github.com/vbrazo/my-claude-code-framework.git
cd my-claude-code-framework
bash .claude/setup/install.sh
```

The script is interactive — it asks before installing each group (commands / hooks / rules /
templates / MCP config). After install:

1. Review the files it placed in `~/.claude/`.
2. Update hook script paths in `~/.claude/hooks.json` to absolute paths (see [Hooks](#hooks)).
3. Restart Claude Code, then try `/git:commit` or `/testing:tdd`.

### Option B — Per-project (vendored)

Copy just the pieces a project needs into its own `.claude/` directory so they're shared with the
team via git:

```bash
mkdir -p myproject/.claude
cp -r my-claude-code-framework/.claude/agents      myproject/.claude/
cp -r my-claude-code-framework/.claude/commands    myproject/.claude/
cp -r my-claude-code-framework/.claude/skills       myproject/.claude/
cp -r my-claude-code-framework/.claude/rules        myproject/.claude/
cp    my-claude-code-framework/.claude/CLAUDE-template-1.md  myproject/CLAUDE.md
```

Project-level `.claude/` takes precedence over the global `~/.claude/` when both define the same name.

---

## Using the components

### Subagents

110 specialized agents organized into 8 categories under `.claude/agents/`:

| Category | Examples |
|----------|----------|
| `core-development` | backend-developer, frontend-architect, fullstack-engineer, api-designer |
| `language-experts` | python-engineer, rust-systems, golang-developer, typescript-specialist |
| `data-ai` | ml-engineer, data-scientist, llm-architect, nlp-engineer |
| `infrastructure` | devops-engineer, kubernetes-specialist, cloud-architect, terraform-engineer |
| `quality-assurance` | code-reviewer, test-architect, security-auditor, performance-engineer |
| `developer-experience` | build-engineer, dx-optimizer, refactoring-specialist, documentation-engineer |
| `business-product` | product-manager, business-analyst, technical-writer, ux-researcher |
| `orchestration` | multi-agent-coordinator, workflow-director, context-manager |

Each agent is a Markdown file with frontmatter (`name`, `description`, `tools`, `model`). Invoke one by
naming it ("use the **rust-systems** agent to…") or let Claude pick based on the task. Edit the
frontmatter to change which tools or model an agent uses.

### Slash commands

51 commands grouped by namespace. Invoke as `/namespace:command`:

| Namespace | Commands |
|-----------|----------|
| `/git:` | commit, pr-create, pr-review, changelog, release, fix-issue, worktree |
| `/testing:` | tdd, e2e, integration-test, snapshot-test, test-coverage, test-fix |
| `/architecture:` | plan, adr, design-review, diagram, migrate, refactor, explain-architecture-pattern |
| `/security:` | audit, security-audit, hardening, dependency-audit, secrets-scan, csp, check-best-practices, secure-prompts |
| `/refactoring:` | cleanup, dead-code, extract, rename, simplify |
| `/documentation:` | doc-gen, api-docs, onboard, create-readme-section, create-release-notes, memory-bank, update-codemap |
| `/devops:` | ci-pipeline, deploy, dockerfile, k8s-manifest, monitor |
| `/workflow:` | orchestrate, checkpoint, wrap-up |

Example: `/git:commit` analyzes staged changes and writes a conventional commit; `/testing:tdd` starts
a Red-Green-Refactor cycle for a feature you describe.

### Skills

40 skills under `.claude/skills/` covering language idioms, framework patterns, and workflows
(e.g. `python-best-practices`, `react-patterns`, `kubernetes-operations`, `tdd-mastery`,
`security-hardening`, `deep-dive`). Claude loads a skill automatically when your request matches its
description — no explicit invocation needed. Browse the directory to see the full catalog.

### Hooks

20 scripts in `.claude/hooks/scripts/`, wired up in `.claude/hooks/hooks.json`. They run automatically
on tool and session lifecycle events to enforce guardrails and automate chores, including:

- **`smart-approve.py`** — decomposes compound bash commands and checks each against allow/deny lists
- **`commit-guard.js`** — validates conventional-commit message format before `git commit`
- **`pre-push-check.js`** — confirms branch/remote before `git push`
- **`secret-scanner.js`** — blocks edits that would commit secrets
- **`block-dev-server.js`** — prevents orphaned dev-server processes
- **`auto-test.js` / `lint-fix.js` / `type-check.js`** — run after edits
- **`session-start.js` / `session-end.js` / `context-loader.js`** — manage session context and memory

> After installing, edit `~/.claude/hooks.json` so each `command` points to an **absolute** path under
> `~/.claude/hooks/scripts/`. The bundled paths are relative for repo use.

### MCP server configs

14 ready-made bundles in `.claude/mcp-configs/` (e.g. `frontend.json`, `fullstack.json`, `devops.json`,
`data-science.json`, `kubernetes.json`, `security.json`, `observability.json`, plus `recommended.json`).
Each lists MCP servers for a stack with placeholder credentials. To use one, copy the servers you need
into your project's `.mcp.json` (or `~/.claude/mcp.json`) and replace the placeholders with real values.

### Context profiles

5 mode files in `.claude/contexts/` — `dev.md`, `debug.md`, `deploy.md`, `research.md`, `review.md` —
each a set of behavioral instructions tuned for a phase of work (e.g. `dev` favors speed and iteration;
`review` favors scrutiny). Paste one into a session, or reference it from your CLAUDE.md, to shift
Claude's posture.

---

## Memory system

This framework uses a **dual-memory architecture** so project knowledge survives context resets:

- **Primary (git-shared):** `CLAUDE-*.md` memory-bank files in the repo root, read on demand —
  `CLAUDE-activeContext.md`, `CLAUDE-patterns.md`, `CLAUDE-decisions.md`,
  `CLAUDE-troubleshooting.md`, `CLAUDE-config-variables.md`.
- **Shadow (machine-local):** native auto-memory under `memory/` that mirrors key content
  (`MEMORY.md` index, `patterns.md`, `architecture.md`, `build.md`).

After significant work, update the `CLAUDE-*.md` files and sync key points into the auto-memory topic
files. If `CLAUDE.md` is ever wiped, the auto memory retains project knowledge — recover it via
`/memory`. See `.claude/CLAUDE.md` for the full workflow and the `/documentation:memory-bank` command
to automate updates.

### CLAUDE.md templates

Three starter templates with different philosophies, plus a migration guide:

| Template | Lines | Philosophy |
|----------|-------|------------|
| `CLAUDE-template-1.md` | ~101 | Compact, self-contained + memory resilience |
| `CLAUDE-template-2.md` | ~153 | Memory-bank headline + dual memory |
| `CLAUDE-template-3.md` | ~105 | Progressive-disclosure native |

To migrate an existing `CLAUDE.md`, see `.claude/CLAUDE-migrate-to-new-template.md`.

---

## Behavioral rules

`.claude/rules/core-rules.md` is always loaded and defines how Claude works in this setup:
investigation before claims, scope discipline, verification before declaring done, asking before
destructive actions, and memory resilience. Add path-scoped rules (`.claude/rules/*.md`) to apply
guidance only to matching files.

---

## Repository layout

```
.claude/
├── CLAUDE.md                      # Project context (always loaded)
├── CLAUDE-template-{1,2,3}.md     # Starter CLAUDE.md templates
├── CLAUDE-migrate-to-new-template.md
├── AGENTS.md                      # Fast-tools cheat sheet (rg/fd/jq)
├── agents/                        # 110 subagents (8 categories)
├── commands/                      # 51 slash commands (by namespace)
├── skills/                        # 40 skills
├── plugins/                       # 108 plugins
├── hooks/                         # hooks.json + 20 scripts
├── mcp-configs/                   # 14 MCP server bundles
├── contexts/                      # 5 mode profiles
├── rules/                         # core-rules.md (always loaded)
└── setup/install.sh               # Interactive global installer
```

---

## License

See repository for license details.
