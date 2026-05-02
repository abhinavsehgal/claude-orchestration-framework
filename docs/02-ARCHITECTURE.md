# 02 — Architecture

The physical layout of files in a project that uses this framework.

## The full picture

```
your-project/
│
├── CLAUDE.md                              ← project router (auto-loaded by Claude Code)
│
├── .claude/
│   ├── agents/                            ← orchestrator + N specialists
│   │   ├── <project>-orchestrator.md
│   │   ├── <specialist-1>.md
│   │   ├── <specialist-2>.md
│   │   └── ...
│   ├── rules/                             ← path-globbed invariants
│   │   ├── <domain-1>.md
│   │   ├── <domain-2>.md
│   │   └── ...
│   ├── skills/                            ← repeatable workflows
│   │   ├── <workflow-1>/SKILL.md
│   │   └── ...
│   ├── settings.json                      ← project-wide Claude Code settings
│   └── settings.local.json                ← personal permissions allowlist (gitignored if sensitive)
│
├── docs/
│   ├── ai-context/                        ← orientation maps (read by agents)
│   │   ├── INDEX.md                       ← task → docs router
│   │   ├── HANDOFF_SCHEMA.md              ← bidirectional handoff schema
│   │   ├── ORCHESTRATION_SPOONFEEDER.md   ← human usage guide
│   │   └── <area>-experience.md           ← per-domain orientation, 50-150 lines each
│   ├── ARCHITECTURE.md                    ← canonical refs (full detail)
│   ├── API.md
│   ├── <YOUR-OTHER-CANONICAL-DOCS>.md
│   └── _archive/                          ← frozen historical
│       ├── README.md                      ← archive convention
│       └── <YYYY-MM>/                     ← dated reports, post-mortems
│
└── <your application code>/
```

## What lives where, and why

### `CLAUDE.md` (root)

The router. Auto-loaded into every Claude Code session in this project. Should be **under 200 lines**. Contains:
- Identity (one sentence about the project)
- Golden rules (security, deployment, etc.)
- Mandatory workflow (numbered steps for every task)
- Routing tables (task → doc, task → agent)
- Cross-links to the spoonfeeder, INDEX, and canonical docs

Does NOT contain:
- Long technical detail (lives in canonical docs)
- Per-domain gotchas (lives in `.claude/rules/*.md`)
- Sprint history or audit content (lives in `docs/_archive/`)

### `.claude/agents/`

One markdown file per agent. Frontmatter declares `name`, `description`, `tools`, `maxTurns`, optionally `model`/`memory`/etc. Body declares the agent's contract: when to use, when not, required reading, I CAN, I CANNOT, definition of done, plus the handoff validation + return schema sections.

Always includes exactly one orchestrator. Specialists are added based on the project's natural domain boundaries (typically 5–12 specialists for a medium-complexity codebase).

See `03-AGENTS-GUIDE.md` for how to design agents.

### `.claude/rules/`

Path-globbed invariants. Each rule file's frontmatter has `applies_to:` listing path globs. When an agent is about to edit a file matching one of those globs, it must read the rule first. The rule contains "hard rules," "what not to do," and cross-links.

Example: `.claude/rules/database.md` might apply to `src/db/**/*.ts` and contain rules like "never run raw SQL in route handlers; use the repository layer."

Rules carry the **hard-won gotchas** of your codebase — things that broke production once and must not break again. They are short, scoped, and grow over time.

### `.claude/skills/`

Each skill is a directory with a `SKILL.md` file inside. The skill is a **repeatable workflow** — a template for tasks that recur. Examples:
- `investigate-bug/SKILL.md` — bug investigation steps
- `build-feature/SKILL.md` — feature build steps
- `qa-flow/SKILL.md` — pre-release QA pass
- `compliance-review/SKILL.md` — review for regulatory risks

Skills are invoked by name when the orchestrator (or a human) decides this task fits a known pattern.

### `docs/ai-context/`

Orientation maps. Each file is 50-150 lines and covers one area (e.g. `auth-and-sessions.md`, `payments.md`, `realtime-sync.md`). Format:
- Top: 2-sentence orientation
- Middle: key file paths + table list + the 3-5 gotchas worth knowing
- Bottom: cross-links to canonical refs

