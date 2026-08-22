# BOOTSTRAP PROMPT

Paste this AFTER you have completed the INVENTORY-PROMPT pass and approved the proposed specialists. Adjust `<framework path>` to match your install location.

---

I've reviewed the inventory. Now bootstrap the Claude Orchestration Framework on this project. Use the templates from `<framework path>/templates/` (wherever you cloned the framework — the quickstart uses `~/frameworks/claude`). Read those template files before generating anything.

---


> **Greenfield (no INVENTORY pass)?** The steps below refer to "inventory section N". If you skipped the
> inventory because the project is new, paste this block first and fill it — each item stands in for
> the corresponding inventory section:
>
> ```
> GREENFIELD ANSWERS
> 1. Identity: <one sentence — what the product is, for whom, the stack in five words>
> 2. Roles: <role → permission boundary, one line each>
> 3. Specialists: <repo-slug>-orchestrator + <4–8 domain-shaped specialists; mark REVIEW-ONLY ones>
> 4. Rule files: <domain → the path globs it guards, one line each>
> 5. Sources of truth: <concept → canonical helper/table, if any exist yet>
> 6. Build / test / dev commands: <copied from the real config, or "none yet">
> 7. Branches: base = <BASE_BRANCH>, production = <PROD_BRANCH>
> ```

## ⚠ PRE-FLIGHT SAFETY CHECKS (run BEFORE creating anything)

This project may already have some Claude Code configuration (e.g. from running `/init`, or from prior team work). Bootstrap is designed to coexist with existing config, but ONLY if you detect collisions first. Run these checks in order. Do NOT skip even if the inventory said the project was greenfield.

### Pre-flight 1 — Auto-snapshot (mandatory)

Before any write, create a backup of any existing Claude Code config:

```bash
mkdir -p .claude-pre-bootstrap-backup
[ -f CLAUDE.md ] && cp CLAUDE.md .claude-pre-bootstrap-backup/
[ -d .claude/agents ] && cp -r .claude/agents .claude-pre-bootstrap-backup/
[ -d .claude/rules ] && cp -r .claude/rules .claude-pre-bootstrap-backup/
[ -d .claude/skills ] && cp -r .claude/skills .claude-pre-bootstrap-backup/
[ -f .claude/settings.json ] && cp .claude/settings.json .claude-pre-bootstrap-backup/
[ -f .claude/settings.local.json ] && cp .claude/settings.local.json .claude-pre-bootstrap-backup/
[ -d docs/ai-context ] && cp -r docs/ai-context .claude-pre-bootstrap-backup/
ls -la .claude-pre-bootstrap-backup/
```

Report what was backed up. If the backup directory is empty, the project had no prior Claude Code config — proceed without further pre-flight concerns.

If the backup directory has content, continue with pre-flight 2-5.

### Pre-flight 2 — Naming collision check (per file)

For EACH file you plan to create, check if the destination path already exists:

```bash
# For each proposed agent
for agent in <project-slug>-orchestrator <specialist-1> <specialist-2> <...>; do
  if [ -f ".claude/agents/$agent.md" ]; then
    echo "COLLISION: .claude/agents/$agent.md already exists"
  fi
done

# For each proposed rule
for rule in <name-1> <name-2> <...>; do
  if [ -f ".claude/rules/$rule.md" ]; then
    echo "COLLISION: .claude/rules/$rule.md already exists"
  fi
done

# For each proposed skill
for skill in investigate-bug build-feature <...>; do
  if [ -d ".claude/skills/$skill" ]; then
    echo "COLLISION: .claude/skills/$skill/ already exists"
  fi
done
```

For EACH collision found, STOP and ASK:
> `<NEEDS USER CONFIRMATION: .claude/agents/<name>.md already exists. Options: (a) overwrite (existing content goes to backup), (b) skip this agent, (c) merge content. Which?>`

Do NOT silently overwrite ANY collision. If the user picks (c) merge, propose the merged content and get explicit approval before writing.

### Pre-flight 3 — `paths:` glob conflict check

For each new rule file you plan to create, check if existing rule files have an OVERLAPPING `paths:` glob:

```bash
# Read all existing rule file frontmatters
for f in .claude/rules/*.md; do
  [ -f "$f" ] && echo "=== $f ===" && awk '/^paths:/,/^---$/' "$f" | head -10
done
```

If your proposed `paths:` glob (e.g. `"src/api/**/*.ts"`) overlaps with an existing rule's glob, BOTH rule files will be referenced when editing matching files — leading to potentially contradictory rules. STOP and ASK:
> `<NEEDS USER CONFIRMATION: My proposed .claude/rules/api.md (paths: "src/api/**/*.ts") overlaps with existing .claude/rules/server.md (paths: "src/api/**,src/server/**"). Options: (a) merge into the existing file, (b) tighten my proposed glob to a non-overlapping subset, (c) accept the overlap (both will be referenced). Which?>`

