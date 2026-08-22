# 03 — Agents Guide

How to design the orchestrator and specialists for your project.

## The orchestrator

Every project has exactly **one** orchestrator. Naming convention: `<project-name>-orchestrator` (e.g. `acme-orchestrator`, `payments-orchestrator`).

### Responsibilities

- Read incoming task. Restate it.
- Identify affected roles / domains.
- Read minimum docs (INDEX.md → 1-3 orientation maps).
- Pick the right specialists (typically 1-3 per task).
- Issue structured handoffs with claims + evidence + failure_condition.
- Receive returns. Verify rejected_claims. Re-verify unverified_claims before propagating.
- Aggregate findings into a final summary with the project's "Definition of Done."

### Constraints

- **No Edit/Write tools.** The orchestrator coordinates; specialists implement.
- **Restricted Agent allowlist.** Only project specialists, not built-ins or plugins.
- **Modest maxTurns.** 20-30 turns is typical (read INDEX → delegate 1-3 times → aggregate).

### Frontmatter template

```yaml
---
name: <project>-orchestrator
description: Main task dispatcher for <Project>. Use for medium/complex tasks. Picks minimum docs/agents/skills; never loads everything.
tools: Read, Grep, Glob, Bash, WebFetch, WebSearch, TodoWrite, Agent(<specialist1>, <specialist2>, <specialist3>, ...)
maxTurns: 25
---
```

## Specialists

A specialist owns one domain. Specialists are domain-shaped, not technology-shaped. Bad: `react-specialist`, `typescript-specialist`. Good: `frontend-ui`, `payments`, `realtime-sync`, `compliance`.

### How many do you need?

| Project size | Specialist count |
|---|---|
| Solo project / prototype | 0-2 (just use plain `claude`) |
| Small team / single product | 4-6 |
| Medium codebase / multiple roles | 6-10 |
| Large codebase / multi-product | 10-15 |

More than 15 is a smell — you're probably making them too narrow.

### Common specialists by project type

#### Web app (any stack)

| Specialist | Domain |
|---|---|
| `frontend-ui` | UI components, routing, state management, client-side interactions |
| `backend-api` | API routes, server logic, request handling |
| `database` | Schema, migrations, query layer, indexes |
| `auth-security` | Authentication, authorization, sessions, secrets |
| `qa-functional` | Browser/E2E testing, regression checks |
| `release-devops` | Build, deploy, env config, observability |

#### Mobile app

| Specialist | Domain |
|---|---|
| `mobile-ui` | Screens, navigation, gestures, platform-specific UX |
| `mobile-state` | State management, persistence, offline sync |
| `mobile-native` | Platform bridges (iOS/Android native code or Flutter plugins) |
| `backend-api` | Server-side endpoints the app calls |
| `qa-mobile` | Device testing, simulator/emulator flows |

#### Backend / API service

| Specialist | Domain |
|---|---|
| `api-routes` | HTTP/gRPC route handlers |
| `domain-logic` | Core business logic |
| `data-layer` | DB access, repositories, ORM patterns |
| `integrations` | Third-party API clients |
| `observability` | Logging, metrics, tracing |
| `release-devops` | Deploy, env, secrets |

#### ML / data project

| Specialist | Domain |
|---|---|
| `data-pipeline` | ETL, ingestion, transformation |
| `model-training` | Training scripts, hyperparam tuning, experiment tracking |
| `inference-serving` | Model serving, latency, batching |
| `data-quality` | Validation, anomaly detection, monitoring |

#### Cross-cutting (add to most projects)

| Specialist | Domain |
|---|---|
| `product-flow` | Acceptance criteria, cross-role behavior, requirement clarification (LIGHT EDITS ONLY) |
| `qa-functional` | Tests across the system |
| `security-privacy` | Auth/RLS/sensitive-data review (REVIEW-ONLY) |
| `legal-compliance` | Regulatory review for relevant laws (REVIEW-ONLY) |
| `release-devops` | Build/deploy/cron/env |
| `context-librarian` | Maintains `.claude/`, `docs/ai-context/`, archives drift |
| `docs-writer` | Canonical doc updates |

### REVIEW-ONLY specialists

Some specialists should explicitly **not** edit code. Tag them in the description, give them no Edit/Write tools, and document the constraint in their I CANNOT section.

Examples:
- `legal-compliance` — flags risks; never claims final legal advice
- `security-privacy` — identifies vulnerabilities; doesn't ship fixes
- `architecture-review` — reviews proposed designs; doesn't implement

