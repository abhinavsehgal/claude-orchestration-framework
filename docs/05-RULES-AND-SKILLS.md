# 05 — Rules and Skills

Two patterns for capturing knowledge that doesn't belong in agents: **rules** (path-globbed invariants) and **skills** (repeatable workflows).

## Rules

A rule is a markdown file in `.claude/rules/` with frontmatter declaring which file paths it applies to. When an agent is about to edit a matching file, it must read the rule first.

### When to write a rule

Write a rule when:
- A bug shipped to production once and must not recur
- A pattern is non-obvious from reading the code (e.g. "this column looks like a string but it's a JSON-encoded string in legacy rows")
- A constraint crosses multiple files but isn't enforced by types/tests
- An anti-pattern is tempting but wrong (e.g. "this looks like dead code but is actually called via reflection")

Don't write a rule for:
- Things obvious from the code (well-named identifiers do that work)
- Style preferences (lint config does that work)
- One-off decisions (PR descriptions do that work)
- Anything covered by an existing rule (extend the existing one)

### Rule file structure

```markdown
---
name: <domain-name>
description: Scoped rules for <domain>. Read before editing any of the path globs below.
applies_to:
  - "<glob1>"
  - "<glob2>"
---

# <Domain Name> Rules

## Hard rules

### <Rule title — short imperative>

**Why:** <one-sentence reason — usually a past incident or invariant>

**How to apply:** <2-4 sentences — what to do, what helper to use, what to check>

### <Next rule>

...

## What NOT to do

- Bullet list of anti-patterns

## How to use this file

1. If you're editing any path in `applies_to`, read this file first.
2. Report which rule files were read in your task summary.

## Cross-links

- `<related orientation map>`
- `<related canonical doc>`
- `<related rule>`
```

### Glob patterns

Use standard shell globs. Examples:

```yaml
applies_to:
  - "src/api/**/*.ts"                      # all TypeScript under src/api/
  - "src/db/migrations/**/*.sql"           # all migration SQL files
  - "src/{lib,utils}/auth-*.ts"            # auth-related lib/util files
  - "components/payment/**/*.{tsx,jsx}"    # payment components
  - "ios/Runner/**/*.swift"                # iOS Swift files
  - "lib/screens/**/*.dart"                # Flutter screens
  - "internal/handlers/**/*.go"            # Go API handlers
  - "app/models/*.rb"                      # Rails models
```

Globs are matched against the relative path from project root.

### Rule examples by tech

#### Backend / API

```markdown
### Always validate request body before reading database

**Why:** Three production incidents traced to unvalidated body data flowing to SQL parameters.

**How to apply:** Use `validateRequest(schema)` middleware in `src/lib/validation.ts`. Never read `req.body.<field>` directly in a handler.
```

#### Database / migrations

```markdown
### Migration order is sacred — never reorder, never edit applied migrations

**Why:** Migrations apply in filename order. Editing an applied migration silently diverges staging from production.

**How to apply:** New migrations get the next sequential timestamp prefix. Bug fixes to applied migrations require a NEW migration that supersedes the old behavior.
```

#### Frontend / UI

```markdown
### Never trust component props for security checks — re-validate on the server

**Why:** A user can edit any prop via React DevTools.

**How to apply:** Server-side handlers re-validate authorization. Props are for rendering, not authorization.
```

#### Mobile

```markdown
### iOS auto-zooms on input focus when font-size < 16px

**Why:** iOS Safari WebView feature; jarring UX.

**How to apply:** Set `font-size: 16px` minimum on all `<input>`, `<select>`, `<textarea>`. For visual size adjustment, scale via padding/transform, not font-size.
```

#### Infra / DevOps

```markdown
### Never commit secrets — use env vars + secrets manager

**Why:** Once committed, secrets are in git history forever; rotation required.

**How to apply:** Add new secrets via `<your-secrets-manager>`. The deploy pipeline injects them. Never `git add` files matching `.env*`.
```

### Anti-patterns in rules

❌ **Long rationale paragraphs.** Rules should be ≤ 4 sentences per rule. Move long stories to the canonical doc.

❌ **Aspirational rules.** "We should always..." — if it's not enforced, it's not a rule. Either write it as enforced (with the helper / check) or move to a style guide.