### Pre-flight 4 — Drift detection on existing `CLAUDE.md`

If `CLAUDE.md` exists, READ IT and compare against current project reality:

```bash
# Re-read current tech stack from manifest
cat package.json 2>/dev/null | head -30
cat Cargo.toml 2>/dev/null | head -30
cat go.mod 2>/dev/null | head -10
cat pyproject.toml 2>/dev/null | head -30

# Compare to what existing CLAUDE.md says
cat CLAUDE.md
```

Look for stale claims in the existing file:
- Tech stack mentions that don't match current dependencies (e.g. file says "Vue 3" but package.json has React)
- Directory paths that don't exist anymore (e.g. file says `src/components/` but the project moved to `app/components/`)
- Conventions referencing removed features (e.g. mentions of `/api/v1/legacy/` routes that were deleted)
- Branch names that no longer exist (e.g. mentions `master` but the repo is `main`)

For EACH stale entry found, STOP and ASK:
> `<NEEDS USER CONFIRMATION: Existing CLAUDE.md says "<stale claim>" but current reality is "<actual>". Should I drop / update / preserve as-is?>`

Do NOT silently drop content. The existing rules may be valuable context the team added intentionally.

### Pre-flight 5 — Existing agents differ in style

If `.claude/agents/` already has files, check whether they follow a similar persona/contract style. Two parallel conventions in the same repo confuses the team.

```bash
ls .claude/agents/ 2>/dev/null
# For each existing agent, read its frontmatter + first ~30 lines of body
for f in .claude/agents/*.md; do
  [ -f "$f" ] && echo "=== $f ===" && head -40 "$f"
done
```

If existing agents exist with persona contracts that overlap with what you'd create, STOP and ASK:
> `<NEEDS USER CONFIRMATION: Existing agent .claude/agents/code-reviewer.md exists with its own persona. Should the new <project-slug>-orchestrator coordinate WITH it (treat it as another specialist), REPLACE it, or IGNORE it? If you have invested in the existing agent system, replacement risks losing team knowledge.>`

Also check whether existing agents use the new `tools:` / `maxTurns:` / `Agent(...)` allowlist patterns — if not, propose adding them as part of bootstrap (with user approval per agent).

### Decision gate — STOP if ANY pre-flight check raised an issue

If pre-flight 1-5 raised even one `<NEEDS USER CONFIRMATION>` flag, STOP and present ALL flags as a numbered list. Wait for the user to answer EVERY flag before proceeding to Step 1 below. Do not partially proceed.

If pre-flight 1-5 raised zero issues (truly greenfield Claude Code setup), proceed to Step 1.

---

## What to create (order matters)

### Step 1 — Create the directory skeleton

```
.claude/agents/
.claude/rules/
.claude/skills/
docs/ai-context/
docs/_archive/
```

### Step 2 — Create `docs/ai-context/HANDOFF_SCHEMA.md`

Use `templates/HANDOFF_SCHEMA.md.template`. Substitute `<orchestrator-name>` with `<project-slug>-orchestrator`. Customize the worked examples to use realistic file paths from THIS project's codebase (not generic placeholders). Show me the file before saving.

### Step 3 — Create `docs/_archive/README.md`

Use `templates/archive-README.md.template`. No placeholder substitution needed. Show me the file before saving.

### Step 4 — Create `docs/ai-context/INDEX.md`

Use `templates/INDEX.md.template`. Populate the routing table from the inventory's section 3 (specialists) and section 5 (orientation candidates). For each row: task type → which orientation map(s) to read → which rules might apply → which agent owns it. Show me the file before saving.

### Step 5 — Create the orchestrator agent

`.claude/agents/<project-slug>-orchestrator.md` — use `templates/orchestrator-agent.md.template`. Substitute:
- `<PROJECT_NAME>` — the project's identity from inventory section 1
- `<PROJECT_SLUG>` — the lowercase-hyphenated slug
- `<SPECIALIST_LIST>` — comma-separated list of specialist names from inventory section 3
- `<PROJECT_SPECIFIC_ANTI_PATTERNS>` — 3-5 anti-patterns specific to this project (derived from inventory)

Show me the file before saving.

### Step 6 — Create each specialist agent

For each specialist from the inventory section 3:
- If REVIEW-ONLY: use `templates/review-only-agent.md.template`
- If implementation: use `templates/specialist-agent.md.template`

Substitute placeholders. The `Incoming handoff validation` and `Return schema (required)` sections are IDENTICAL across every specialist — paste verbatim from the template. Customize the specialist-specific sections (When to use / When NOT to use / Required reading / I CAN / I CANNOT / Definition of Done / Cross-links).

