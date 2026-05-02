# BOOTSTRAP PROMPT

Paste this AFTER you have completed the INVENTORY-PROMPT pass and approved the proposed specialists. Adjust `<framework path>` to match your install location.

---

I've reviewed the inventory. Now bootstrap the Claude Orchestration Framework on this project. Use the templates from `<framework path>` (default: `/Users/<you>/Desktop/claude-orchestration-framework/templates/`). Read those template files before generating anything.

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

### Step 9 — Create the project's `CLAUDE.md` router

This file goes at the repo root. Should be UNDER 200 lines. Include:
- 1-paragraph project identity
- Golden rules (always include "never push to protected branches without owner approval")
- Mandatory workflow (numbered steps)
- Routing tables: task touches X → read Y; task class → use agent Z
- Skills list (cross-link to `.claude/skills/`)
- Reference docs list (cross-link to `docs/<UPPERCASE>.md`)
- Branches & deploy (target deploy URLs per branch)
- Local dev commands

Use templates conceptually but adapt to this project's actual structure. Show me before saving.

### Step 10 — Update `.gitignore`

Append (don't overwrite) these entries to the existing `.gitignore`:

```gitignore
# Tool caches that should never be committed
.copilot-audit/
.playwright-mcp/
test-results/
logs/

# Vercel/CLI env exports — broader pattern
.env*.bak*
.env*staging_tmp
.env*tmp

# Claude Code worktrees (local-only)
.claude/worktrees/
.claude/launch.json
```

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

## Constraints

- **Do not commit anything.** Show me each file. I'll commit at the end.
- **Do not modify existing application code.** Only create new files in `.claude/`, `docs/ai-context/`, `docs/_archive/`, and update `.gitignore` + `CLAUDE.md`.
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

# 3. Build still passes (use whatever the project's build command is)
<project's build command>
```

Report:
- Number of project agents that registered (should match what we created)
- Any broken doc links (should be zero)
- Build pass/fail

After verification passes: I'll review the diff, commit, and push to a branch + open a PR (NOT direct push to develop/main).

---

(End of prompt.)
