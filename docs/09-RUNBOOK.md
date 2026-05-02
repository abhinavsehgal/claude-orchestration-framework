# 09 — Runbook: Bootstrap a New Project

Step-by-step guide for adopting this framework in a real codebase. Plan for ~2-4 hours of focused work.

## Prerequisites

- Claude Code installed (`claude --version` should print v2.x or later)
- A project repository (any tech stack)
- Read access to this framework at `~/Desktop/claude-orchestration-framework/`
- 2-4 hours of focused time (don't rush this; the foundation lasts years)

## Phase 0 — Decide if you need this framework (10 min)

Skip this framework if:
- Your project is a solo prototype
- Your project has fewer than 4 distinct domain areas (frontend, backend, etc.)
- Your team uses Claude Code for the occasional question, not as a development partner
- Your project is < 5,000 lines of code

Use this framework if:
- Your project has multiple roles (admin, customer, partner, etc.)
- Multiple data systems are in play (DB + cache + search + queue)
- Real compliance/security/payments concerns exist
- A team uses Claude Code daily on the same codebase
- You've experienced cascading hallucinations or context flooding

## Phase 1 — Inventory your project (30 min)

Open Claude Code in your project root:

```bash
cd ~/your-project
claude
```

Paste `prompts/INVENTORY-PROMPT.md` (from this framework). Claude will:
- Scan your codebase structure
- Propose a list of specialists based on your domain boundaries
- Surface clutter that should be archived
- Identify your protected branches (main, develop, etc.)
- Identify your tech stack and which MCP servers will be useful

You answer: confirm/adjust the proposed specialist list. This is the foundation of everything that follows.

## Phase 2 — Bootstrap the framework files (45 min)

In the same Claude Code session, paste `prompts/BOOTSTRAP-PROMPT.md`. Reference this framework's path so Claude can read templates:

```
The framework lives at /Users/<you>/Desktop/claude-orchestration-framework/.
Read templates from there. Customize for this project: <project-name>.
```

Claude will:
1. Create `.claude/agents/<project>-orchestrator.md` from `templates/orchestrator-agent.md.template`
2. Create one `.claude/agents/<specialist>.md` per specialist, from the appropriate template
3. Create `docs/ai-context/HANDOFF_SCHEMA.md` from the template
4. Create `docs/ai-context/INDEX.md` with task → docs map
5. Create `docs/ai-context/ORCHESTRATION_SPOONFEEDER.md` with usage guide
6. Create skeleton `docs/ai-context/<area>-experience.md` files (one per major domain)
7. Create `CLAUDE.md` at root (the router)
8. Create `docs/_archive/README.md`
9. Update `.gitignore` with framework-relevant entries

You verify each file before saving. Claude shouldn't ship anything you haven't read.

## Phase 3 — Add your first rules (30 min)

The hardest part of this framework is writing useful rules. They take real domain knowledge.

In the Claude Code session, ask:

```
Based on the codebase scan, propose 3-5 rule files for .claude/rules/.
For each, list the path globs (applies_to:) and 2-3 hard rules with WHY + HOW TO APPLY.
Do not create any rule that's just style preference — only invariants where breaking them caused or could cause a real production issue.
Wait for my approval before creating any files.
```

Review each proposal. Approve, edit, or reject. Common rule files to start with:
- `backend-api.md` (or your equivalent server-side concern)
- `frontend-ui.md` (or `mobile-ui.md`)
- `database.md`
- `auth-security.md`
- `testing.md`

Aim for 3-5 rules per file at first. You'll add more over time as production teaches you new gotchas.

## Phase 4 — Add 1-2 starter skills (20 min)

Most projects benefit from at least these skills:
- `investigate-bug/SKILL.md`
- `build-feature/SKILL.md`

Ask Claude:

```
Create .claude/skills/investigate-bug/SKILL.md and .claude/skills/build-feature/SKILL.md based on templates/skill.md.template, customized for this project's tech stack and roles.
```

You can add more skills later as patterns recur (qa-flow, compliance-review, audit-pipeline, context-refactor).

## Phase 5 — Verify (15 min)

Run these checks:

```bash
# All agents register
claude agents

# Doc-link sweep — find broken refs
grep -rohE "(docs|\.claude)/[A-Za-z0-9_/-]+\.md" CLAUDE.md docs/ .claude/ | sort -u | while read p; do [ -f "$p" ] || echo "BROKEN: $p"; done

# Build still passes (whatever your build command is)
npm run build  # or: pytest, go build, cargo build, etc.
```

Fix any breakage before moving on. Common issues:
- Agent file YAML invalid (claude agents will tell you)
- Markdown link in CLAUDE.md or INDEX.md to a file that doesn't exist (typo)
- Rule glob points to a path that doesn't match anything (mostly harmless but confusing)

## Phase 6 — Run a real task through the orchestrator (30 min)

Pick a real task — ideally a small bug or a small feature. Open a NEW Claude Code session in orchestrator mode:

```bash
cd ~/your-project
claude --agent <project>-orchestrator
```

Verify the startup header shows `@<project>-orchestrator`.

Give it a real task. Watch what happens:
- Does the orchestrator emit the outbound handoff YAML block?
- Do specialists return the inbound YAML block?
- Does the orchestrator catch any rejected_claims correctly?
- Does verification (tests, build) actually run?

If anything is off:
- Tighten the orchestrator's "outbound discipline" section
- Tighten the specialist's "incoming handoff validation" section
- Add a missing rule
- Add a missing orientation map

Don't add hooks yet. Get the documentation enforcement working through 2-3 real tasks first.

## Phase 7 — Document for your team (20 min)

Add to your project's existing onboarding docs:
- "We use Claude Code with a custom orchestration setup. See `docs/ai-context/ORCHESTRATION_SPOONFEEDER.md`."
- "Always pick the right invocation mode: `claude` for quick work, `claude --agent <orchestrator>` for cross-domain work."
- "Don't push to main without project-owner approval."

Optional: announce in your team chat with a 30-second demo of the orchestrator handling a multi-step task.

## Phase 8 — Add hardening as needed (optional, after 2 weeks)

After running real tasks for a couple weeks, decide if you need:

### Hop limits via PreToolUse hook

If you see the orchestrator delegating to the same specialist 3+ times in a row without progress, add a hook that counts Agent invocations per turn and warns past N hops.

### Hard schema enforcement via PreToolUse hook

If specialists keep emitting malformed return blocks, add a hook that validates the YAML before letting the call complete.

### `permissions.deny` for built-ins/plugins on main thread

If team members keep accidentally invoking `Explore`/`Plan`/plugin agents from default `claude` sessions when they should be in orchestrator mode, add to `.claude/settings.json`:

```json
{
  "permissions": {
    "deny": [
      "Agent(Explore)",
      "Agent(Plan)",
      "Agent(general-purpose)",
      "Agent(<plugin>:*)"
    ]
  }
}
```

### Per-agent memory for cross-session learning

For implementation specialists where institutional knowledge accumulates (debugging recipes, schema gotchas), add `memory: project` to the agent frontmatter. **Do NOT enable for REVIEW-ONLY agents** — see Pitfall 2.

## Phase 9 — Quarterly maintenance

Every 3 months, dedicate a session to:
1. Run the `context-refactor` skill (or just have the `context-librarian` specialist do a sweep)
2. Move stale dated reports to `docs/_archive/<YYYY-MM>/`
3. Reconcile any rule conflicts
4. Update orientation maps for areas where reality has drifted
5. Verify all agents still register after dependency updates

## Common time-sinks to avoid

- **Don't try to make every rule perfect on day 1.** Ship 5 rules; add 1 per month as production teaches you.
- **Don't over-specialize.** 6-8 specialists is plenty. Splitting too narrowly creates routing confusion.
- **Don't write canonical docs from scratch in this framework.** Use what you already have. The framework adds the orientation/router layer on top.
- **Don't manually merge agent files when adding a rule.** Always edit the rule file; agents reference it via `applies_to`.

## When to stop and ask

- If `claude agents` shows fewer agents than you expect → YAML frontmatter syntax error
- If specialist outputs don't include the return YAML block → the specialist's body needs the "Return schema" section
- If the orchestrator keeps delegating in a loop → check `failure_condition` is articulated
- If a specialist refuses a delegation → check the orchestrator's outbound block has all required fields

## End state

You have:
- A `<project>-orchestrator` and 4-10 specialists
- 3-5 rule files covering your highest-risk paths
- 2-4 skills for recurring workflows
- Three-tier docs (orientation maps + canonical refs + archive)
- A team that knows how to pick invocation modes
- Auditable handoffs for every cross-domain task

Time invested: 2-4 hours setup + ~30 min/quarter maintenance.

Time saved: every cascading-hallucination incident you DIDN'T have. Every cross-domain task that got the right specialists without you thinking about it. Every new engineer who could navigate the codebase without 2 weeks of pairing.
