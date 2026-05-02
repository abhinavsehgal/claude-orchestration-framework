# Claude Orchestration Framework

> **Purpose.** A reusable multi-agent orchestration setup for [Claude Code](https://docs.claude.com/en/docs/claude-code) that prevents cascading hallucinations, enforces evidence-based handoffs between agents, and makes Claude usable on production codebases by teams. Tech-stack agnostic — drops into any project (web, mobile, backend, ML, infra) in 2-4 hours.

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
├── Claude-Orchestration-Framework.pdf  ← consolidated 57-page printable
├── LICENSE
│
├── docs/                                ← the framework explained (9 chapters)
│   ├── 01-PRINCIPLES.md                 ← seven core principles
│   ├── 02-ARCHITECTURE.md               ← .claude/ + docs/ layout (with monorepo / mobile+web / multi-product variants)
│   ├── 03-AGENTS-GUIDE.md               ← how to design orchestrator + specialists (per-stack tables)
│   ├── 04-HANDOFF-SCHEMA.md             ← bidirectional schema + worked examples (web / mobile / REVIEW-ONLY)
│   ├── 05-RULES-AND-SKILLS.md           ← path-globbed rules + repeatable workflows
│   ├── 06-INVOCATION-MODES.md           ← claude vs --agent vs specialist mode
│   ├── 07-FOLDER-STRUCTURE.md           ← three-tier doc organization
│   ├── 08-COMMON-PITFALLS.md            ← 15 hard-won lessons
│   └── 09-RUNBOOK.md                    ← step-by-step bootstrap (~2-4 hours)
│
├── prompts/                             ← ready-to-paste prompts for Claude Code
│   ├── INVENTORY-PROMPT.md              ← scan + propose specialists (run first)
│   ├── BOOTSTRAP-PROMPT.md              ← generate all framework files (run second)
│   └── REFINEMENT-PROMPT.md             ← post-bootstrap hardening (run after 2-3 real tasks)
│
└── templates/                           ← drop-in templates with placeholders
    ├── orchestrator-agent.md.template
    ├── specialist-agent.md.template
    ├── review-only-agent.md.template
    ├── HANDOFF_SCHEMA.md.template
    ├── INDEX.md.template
    ├── SPOONFEEDER.md.template
    ├── rule.md.template
    ├── skill.md.template
    └── archive-README.md.template
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

# 5. Review/adjust Claude's proposed inventory + answer Open Questions

# 6. Run the bootstrap pass (creates all framework files)
# Paste: prompts/BOOTSTRAP-PROMPT.md (in the same Claude Code session)

# 7. Verify (in terminal)
claude agents                  # should list all your project specialists
npm run build                  # or whatever your build is — should still pass

# 8. Run a real task via the orchestrator
claude --agent <project-slug>-orchestrator
# Give it a real bug or small feature; verify the handoff schema works end-to-end

# 9. Commit and PR
git add .claude/ docs/ai-context/ docs/_archive/ CLAUDE.md .gitignore
git commit -m "chore: bootstrap Claude Code orchestration framework"
git push -u origin setup/claude-orchestration
gh pr create --base <your-base-branch> --title "Bootstrap Claude orchestration framework"
```

Estimated time: **2-4 hours** (split across phases — see `docs/09-RUNBOOK.md`).

### Scenario C — Just want to read the framework

Read the **PDF** (`Claude-Orchestration-Framework.pdf`) — 57 pages, fully self-contained. Or browse the markdown files in `docs/` for clickable cross-links.

The most actionable single chapter is **`docs/09-RUNBOOK.md`**.

---

## Prerequisites

- **Claude Code** installed and authenticated. Verify with `claude --version` (need 2.x or later). [Install guide](https://docs.claude.com/en/docs/claude-code).
- **Git** installed (any recent version).
- **Optional but recommended:** GitHub CLI (`gh`) for PR creation.

No other dependencies. The framework is markdown + Claude Code's built-in subagent features.

## What this framework does NOT include

- **Runtime hooks.** Documentation enforcement covers ~95% of value at 5% of complexity. Hooks are added later (per `docs/08-COMMON-PITFALLS.md` recommendations) IF documentation enforcement demonstrably fails.
- **Code generation.** This is purely a documentation + agent-config framework. Your application code stays untouched.
- **Vendor lock-in.** No external services, no SaaS, no API keys. Just markdown.
- **Project-specific specialists.** The templates have placeholders. You customize per project.

## Customizing the framework for your project

Three things change per project:

1. **Project name + slug** (used to name the orchestrator: `<slug>-orchestrator`)
2. **Specialist list** (5-10 specialists matching your project's domain boundaries)
3. **Path globs in rules** (matched to your actual code paths)

Everything else (the handoff schema, the universal evidence rule, the failure_condition pattern, the three-tier docs structure, the invocation modes) stays the same across projects.

See **`docs/03-AGENTS-GUIDE.md`** for how to pick your specialist list and **`docs/05-RULES-AND-SKILLS.md`** for how to write rules.

## Maintenance

After bootstrapping, the framework needs minimal upkeep:

- **Per task:** rules accumulate naturally as production teaches you new gotchas (add 1-2 per month is normal)
- **Per quarter:** run `prompts/REFINEMENT-PROMPT.md` to audit specialist scope drift, archive stale docs, and decide if hooks/memory should be added
- **Per major refactor:** the `context-librarian` specialist (if you spawned one) handles docs reorganization

## Where this came from

Battle-tested on a production K-12 codebase with:
- 11 specialist agents (frontend, backend, real-time video, payments, security, legal-compliance, QA, DevOps, AI/ML pipeline, product-flow, docs)
- 7 path-globbed rule files
- 6 repeatable workflows
- Three-tier doc organization (~21 orientation maps + 17 canonical refs + frozen archive)
- Bidirectional handoff schema enforced via specialist refusal contracts

The cascading-hallucination defense (`failure_condition`) and the universal evidence rule are the framework's two biggest novel contributions beyond stock Claude Code.

## Frequently asked

**Q: Is this Anthropic-official?**
No. This is a community framework built on top of Claude Code's documented subagent features. No special access required.

**Q: Will it work with the Claude API directly (not Claude Code)?**
The orchestrator-worker pattern and handoff schema concepts translate, but the runtime enforcement (`tools`, `maxTurns`, `Agent(...)` allowlists) is Claude Code specific. For pure API usage, you'd need to implement those with hooks or middleware.

**Q: Does it work with Cursor / GitHub Copilot / other AI assistants?**
The documentation patterns are portable, but `tools` allowlists and subagent isolation aren't. You'd lose most of the runtime enforcement layer.

**Q: Can I use this on a monorepo?**
Yes. See `docs/02-ARCHITECTURE.md` § "Variations" for monorepo, multi-product, and mobile+web split patterns.

**Q: I'm worried about leaking sensitive code.**
The framework doesn't change Claude Code's data handling. Use Claude Code's `permissions.deny` rules for sensitive paths. See `docs/09-RUNBOOK.md` § "What if I'm worried about leaking sensitive code".

**Q: Can I share this with another engineer who isn't on my team?**
This repo is private. Add them as a collaborator on GitHub, or zip the folder and email it. The framework itself is free to use — see `LICENSE`.

## License

MIT (see `LICENSE`). Use freely. Adapt freely. Attribution appreciated but not required.

## Contributing back

If you discover a missing pitfall, a better template pattern, or a useful new prompt, edit the markdown files and commit. The framework improves as it sees more projects.

If you've used it on a project and want to share what worked / didn't, open an issue on this repo with notes — useful for evolving the templates.
