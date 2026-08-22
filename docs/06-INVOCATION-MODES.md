# 06 — Invocation Modes

Three modes coexist by design. Pick by task shape, not by habit.

## Mode 1: Default `claude` (no flags)

Use for:
- Quick questions about code ("what does X do?", "where is Y defined?")
- Single-line copy / typo fixes
- Pure read tasks (review a file, summarize a function)
- One-file bug fixes where the rule + specialist are obvious from the task description
- Iterative debugging where you want low-friction back-and-forth

What you get: standard Claude Code main session. Full default tool access (Read, Edit, Write, Bash, all MCP). No `maxTurns` cap. Can spawn any subagent including built-ins (Explore, Plan, general-purpose) and plugin agents.

How to confirm: startup header shows `Claude Code` without an agent name.

## Mode 2: Explicit orchestrator `claude --agent <project>-orchestrator`

Use for:
- Anything spanning more than one role/domain
- Anything spanning more than one feature area
- Data integrity, payments, security/privacy, compliance work
- Bug investigations where root cause may cross layers (UI → API → DB → infra)
- Feature builds where acceptance criteria need writing before implementation
- Pre-release QA across multiple roles
- Anything where the scope is ambiguous

What you get: orchestrator's full discipline as runtime constraints — `tools` allowlist (no direct Edit/Write; must delegate), `maxTurns` cap, `Agent(...)` restricted to project specialists only. The structured handoff schema is enforced via the orchestrator's instructions.

How to confirm: startup header shows `@<project>-orchestrator`. Confirm visually before starting any production-sensitive task.

## Mode 3: Specialist mode `claude --agent <specialist-name>`

Also valid for narrow work in a single domain.

Examples:
- `claude --agent frontend-ui` — UI session with browser MCP + Edit/Write
- `claude --agent backend-api` — API work with full data tooling
- `claude --agent qa-functional` — QA pass with browser MCP + test runners
- `claude --agent legal-compliance` — review-only session
- `claude --agent database` — schema/migration work

Specialist sessions inherit the specialist's `tools` allowlist and `maxTurns` cap. They cannot delegate further (subagents do not spawn other subagents per Claude Code semantics). Use when you know exactly which domain the work lives in and don't need cross-domain coordination.

## Mode 4: Headless `claude -p` (scripts, CI, cross-repo delegation)

`claude -p "<prompt>" --agent <name> --output-format json` runs one turn non-interactively and
exits. Verified 2026-08-22: without `--bare`, a `-p` session loads the working directory's
`CLAUDE.md`, rules, agents, skills **and runs its hooks** — so it is a faithful way to run a repo's
orchestrator from a script or from another repo's session (Chapter 12, Mechanism A). `--json-schema`
forces the answer into `structured_output` (use it for the `return:` block); `session_id` in the
JSON lets you `--resume` later. `-p` never prompts, so pre-approve tools with `--allowedTools` or a
`--permission-mode`, and keep that list narrow. `--bare` is for CI lint-style calls that should load
*nothing* from the host; never use it when you want the repo's setup.

## Mode 5: Dynamic workflows (many-agent sweeps)

Script-driven orchestration — `agent()`, `parallel()`, `pipeline()` — triggered with the
`ultracode` keyword. Use for repeatable fan-outs (review every changed file across N dimensions,
migrate 40 call sites, exhaustive research). It complements, not replaces, the orchestrator: the
orchestrator decides *what* to do and issues schema-bound handoffs; a workflow is how it runs twenty
of them at once. Budget accordingly — a workflow can spawn dozens of agents.

## Why we don't make the orchestrator the default

Setting `agent: "<orchestrator-name>"` in `.claude/settings.json` would force every session into orchestrator mode, which has:

1. **No Edit/Write tools** — every code change requires `Agent(specialist)` delegation. Heavy friction for typos and one-line fixes.
2. **`maxTurns` cap restrictive** for interactive iterative work that often runs 50+ turns.
3. **Body prompt designed as a routing playbook**, not a general-purpose system prompt.
4. **Default Claude Code system prompt is replaced entirely** ([per official docs](https://code.claude.com/docs/en/sub-agents)).

The friction cost on everyday work outweighs the marginal enforcement gain. Real always-on enforcement of orchestrator discipline belongs to **hooks** or **`permissions.deny` rules** — both are deferred until you have run 2–3 real tasks through the explicit orchestrator path and confirmed the schema works under load.

Until then: pick the right mode for the task. If you are about to do anything cross-domain or production-sensitive and you are NOT in `--agent <orchestrator-name>`, stop and start a new session with the flag.

## Practical recipe

A well-functioning team uses all three modes:

```bash
# Quick fix (90% of casual work)
cd ~/your-project
claude
# → "Update the button label from 'Sign In' to 'Sign in'"

# Specialist deep-dive (focused domain work)
cd ~/your-project
claude --agent frontend-ui
# → "Audit our Suspense boundaries for client-component leaks"

# Full orchestrated work (multi-domain, production-sensitive)
cd ~/your-project
claude --agent acme-orchestrator
# → "Add a new payment method that requires KYC review and cron-driven activation"
```

## Anti-patterns

❌ **Using `claude` for everything.** Cross-domain work flooded with context. Specialists never invoked. Defeats the point.

❌ **Using `claude --agent <orchestrator>` for typos.** Forced delegation overhead. Specialist spawn for a one-line fix.

❌ **Not knowing which mode you're in.** Always check the startup header. If you're not sure, you're probably in the wrong mode.

## When you change projects

Each project has its own `.claude/agents/`, so `claude --agent <orchestrator-name>` is project-specific. The orchestrator name should be `<project>-orchestrator` (e.g. `acme-orchestrator`, not just `orchestrator`) so that when you have multiple projects open simultaneously, you can tell at a glance which orchestrator is active.
