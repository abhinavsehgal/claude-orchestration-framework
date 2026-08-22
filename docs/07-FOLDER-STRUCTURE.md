# 07 — Folder Structure

How to organize documentation so Claude (and humans) always know where to look.

## The three tiers

```
docs/
├── ai-context/                    ← TIER 1: orientation maps (read by agents)
│   ├── INDEX.md
│   ├── HANDOFF_SCHEMA.md
│   ├── ORCHESTRATION_SPOONFEEDER.md
│   ├── PROJECT.md                 ← (v1.2) current truth — what is live where
│   ├── LEARNINGS.md               ← (v1.2) decisions / failures / corrections
│   ├── GLOSSARY.md                ← (v1.2) one name per concept
│   └── <area>-experience.md       ← 50-150 lines per area
│
├── ARCHITECTURE.md                ← TIER 2: canonical references (full detail)
├── API.md
├── TECH_STACK.md
├── <UPPERCASE>.md
├── <AREA>_BACKLOG.md              ← (v1.2) deferred work, one per feature area
│
└── _archive/                      ← TIER 3: frozen historical
    ├── README.md                  ← archive convention
    └── <YYYY-MM>/                 ← dated reports
```

## Tier 1 — Orientation maps (`docs/ai-context/`)

**Purpose:** Per-area guides Claude reads at the start of any task in that area. Cheaper than reading thousand-line canonical docs.

**Format:** 50-150 lines per file.

**Audience:** Agents (primary), humans (secondary).

**Naming:** `<area>-experience.md` or `<area>-<concern>.md`. Examples:
- `customer-experience.md`
- `admin-experience.md`
- `auth-and-sessions.md`
- `payments.md`
- `realtime-sync.md`
- `data-pipeline.md`
- `infrastructure.md`

**Body sections:**
1. **2-sentence orientation.** What this area is, what it isn't.
2. **Key file paths.** Where to look for the code.
3. **3-5 gotchas worth knowing.** The traps that bit you in production.
4. **Cross-links.** To the canonical docs for full detail and to related rules.

**Special files:**
- `INDEX.md` — task-type → docs + agent map. The router for the orientation maps.
- `HANDOFF_SCHEMA.md` — the bidirectional handoff schema.
- `ORCHESTRATION_SPOONFEEDER.md` — the human-facing usage guide for the framework.
- `PROJECT.md`, `LEARNINGS.md`, `GLOSSARY.md` — (v1.2) the project-truth set (`11-PROJECT-TRUTH-AND-LEARNINGS.md`).
- `HOOKS.md` (optional) — what the installed hooks enforce (`10-HOOK-HARDENING.md`).
- `MIGRATION_LEDGER.md` (optional) — tracks doc/structure migrations in progress.
- `LEGACY_BACKUP.md` (optional) — frozen pre-refactor snapshot, if you migrated from a single huge CLAUDE.md.

## Tier 2 — Canonical references (`docs/<UPPERCASE>.md`)

**Purpose:** Authoritative source of truth on the topics they cover. Full detail. Updated when the underlying truth changes.

**Format:** As long as needed. No artificial line cap. A canonical doc may be 500-2000 lines.

**Audience:** Humans (primary), agents (when deeper detail is needed than the orientation map provides).

**Naming:** `UPPERCASE.md`. Convention signals "this is canonical."

**Common files:**
- `ARCHITECTURE.md` — system architecture, components, data flow
- `API.md` — endpoint reference (or link to OpenAPI spec)
- `TECH_STACK.md` — what's installed, why, version constraints
- `CHANGELOG.md` — release history
- `PRODUCT_REQUIREMENTS.md` — PRD
- `BUSINESS_DOCUMENT.md` — non-technical product/business context

**Cross-referencing:**
- Orientation maps in `ai-context/` link DOWN to canonical docs.
- Canonical docs do NOT link UP to orientation maps (orientation is just a summary; canonical is the truth).
- Agents link to either, depending on depth needed.

## Tier 3 — Frozen archive (`docs/_archive/`)

**Purpose:** Historical material that is no longer authoritative but worth preserving for audits.

**Format:** Whatever the original was — markdown, HTML, PNG, CSV.

**Audience:** Audits only. Active docs never link here.

**Naming:** Date-prefixed subdirectories: `<YYYY-MM>/<file>`. Examples:
- `_archive/2026-04/SPRINT_2_GOLIVE_2026-04-25.md`
- `_archive/2026-04/PRODUCTION_HANDOFF_2026-04-26.md`
- `_archive/2025-12/AUDIT_REPORT_2025-12-15.md`

**Sub-categories** under `_archive/`:
- `_archive/2026-04/` — dated reports
- `_archive/copilot-audit/` — old AI tool audits
- `_archive/screenshots/` — historical UI captures
- `_archive/seed-history/` — old seed data
- `_archive/html/` — generated HTML reports