These are what agents read when deciding "what do I need to know about <area> before touching code in it." Cheaper than reading 1000-line canonical docs.

Special files in `docs/ai-context/`:
- `INDEX.md` — the master router (task → docs + rules + agent map)
- `HANDOFF_SCHEMA.md` — the bidirectional handoff schema (this framework's central artifact)
- `ORCHESTRATION_SPOONFEEDER.md` — the human-facing usage guide
- `MIGRATION_LEDGER.md` (optional) — tracks doc/structure migration progress
- `LEGACY_BACKUP.md` (optional) — frozen pre-refactor snapshot

### `docs/<UPPERCASE>.md`

Canonical references. Full detail. Examples:
- `ARCHITECTURE.md` — system architecture
- `API.md` — endpoint reference
- `TECH_STACK.md` — what's installed and why
- `BUSINESS_DOCUMENT.md` — non-technical product/business notes
- `CHANGELOG.md` — release log

These don't change file layout per-task; they're the encyclopedic source of truth.

### `docs/_archive/`

Frozen historical material. Sprint reports, post-mortems, dated audits, migration logs. Two rules:
1. Active docs (CLAUDE.md, `docs/ai-context/*`, agents, rules) **never link into `_archive/`**.
2. The archive is append-only — date subdirs, never reorganize within.

The `docs/_archive/README.md` documents these conventions for future contributors.

## Why this layout works

### Predictability for Claude

Every agent knows where to find:
- Its own contract: `.claude/agents/<self>.md`
- The handoff schema: `docs/ai-context/HANDOFF_SCHEMA.md`
- Routing: `docs/ai-context/INDEX.md`
- Rules for paths it might edit: `.claude/rules/*.md` matched via `applies_to`
- Canonical refs: `docs/<UPPERCASE>.md`

There is no ambiguity. New tasks follow the same path every time.

### Predictability for humans

A new engineer joining the project sees:
- `CLAUDE.md` at root — tells them how Claude Code is set up
- `docs/` — where documentation lives
- `.claude/` — Claude-specific config
- Application code in its standard tech-stack location

They can ignore `.claude/` if they don't use Claude Code. The framework adds value without imposing tax on humans who prefer their own tooling.

### Minimal coupling to tech stack

Nothing in `.claude/` or `docs/ai-context/` cares about your framework, language, or deploy target. The agents reference paths in your actual codebase, but the agent FILES themselves are pure markdown. You can reorganize your codebase entirely and only need to update the rule globs and the orientation maps' cross-links.

## What does NOT belong in this framework

- **Application code.** Stays where your tech stack puts it.
- **Build config.** Stays at root (your `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, etc.).
- **CI workflows.** Stays in `.github/`, `.gitlab-ci.yml`, etc.
- **Test files.** Stays in your tests directory.
- **Generated artifacts.** Gitignored.

The framework is **purely additive** to whatever else lives in your repo.

## Variations

### Monorepos

For monorepos, you can either:
- Have one `.claude/` at the root that knows about all packages, or
- Have one `.claude/` per package + a root `.claude/` that orchestrates across packages

The first is simpler. The second is more isolated. Pick based on whether tasks routinely cross package boundaries.

### Mobile + web split

If your project has separate mobile and web codebases in one repo, your orchestrator typically delegates to a `mobile-ui` specialist OR a `web-ui` specialist depending on the task. They can share a `backend-api` specialist for the API-side work.

### Multi-tenancy / multi-product

If you serve multiple products from one codebase, consider per-product orientation maps and per-product rules. The orchestrator can be one or per-product depending on whether tasks routinely cross products.

### Heavy infra / DevOps focus

Add specialists for `infrastructure`, `observability`, `release-automation`. Their rule files cover Terraform/CDK/Kubernetes/Helm conventions.

### ML / data heavy

Add specialists for `data-pipeline`, `model-training`, `inference-serving`. Their orientation maps cover dataset locations, experiment tracking, model versioning.

The architecture flexes; the principles don't.