Frontmatter for REVIEW-ONLY:
```yaml
---
name: legal-compliance
description: REVIEW-ONLY. Flags <list of regulations> risks. Produces attorney-checklist content. Never claims final legal advice.
tools: Read, Grep, Glob, WebFetch, WebSearch, TodoWrite
maxTurns: 12
---
```

Notably absent: `Edit`, `Write`, `Bash`. The harness physically prevents this agent from modifying files.

⚠ **Do NOT enable `memory:` on REVIEW-ONLY agents.** Per Claude Code docs, enabling `memory:` automatically grants Read/Write/Edit so the agent can manage its memory files — silently bypassing your REVIEW-ONLY contract.

### Implementation specialists

Get full Edit/Write/Bash access plus relevant MCP tools.

Frontmatter for an implementation specialist:
```yaml
---
name: backend-api
description: Builds and fixes API routes, server logic, request handling. Coordinates with database and auth-security specialists.
tools: Read, Edit, Write, Grep, Glob, Bash, TodoWrite
maxTurns: 20
---
```

For specialists that need MCP server access (e.g. browser testing), use the documented wildcard:
```yaml
tools: Read, Edit, Write, Grep, Glob, Bash, TodoWrite, mcp__chrome-devtools__*, mcp__playwright__*
```

The wildcard is supported and covers all current + future tools from that MCP server.

## Agent body structure (every agent)

Every agent file body should include these sections in order:

```markdown
# <Agent Display Name>

<2-sentence description of the agent's domain.>

## When to use

- Bullet list of task types this agent owns

## When NOT to use

- Bullet list of task types that belong to other agents

## Required reading

1. `docs/ai-context/INDEX.md` — task → docs map
2. <area-specific orientation map>
3. **`.claude/rules/<your-rule>.md`** — always (when applicable)
4. <other rules / canonical refs>

## Incoming handoff validation

<paste the standard incoming handoff validation block — see template>

## Return schema (required)

<paste the standard return schema block — see template>

## I CAN

- Bullet list of what this agent is allowed to do

## I CANNOT

- Bullet list of what this agent must refuse
- Always include: "Push to main" (or your equivalent protected branch)

## Definition of Done

1. Files changed (paths)
2. Rule files read before editing
3. Tests run
4. Risks remaining
5. (any agent-specific Definition of Done items)

## Cross-links

- `<related rule>`
- `<related orientation map>`
- `<canonical doc>`
```

The "Incoming handoff validation" and "Return schema" sections are **identical across all specialists** — they enforce the universal handoff contract. See `templates/specialist-agent.md.template`.

## Naming conventions

- Lowercase with hyphens: `backend-api`, `frontend-ui`, `legal-compliance`
- Filename matches `name`: `backend-api` → `backend-api.md`
- Be specific but not over-narrow: `payments` good, `stripe-webhook-handler` too narrow
- Cross-cutting concerns get their own agent: `qa-functional`, `security-privacy`, not folded into another specialist

## When to add a new specialist vs. extend an existing one

Add a new specialist when:
- The domain has its own gotchas (would warrant its own rule file)
- Tasks in that domain frequently arrive
- The existing specialists' "I CANNOT" sections would all reject this work

Extend an existing specialist when:
- The work overlaps with an existing specialist's domain
- The new domain is small (fits in 1-2 paragraphs of the existing agent's body)
- You haven't seen 3+ tasks in this domain yet

When in doubt: don't add. It's easier to split a specialist later than to delete one that accumulated cruft.

## Anti-patterns

❌ **Technology-named specialists.** `typescript-specialist` doesn't have a clear domain — what does it own that `frontend-ui` and `backend-api` don't already cover?

❌ **Layer-named specialists.** `database-specialist` is fine if "database" means a domain (schema, migrations, repository layer); not fine if it means "anyone touching SQL anywhere should consult me."

❌ **Specialists with overlapping `tools` and overlapping descriptions.** Confuses the orchestrator; it won't know which to pick.

❌ **Specialists with no `tools:` field.** They inherit everything, defeating defense-in-depth. Always declare `tools` explicitly.

❌ **Specialists that delegate to other specialists.** Subagents *can* spawn subagents (up to a configurable depth — see Pitfall 18), but a specialist that does so has become an un-audited orchestrator: the handoff schema, the evidence rule and the orchestrator's return-validation are all bypassed one level down. By framework convention a specialist returns to the orchestrator with `recommended_next_agent` and lets the orchestrator chain the work. The only sanctioned nesting is orchestrator → orchestrator (Chapter 12, multi-repo), where both hops carry the full schema.

❌ **Specialists whose "I CAN" includes "approve production deploys."** That's the project owner's call, not an agent's.
