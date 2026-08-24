# Claude Orchestration Framework

> **Version 1.2.1** ([changelog](CHANGELOG.md)) · MIT license · `templates/` + `docs/` are tech-stack and domain agnostic
>
> **v1.2.0 (2026-08-22) — three months of production use, folded back in.** Two new chapters: **11 — Project truth, learnings and the evidence ladder** (the docs a fresh agent reads first, the six-gate playbook, "deferred work must be written", "every production push freshens the docs") and **12 — Multi-repo workspaces** (web + mobile + microservices in separate repos: three layers, two delegation mechanisms, `templates/workspace/`). Nine new pitfalls. A fifth hook (doc-freshness gate). **And two v1.0 platform claims corrected:** subagents *can* nest now, and `.claude/rules/` with `paths:` frontmatter is native — the framework's `applies_to:` field is renamed `paths:` (the hook reads both). Every platform claim in v1.2.0 carries a verified-on date.
>
> ⚠ Adopters of v1.1.0 should be on ≥ v1.1.2 (Stop-hook stderr fix). v1.2.0 is additive on top.
>
> **Purpose.** A reusable multi-agent orchestration setup for [Claude Code](https://docs.claude.com/en/docs/claude-code) that prevents cascading hallucinations, enforces evidence-based handoffs between agents, and makes Claude usable on production codebases by teams. Tech-stack agnostic — drops into any project (web, mobile, backend, ML, infra) in 2-4 hours.

> **Read the onboarding guide online:** https://abhinavsehgal.github.io/claude-orchestration-framework/ — all three editions, one page.

---

## Who this is for

- Engineering teams using Claude Code on real production codebases
- Projects with multiple roles (admin / customer / partner / etc.), multiple data systems, or compliance/security/payments concerns
- Anyone who has experienced Claude flooding context, drifting on rules, or silently propagating hallucinations across hops

If your project is a solo prototype or under ~5,000 lines, you probably don't need this yet — default Claude Code is enough. Adopt this when complexity passes the threshold.

## The 30-second pitch

Default Claude Code is excellent for small tasks. For complex projects, you need:

- **One orchestrator that delegates to specialists** (instead of one agent doing everything and flooding context)
- **Bidirectional structured handoffs with evidence-bound claims** (instead of free-form prompts that propagate hallucinations)
- **Per-agent runtime constraints** — REVIEW-ONLY agents physically can't write; tool allowlists scope MCP access; `maxTurns` caps bound execution
- **A `failure_condition` field** that forces specialists to STOP if the orchestrator's premise turns out wrong (instead of pushing through on a false hypothesis)
- **Three-tier docs** — orientation maps for Claude, canonical refs for humans, frozen archive for history

This framework gives you all of that without runtime hooks or external dependencies — just Claude Code's built-in subagent features (`tools`, `maxTurns`, `Agent(...)`, `mcpServers`) plus documentation conventions.

## What's in the box

```
claude-orchestration-framework/
├── README.md                           ← this file
├── Claude-Orchestration-Framework.pdf  ← consolidated 57-page printable (v1.1.2 render — Chapters 11–12 not yet included)
├── LICENSE
│
├── docs/                                ← the framework explained (quickstart + 13 chapters)
│   ├── 00-QUICKSTART.md                 ← START HERE: step-by-step onboarding for any project, incl. many repos
│   ├── 00-QUICKSTART.html                ← the same guide as one offline page with tabs for all three editions (open in a browser)
│   │                                    live: https://abhinavsehgal.github.io/claude-orchestration-framework/
│   ├── 01-PRINCIPLES.md                 ← seven core principles
│   ├── 02-ARCHITECTURE.md               ← .claude/ + docs/ layout (with monorepo / mobile+web / multi-product variants)
│   ├── 03-AGENTS-GUIDE.md               ← how to design orchestrator + specialists (per-stack tables)
│   ├── 04-HANDOFF-SCHEMA.md             ← bidirectional schema + worked examples (web / mobile / REVIEW-ONLY)
│   ├── 05-RULES-AND-SKILLS.md           ← path-globbed rules + repeatable workflows
│   ├── 06-INVOCATION-MODES.md           ← claude vs --agent vs specialist vs headless -p vs dynamic workflows vs routines
│   ├── 07-FOLDER-STRUCTURE.md           ← three-tier doc organization
│   ├── 08-COMMON-PITFALLS.md            ← 28 hard-won lessons
│   ├── 09-RUNBOOK.md                    ← step-by-step bootstrap (~2-4 hours)
│   ├── 10-HOOK-HARDENING.md             ← (v1.1) optional hook-based enforcement — five patterns as of v1.2
│   ├── 11-PROJECT-TRUTH-AND-LEARNINGS.md← (v1.2) PROJECT.md / LEARNINGS.md / backlogs, the evidence ladder, the six-gate playbook
│   ├── 12-MULTI-REPO-WORKSPACES.md      ← (v1.2) web + mobile + microservices across repos: layers, delegation, contracts
│   └── 13-STANDING-ROUTINES.md          ← (v1.3) scheduled autonomy: routine fleets, output contracts, budgets, review gates
│
├── prompts/                             ← ready-to-paste prompts for Claude Code
│   ├── INVENTORY-PROMPT.md              ← scan + propose specialists (run first)
│   ├── BOOTSTRAP-PROMPT.md              ← generate all framework files (run second)
│   └── REFINEMENT-PROMPT.md             ← post-bootstrap hardening (run after 2-3 real tasks)
│
└── templates/                           ← drop-in templates with placeholders
    ├── CLAUDE.md.template               ← (v1.2) the root router, golden rules 1–13
    ├── PROJECT.md.template              ← (v1.2) current truth — what is live where
    ├── LEARNINGS.md.template            ← (v1.2) decisions / failures / bug patterns / corrections
    ├── BACKLOG.md.template              ← (v1.2) deferred work, written not spoken
    ├── GLOSSARY.md.template             ← (v1.2) one name per concept
    ├── engineering-playbook-skill.md.template ← (v1.2) six gates + evidence ladder
    ├── orchestrator-agent.md.template
    ├── specialist-agent.md.template
    ├── review-only-agent.md.template
    ├── HANDOFF_SCHEMA.md.template
    ├── INDEX.md.template
    ├── SPOONFEEDER.md.template
    ├── rule.md.template
    ├── skill.md.template
    ├── archive-README.md.template
    ├── slash-command.md.template        ← (v1.1) /<command>-style slash command
    ├── routine.md.template              ← (v1.3) standing-routine charter (Chapter 13)
    ├── hill-climb-skill.md.template     ← (v1.3) metric loop: iterate on X until it hits Y
    ├── hooks/                            ← (v1.1) optional hook-based hardening
    │   ├── surface-matching-rules.mjs.template      ← Pattern 1: PreToolUse rule-surfacing (reads native `paths:`)
    │   ├── correction-capture-prompt.mjs.template   ← Pattern 2: Stop correction-capture
    │   ├── build-gate.mjs.template                  ← Pattern 3: Stop build-gate (killed build = inconclusive)
    │   ├── lint-fix.mjs.template                    ← Pattern 4: PostToolUse lint-fix
    │   ├── doc-freshness-gate.mjs.template          ← (v1.2) Pattern 5: production push ⇒ docs freshened
    │   ├── settings.json.snippet                    ← sample wiring for all five
    │   └── HOOKS.md.template                        ← orientation note for docs/ai-context/HOOKS.md
    └── workspace/                        ← (v1.2) the multi-repo layer (Chapter 12)
        ├── bootstrap.sh.template            ← (v1.2) creates the whole layer from a filled workspace.json
        ├── CLAUDE.md.template · workspace.json.template · gitignore.template
        ├── orchestrator-agent.md.template · contract-guardian-agent.md.template · service-mapper-agent.md.template
        ├── cross-repo-contracts.md.template · SERVICE_MAP.md.template · CONTRACTS.md.template
        └── sync-repos.sh.template · delegate.sh.template · return-schema.json · settings.json.snippet
```

---

## Quick start — pick your scenario

### Scenario A — Brand new project (greenfield)

```bash
# 1. Install Claude Code (if not already)
#    https://docs.claude.com/en/docs/claude-code

# 2. Open Claude Code in your new project root
cd ~/path/to/new-project
claude

# 3. Paste the BOOTSTRAP prompt directly (no inventory needed for greenfield)
#    Open prompts/BOOTSTRAP-PROMPT.md, copy contents, paste into Claude Code,
#    replace <framework path> with the absolute path to this repo on your machine
```

For greenfield, you can skip the inventory step and tell Claude what specialists you want directly:

```
Set up the Claude Orchestration Framework for a new <Next.js + Postgres + Stripe>
project named "<your-project-name>".

Specialists I want:
  - <project-slug>-orchestrator
  - frontend-ui
  - backend-api
  - database
  - payments
  - auth-security
  - qa-functional
  - release-devops
  - security-privacy (REVIEW-ONLY)

Use templates from <absolute path to this repo>/templates/.
Show me each generated file before saving.
```

Estimated time: **45 min - 1 hour**.

### Scenario B — Existing project (brownfield)

⚠ **If your project ALREADY has Claude Code configuration** (e.g. you ran `/init`, or your team has been adding to `CLAUDE.md` for months), the bootstrap will detect this and ask before overwriting. The framework includes mandatory pre-flight safety checks:

- **Pre-flight 1** — auto-snapshot existing config to `.claude-pre-bootstrap-backup/`
- **Pre-flight 2** — naming-collision check (per file — STOP for explicit user decision)
- **Pre-flight 3** — `paths:` glob conflict check
- **Pre-flight 4** — drift detection on existing `CLAUDE.md`
- **Pre-flight 5** — existing agent style detection
- **Decision gate** — STOP if any pre-flight raised a `<NEEDS USER CONFIRMATION>` flag

You can also do an extra manual snapshot before starting:
```bash
mkdir -p .claude-pre-bootstrap-backup
[ -f CLAUDE.md ] && cp CLAUDE.md .claude-pre-bootstrap-backup/
[ -d .claude ] && cp -r .claude .claude-pre-bootstrap-backup/
```

The framework's BOOTSTRAP-PROMPT will repeat the snapshot inside the prompt — this is just belt-and-suspenders.

```bash
# 1. Install Claude Code (if not already)

# 2. Open Claude Code in the existing project's root
cd ~/path/to/existing-project

# 3. Make sure git is clean — bootstrap creates new files; you want a clean baseline
git status
git checkout -b setup/claude-orchestration

# 4. Run the inventory pass (read-only — proposes what to build)
claude
# Paste: prompts/INVENTORY-PROMPT.md
# Replace <framework path> with the absolute path to this repo on your machine
# INVENTORY scans for existing CLAUDE.md, .claude/agents/, .claude/rules/, .claude/skills/

# 5. Review/adjust Claude's proposed inventory + answer Open Questions

# 6. Run the bootstrap pass (creates all framework files)
# Paste: prompts/BOOTSTRAP-PROMPT.md (in the same Claude Code session)
# BOOTSTRAP runs pre-flight safety checks BEFORE creating any files
# If existing config is detected, you'll be asked to confirm per-file decisions

# 7. Verify (in terminal)
claude agents                  # should list all your project specialists (or /agents inside a session)
[ -f .claude-pre-bootstrap-backup/CLAUDE.md ] && diff .claude-pre-bootstrap-backup/CLAUDE.md CLAUDE.md
npm run build                  # or whatever your build is — should still pass

# 8. Run a real task via the orchestrator
claude --agent <project-slug>-orchestrator
# Give it a real bug or small feature; verify the handoff schema works end-to-end

# 9. Commit and PR (note: .claude-pre-bootstrap-backup/ is gitignored)
git add .claude/ docs/ai-context/ docs/_archive/ docs/*_BACKLOG.md CLAUDE.md .gitignore
git commit -m "chore: bootstrap Claude Code orchestration framework"
git push -u origin setup/claude-orchestration
gh pr create --base <your-base-branch> --title "Bootstrap Claude orchestration framework"
```

Estimated time: **2-4 hours** (split across phases — see `docs/09-RUNBOOK.md`).

### Scenario C — Just want to read the framework

Read the **PDF** (`Claude-Orchestration-Framework.pdf`) — 57 pages, self-contained **as of v1.1.2** (it does not yet include Chapters 11–12; regenerate is tracked in the changelog). Or browse the markdown files in `docs/` for clickable cross-links.

The most actionable single chapter is **`docs/00-QUICKSTART.md`** — every step with *what / why / paste this / you know it worked when*, through the multi-repo workspace. `docs/09-RUNBOOK.md` is its long form.

---

## Prerequisites

- **Claude Code** installed and authenticated. Verify with `claude --version` (need 2.x or later). [Install guide](https://docs.claude.com/en/docs/claude-code).
- **Git** installed (any recent version).
- **Optional but recommended:** GitHub CLI (`gh`) for PR creation.

No other dependencies. The framework is markdown + Claude Code's built-in subagent features.

## What this framework does NOT include

- **Runtime hooks** (in the default install). Documentation enforcement covers ~95% of value at 5% of complexity. **As of v1.1**, optional hook-hardening templates ship at `templates/hooks/` for projects where documentation discipline has demonstrably failed — see `docs/10-HOOK-HARDENING.md` for when and how to add them.
- **Code generation.** This is purely a documentation + agent-config framework. Your application code stays untouched.
- **Vendor lock-in.** No external services, no SaaS, no API keys. Just markdown.
- **Project-specific specialists.** The templates have placeholders. You customize per project.

## Customizing the framework for your project

Three things change per project:

1. **Project name + slug** (used to name the orchestrator: `<slug>-orchestrator`)
2. **Specialist list** (typically 4-10 specialists matching your project's domain boundaries — sizing table in `docs/03-AGENTS-GUIDE.md`)
3. **Path globs in rules** (matched to your actual code paths)

Everything else (the handoff schema, the universal evidence rule, the failure_condition pattern, the three-tier docs structure, the invocation modes) stays the same across projects.

See **`docs/03-AGENTS-GUIDE.md`** for how to pick your specialist list and **`docs/05-RULES-AND-SKILLS.md`** for how to write rules.

## Maintenance

After bootstrapping, the framework needs minimal upkeep:

- **Per task:** rules accumulate naturally as production teaches you new gotchas (add 1-2 per month is normal)
- **Per quarter:** run `prompts/REFINEMENT-PROMPT.md` to audit specialist scope drift, archive stale docs, re-check platform drift (Pitfall 18), re-stamp `PROJECT.md`, and decide if hooks should be added
- **Per major refactor:** the `context-librarian` specialist (if you spawned one) handles docs reorganization

## Where this came from

Battle-tested on a production codebase (as of 2026-08) with:
- 12 specialist agents, 14 path-scoped rule files, 10 repeatable workflows, 1 slash command, 5 hooks
- Three-tier doc organization (~21 orientation maps + canonical refs + frozen archive) plus the v1.2 project-truth set (`PROJECT.md`, `LEARNINGS.md`, per-area backlogs, glossary)
- Bidirectional handoff schema enforced via specialist refusal contracts
- Two clients (web + mobile) of one backend — the origin of the multi-client parity rules behind Chapter 12

The cascading-hallucination defense (`failure_condition`), the universal evidence rule, and (v1.2) the evidence-confidence taxonomy + "deferred work must be written" are the framework's novel contributions beyond stock Claude Code. Everything project-specific was stripped on the way in: if a lesson could not be restated as "any team, any stack, any domain hits this", it stayed out.

## Frequently asked

**Q: Is this Anthropic-official?**
No. This is a community framework built on top of Claude Code's documented subagent features. No special access required.

**Q: Will it work with the Claude API directly (not Claude Code)?**
The orchestrator-worker pattern and handoff schema concepts translate, but the runtime enforcement (`tools`, `maxTurns`, `Agent(...)` allowlists) is Claude Code specific. For pure API usage, you'd need to implement those with hooks or middleware.

**Q: Does it work with GitHub Copilot / Cursor / other AI assistants?**
For GitHub Copilot, use the companion edition below — same schema, same principles, Copilot's own surfaces (`.github/agents`, `.github/instructions`, `.github/skills`, `.github/hooks`). Running both editions in one repo is supported: VS Code reads `.claude/rules`, `.claude/agents`, `.claude/skills` and `.claude/settings.json` hooks natively, so one corpus can serve two thin routers. For other assistants the documentation patterns port; the runtime enforcement (`tools` allowlists, subagent isolation, hooks) does not.

**Q: Can I use this on a monorepo?**
Yes. See `docs/02-ARCHITECTURE.md` § "Variations" for monorepo, multi-product, and mobile+web split patterns.

**Q: We have one repo per microservice plus separate web and mobile repos. Do we need agents at every level, or one top-level orchestrator?**
Both, in layers — and not a separate framework. Every repo keeps its own install (layer 1); shared specialists become a plugin instead of copies (layer 2); a workspace repo with gitignored clones holds *only* the cross-repo orchestrator, the service map and the contract rules (layer 3), and delegates writes to each child's own orchestrator in the child's own session so the child's hooks still fire. Full design, verified platform behaviour, and a one-afternoon POC recipe: `docs/12-MULTI-REPO-WORKSPACES.md` + `templates/workspace/`.

**Q: I'm worried about leaking sensitive code.**
The framework doesn't change Claude Code's data handling. Use Claude Code's `permissions.deny` rules for sensitive paths. See `docs/09-RUNBOOK.md` § "What if I'm worried about leaking sensitive code".

**Q: Can I share this with another engineer who isn't on my team?**
This repo is public. The framework is free to use — see `LICENSE`.

## Companion editions

All three share the handoff schema, the evidence rule, the three-tier docs, the project-truth set and the multi-repo workspace layer; they differ only in the tool's surfaces. Released together on 2026-08-22.

- [`github-copilot-orchestration-framework`](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) v1.2.0 — GitHub Copilot (VS Code, CLI, cloud agent).
- [`copilot-ios-orchestration-framework`](https://github.com/abhinavsehgal/copilot-ios-orchestration-framework) v1.1.0 — Copilot, pre-filled for native iOS.

## License

MIT (see `LICENSE`). Use freely. Adapt freely. Attribution appreciated but not required.

## Contributing back

If you discover a missing pitfall, a better template pattern, or a useful new prompt, edit the markdown files and commit. The framework improves as it sees more projects.

If you've used it on a project and want to share what worked / didn't, open an issue on this repo with notes — useful for evolving the templates.