**Required:** `_archive/README.md` documenting:
- What this directory is
- Why it exists (separation from active docs)
- Layout
- Three rules: don't link from active docs, don't delete, don't move back to root
- "Known link rot" section if you migrated from a structure that had cross-links to now-archived material

## What goes where — decision tree

```
You have a doc to write or move. Where?

Is it a per-area gotcha guide of 50-150 lines?
  → docs/ai-context/<area>.md

Is it the system architecture, API reference, or another full-detail doc?
  → docs/<UPPERCASE>.md

Is it a path-globbed invariant ("don't do X when editing Y")?
  → .claude/rules/<domain>.md

Is it "what is live where" / a decision / a failed approach / a correction?
  → docs/ai-context/PROJECT.md (state) or LEARNINGS.md (history) — see 11-PROJECT-TRUTH-AND-LEARNINGS.md

Is it deferred work ("we'll do that later")?
  → docs/<AREA>_BACKLOG.md — written, never just spoken

Is it a multi-step workflow that recurs?
  → .claude/skills/<workflow>/SKILL.md

Is it an agent persona definition?
  → .claude/agents/<name>.md

Is it a sprint report, audit, post-mortem, or dated snapshot?
  → docs/_archive/<YYYY-MM>/<file>.md

Is it a one-time decision rationale?
  → PR description or commit message — NOT docs/

Is it just routing ("for X tasks, read Y")?
  → CLAUDE.md or docs/ai-context/INDEX.md
```

## Folder-level conventions

### Root level

The root contains ONLY:
- Tech-stack config files (`package.json`, `tsconfig.json`, `Cargo.toml`, `go.mod`, etc.)
- Build/deploy config (`vercel.json`, `Dockerfile`, `docker-compose.yml`, etc.)
- CI config (`.github/`, `.gitlab-ci.yml`)
- Test runner config (`playwright.config.ts`, `pytest.ini`, etc.)
- The framework files (`CLAUDE.md`, `.claude/`, `docs/`)
- Application directories (`src/`, `app/`, `lib/`, `pkg/`, `cmd/`, etc.)
- Static assets (`public/`, `assets/`)

The root does NOT contain:
- Loose orphan scripts (move to `scripts/_archive/` if not actively used)
- Screenshots or images (move to `docs/_archive/screenshots/` or `assets/`)
- Backup env files (gitignore + delete from history if secrets were committed)
- Generated artifacts (gitignore)
- Stub directories with one file (consolidate elsewhere)

### Tracked-cache cleanup

These directories are commonly tracked by accident and should be gitignored:
- Tool MCP caches (`.playwright-mcp/`, `.copilot-audit/`)
- Test runner per-run state (`test-results/`, `coverage/`)
- Database CLI temp dirs (`supabase/.temp/`, `firebase/.firebase/`)
- Build artifacts (`.next/`, `dist/`, `build/`, `__pycache__/`)
- Local logs (`logs/`)

If they're already tracked: `git rm -r <dir>` and add to `.gitignore` in the same commit.

### Env file safety

Add broad gitignore patterns for env file variants:

```gitignore
.env
.env.*
!.env.example
.env*.bak*
.env*staging_tmp
.env*tmp
```

If env files with secrets are already in git history, that's a separate workstream — rotate secrets first, then scrub history with `git-filter-repo` or BFG. Removing the working-tree copy doesn't remove the history.

## Variants for non-standard projects

### Monorepo

```
monorepo-root/
├── CLAUDE.md                          ← root router; mentions per-package CLAUDE.md exists
├── .claude/
│   ├── agents/                        ← orchestrator + cross-package specialists
│   ├── rules/                         ← cross-package rules
│   └── skills/
├── docs/                              ← cross-package docs
│   └── ai-context/
└── packages/
    ├── package-a/
    │   ├── CLAUDE.md                  ← package-specific router
    │   ├── .claude/agents/            ← package-specific specialists (optional — visible only to sessions started in this package; see 02 § Monorepos)
    │   └── docs/ai-context/           ← package-specific orientation
    └── package-b/
        └── ...
```

### Multi-product in one repo

```
your-product-repo/
├── .claude/agents/
│   ├── <repo>-orchestrator.md
│   ├── product-a-frontend.md
│   ├── product-a-backend.md
│   ├── product-b-frontend.md
│   └── shared-database.md
├── docs/ai-context/
│   ├── INDEX.md                       ← lists tasks per product
│   ├── product-a-experience.md
│   └── product-b-experience.md
```

### Mobile + web shared

```
your-app/
├── apps/
│   ├── web/                           ← Next.js / Vite / etc.
│   └── mobile/                        ← React Native / Flutter / native
├── packages/                          ← shared libs
└── .claude/agents/
    ├── web-ui.md                      ← only edits apps/web/
    ├── mobile-ui.md                   ← only edits apps/mobile/
    ├── shared-libs.md                 ← only edits packages/
    └── backend-api.md                 ← stays separate if hosted elsewhere
```