Show me each file before saving. Do them in this order so I can check the pattern, then approve subsequent ones in batch:
1. First specialist: full review
2. Second specialist: full review
3. Remaining specialists: batch review (just confirm correct frontmatter + correct cross-links)

### Step 7 — Create skeleton orientation maps

For each major domain identified in inventory section 5, create `docs/ai-context/<area>.md` with a 50-150 line skeleton. Format:
- 2-sentence orientation
- "Key file paths" section (extracted from the codebase scan)
- "Top gotchas" section (start with placeholders, mark `<TODO: fill from real bugs>`)
- "Cross-links" section (link to canonical docs that already exist)

Show me each one before saving.

### Step 8 — Create `docs/ai-context/ORCHESTRATION_SPOONFEEDER.md`

Use `templates/SPOONFEEDER.md.template`. Customize the invocation modes section with the project's specific orchestrator name and specialist list.

### Step 9 — Create / update the project's `CLAUDE.md` router

This file goes at the repo root. Should be UNDER 200 lines.

**If `CLAUDE.md` doesn't exist:** create it from `templates/CLAUDE.md.template` (Step 11 names the v1.2 golden rules it carries). Include:
- 1-paragraph project identity
- Golden rules (always include "never push to protected branches without owner approval")
- Mandatory workflow (numbered steps)
- Routing tables: task touches X → read Y; task class → use agent Z
- Skills list (cross-link to `.claude/skills/`)
- Reference docs list (cross-link to `docs/<UPPERCASE>.md`)
- Branches & deploy (target deploy URLs per branch)
- Local dev commands

**If `CLAUDE.md` EXISTS** (verified during pre-flight 1):

1. The original is already snapshotted in `.claude-pre-bootstrap-backup/CLAUDE.md`.
2. **Read the existing file in full.** Identify which sections are: (a) project-specific rules the team added intentionally, (b) auto-generated `/init` content that may be stale, (c) generic best-practice content the framework's template covers anyway.
3. Produce a PROPOSED MERGED version that:
   - Preserves ALL section (a) content (project-specific team rules)
   - Drops or updates section (b) content if pre-flight 4 (drift detection) flagged it as stale
   - Replaces section (c) content with the framework's structured router
4. **Show the user a 3-pane diff** before writing:
   - LEFT: existing file (from backup)
   - RIGHT: proposed merged file
   - MIDDLE: a per-section disposition list ("preserved", "updated for drift", "replaced by template router", "dropped — covered by template")
5. Wait for explicit user approval ("yes, write the merged file") before writing. If the user pushes back on any section, revise and re-show the diff.

⚠ **Keep this file UNDER 200 lines** post-merge. CLAUDE.md is auto-loaded into every Claude Code session — it's a tight router, not a manual.

If the merged file would exceed 200 lines: propose moving sections to `.claude/rules/<NAME>.md` (path-globbed) or `docs/ai-context/<area>.md` (orientation map) instead of bloating the router. Get user approval per moved section.

Use templates conceptually but adapt to this project's actual structure.

### Step 10 — Update `.gitignore`