❌ **Rules that contradict other rules.** Specialist confusion. Have the context-librarian agent reconcile.

❌ **Rules with no `applies_to`.** Without globs, agents have no signal to read it. Either add globs or move to an orientation map.

## Skills

A skill is a directory under `.claude/skills/` containing a `SKILL.md` file. Skills are **repeatable workflows** — task templates that recur frequently enough to deserve a checklist.

### When to write a skill

Write a skill when:
- The same kind of task arrives 3+ times
- The task has multiple steps with verification between them
- The cost of skipping a step is high (e.g. forgot to run security review before launching a new auth flow)
- A junior team member would benefit from a guided checklist

Don't write a skill for:
- One-off tasks
- Tasks where the steps vary too much per instance
- Things that are really just a single agent invocation

### Skill file structure

```markdown
---
name: <skill-name>
description: <One-sentence description of when to use this skill>
---

# Skill — <Skill Name>

## When to invoke

- Bullet list of trigger conditions

## Workflow

### 1. <Step name>

<2-4 sentences — what to do, what to produce, what to verify>

### 2. <Next step>

...

## Definition of Done

1. <Artifact 1>
2. <Artifact 2>
3. <Verification>

## Cross-links

- `<related agents>`
- `<related rules>`
```

### Common skills (most projects benefit from these)

#### `investigate-bug/SKILL.md`

Steps: restate bug → identify affected roles → read minimum docs → trace UI → state → API → DB → tests → match touched paths to rules → reproduce → classify severity → propose smallest fix → verify.

#### `build-feature/SKILL.md`

Steps: restate feature → identify roles → read minimum docs → run pre-feature checklist (data deltas, RLS, payment impact, etc.) → write acceptance criteria per role → identify rule-covered paths → plan smallest implementation → implement incrementally.

#### `qa-flow/SKILL.md`

Steps: identify flows in scope → read testing strategy + relevant orientation map → use browser MCP for UI flows → cover golden path + edge cases + adjacent regressions + cross-role → verify functional + data correctness → severity-rank findings.

#### `compliance-review/SKILL.md`

Steps: identify data involved → identify if regulated subjects involved (minors, PHI, EU residents, etc.) → read compliance orientation map + canonical privacy/terms → apply standing checklist (consent, retention, minimization, portability, deletability, processor coverage, disclosure, consent timing, cross-border, marketplace clauses) → flag risks per regulation → produce attorney checklist.

#### `audit-pipeline/SKILL.md`

For projects with quality-critical generation pipelines (AI question banks, data ETL, content moderation): trace lifecycle → verify schema → verify required fields on every write → verify validation layers → verify admin review surface → identify gaps → severity-rank findings.

#### `context-refactor/SKILL.md`

For maintaining the framework itself: read current INDEX + ledger → categorize suspect content (routing / orientation / rule / skill / agent / outdated / duplicate / conflicting) → move to correct home → consolidate duplicates → flag conflicts (don't silently resolve) → keep root CLAUDE.md small.

### Skills vs Rules vs Agents

| If the knowledge is... | Put it in... |
|---|---|
| A path-scoped invariant ("don't do X when editing Y") | Rule |
| A multi-step workflow that recurs | Skill |
| A persona with its own scope and Definition of Done | Agent |
| A one-time decision rationale | PR description |
| Long-form architecture explanation | Canonical doc |
| Quick orientation for an area | Orientation map (`docs/ai-context/<area>.md`) |

When something feels like it could be 2 of those: prefer rule > skill > agent (in that order). Rules are most precise (path-globbed). Skills are next (named workflow). Agents are heaviest.

## Naming and lifecycle

- Rules: lowercase with hyphens, `.md` suffix. Filename = `name`.
- Skills: lowercase with hyphens for the directory, `SKILL.md` inside.
- New rule from a new gotcha: add to existing rule file if domain matches; create new rule file only if it's a new domain.
- Rule that becomes obsolete (e.g. the underlying constraint was fixed): delete it, don't leave it as "obsolete." Stale rules cause confusion.
- Skill that's invoked < 3x per quarter: consider deleting; the workflow probably wasn't recurrent enough.