Append (don't overwrite) these entries to the existing `.gitignore`:

```gitignore
# Bootstrap backup — keep local-only, do not commit
.claude-pre-bootstrap-backup/

# Tool caches that should never be committed
.copilot-audit/
.playwright-mcp/
test-results/
logs/

# Env-file backups / CLI env exports — broader pattern
.env*.bak*
.env*staging_tmp
.env*tmp

# Claude Code worktrees (local-only)
.claude/worktrees/
# Personal permissions / MCP allowlists — per machine, never shared
.claude/settings.local.json
.claude/launch.json
```

The `.claude-pre-bootstrap-backup/` line is critical — that directory contains the original Claude Code config from BEFORE the bootstrap and should NEVER be committed.

If the project uses Supabase, also add:
```gitignore
supabase/.temp/
```

If the project uses Firebase:
```gitignore
.firebase/
firebase-debug.log
```

(Skip whichever doesn't apply.)

### Step 11 — Create the project-truth set (v1.2.0)

These three files are what a FRESH agent with no transcript reads first. Generate them from the
codebase with the same evidence discipline as the rules — every state claim date-stamped, every
command copied from the real build config, never guessed:

- `docs/ai-context/PROJECT.md` from `<framework path>/templates/PROJECT.md.template`. Fill §2
  (environments + **what the deploy pipeline does NOT do**), §3 (what is live where — if you cannot
  verify an environment, write `unknown`, never a guess), §6 (sources of truth: the canonical helper
  per concept, found by grep), §7 (commands, verified against the build config file).
- `docs/ai-context/LEARNINGS.md` from `<framework path>/templates/LEARNINGS.md.template`. On a
  brownfield repo, seed §A/§B from the git log and any existing post-mortems or ADRs; seed §D with
  the generic corrections already in the template; leave §E for the owner to fill.
- `docs/ai-context/GLOSSARY.md` from `<framework path>/templates/GLOSSARY.md.template` — list every
  domain concept you found called by more than one name, with the file:field evidence.
- `.claude/skills/<project-slug>-engineering/SKILL.md` from
  `<framework path>/templates/engineering-playbook-skill.md.template`.
- One `docs/<AREA>_BACKLOG.md` from `<framework path>/templates/BACKLOG.md.template` for the
  largest area, seeded with any TODO/FIXME clusters you found (each with what / why / effort /
  revisit-when).

Use `<framework path>/templates/CLAUDE.md.template` as the shape for Step 9's router — its golden
rules 9–12 (corrections → rules, deferred work written, production push freshens docs, one name per
concept) are the v1.2.0 additions.

### Step 12 — Create the rule files and starter skills the inventory proposed

- One `.claude/rules/<name>.md` per rule file in inventory section 4, from
  `<framework path>/templates/rule.md.template` — `paths:` globs verified against real files, each
  hard rule with its file:line evidence, nothing generic. (Pre-flights 2 and 3 already checked these
  names and globs for collisions.) Delete the "Notes for the rule author" section before saving.
- `.claude/skills/investigate-bug/SKILL.md` and `.claude/skills/build-feature/SKILL.md` from
  `<framework path>/templates/skill.md.template`, customised to this project's stack and roles
  (the `<project-slug>-engineering` playbook from Step 11 is the one they share).

Show me each file before saving. If the inventory proposed no rule files (or this is a greenfield
project with no inventory yet), say so and skip — never invent a rule without evidence; the runbook's
Phase 3 adds rules once there is code to cite.

## Constraints

- **Do not commit anything.** Show me each file. I'll commit at the end.
- **Do not modify existing application code.** Only create new files in `.claude/`, `docs/ai-context/`, `docs/_archive/`, `docs/<AREA>_BACKLOG.md`, and update `.gitignore` + `CLAUDE.md`.
- **Pre-flight checks are mandatory** (see top of prompt). Do not skip even on apparent greenfield. If pre-flight raises ANY `<NEEDS USER CONFIRMATION>` flag, STOP and present all flags before any file write.
- **Snapshot first, write second.** Pre-flight 1 creates `.claude-pre-bootstrap-backup/` — that backup is your safety net. If you have NOT created the backup, do not proceed to Step 1.
- **Never silently overwrite.** Any pre-existing file at a destination path you'd write requires explicit user approval (per pre-flight 2). "Merge" is not the default — ASK whether to overwrite, skip, or merge.
- **Cite evidence for every gotcha.** If you can't find evidence, mark `<NEEDS USER CONFIRMATION>` rather than guessing.
- **Use the wildcard MCP pattern** for browser-testing specialists: `mcp__chrome-devtools__*`, `mcp__playwright__*`, etc. (per Claude Code permissions docs). Do NOT enumerate individual MCP tools.
- **REVIEW-ONLY specialists must NOT have `memory:`** in frontmatter (per Pitfall 2 in `docs/08-COMMON-PITFALLS.md`).

## After all files are created — verify

Run these commands and report results:

```bash
# 1. All agents register
claude agents

# 2. Doc-link sweep — find broken refs
grep -rohE "(docs|\.claude)/[A-Za-z0-9_/-]+\.md" CLAUDE.md docs/ai-context/ .claude/agents/ .claude/rules/ .claude/skills/ 2>/dev/null | sort -u | while read p; do [ -f "$p" ] || echo "BROKEN: $p"; done

# 3. If existing CLAUDE.md was merged: diff against backup
[ -f .claude-pre-bootstrap-backup/CLAUDE.md ] && diff .claude-pre-bootstrap-backup/CLAUDE.md CLAUDE.md | head -100

# 4. If existing agents/rules/skills were preserved or merged: diff each
for f in .claude-pre-bootstrap-backup/agents/*.md 2>/dev/null; do
  [ -f "$f" ] && [ -f ".claude/agents/$(basename "$f")" ] && diff "$f" ".claude/agents/$(basename "$f")" | head -30
done

# 5. Build still passes (use whatever the project's build command is)
<project's build command>
```

Report:
- Number of project agents that registered (should match what we created)
- Number of pre-existing files preserved / merged / overwritten (per user's pre-flight decisions)
- Any broken doc links (should be zero)
- Build pass/fail
- Confirmation that `.claude-pre-bootstrap-backup/` was created and contains the original files

After verification passes:
- I'll review the diff against `.claude-pre-bootstrap-backup/`, commit, and push to a branch + open a PR (NOT direct push to develop/main).
- The backup directory should be gitignored (added in Step 10).
- Once the PR is merged and verified in real use for ~1 week, the backup directory can be deleted locally.

---

(End of prompt.)
