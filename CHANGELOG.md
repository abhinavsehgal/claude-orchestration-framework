# Changelog

All notable changes to the Claude Orchestration Framework. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.3.1] — 2026-08-25

### Fixed — the scoping traps that fail without a warning

Both come from a real migration off a custom scoping field, where the rename itself was the easy
part and the two silent failures underneath it were not.

- **Pitfall 29 — a scoping field the tool does not recognise fails silently, and the two tools
  fail in OPPOSITE directions.** Claude Code loads an unscoped rule **unconditionally** (fails
  open: ~101k tokens of rules entered every session on the reference project, a docs edit carrying
  the payment rules); Copilot does **not apply** an instruction file with no `applyTo` at all
  (fails closed: the guidance never arrives). Same mistake, opposite damage, neither warns.
- **Pitfall 30 — glob metacharacters in directory names.** `[` opens a character class and `(`
  opens a group, so a folder literally named `[id]` or `(admin)` — the routing convention of
  several mainstream web frameworks — must be escaped or the glob matches nothing. Measured on the
  reference migration: **5 globs** dead from brackets, **13 more across 7 files** from
  parentheses; the parenthesis case was not predicted and surfaced only because a check ran.

### Added

- **`templates/verify-rule-globs.mjs.template`** — asserts every path-scoped file matches at least
  one real tracked file, and fails when two matchers disagree about the same pattern. Pre-filled
  for this edition. This matters most where the tool's own docs do not specify their glob
  implementation (VS Code's do not, for `applyTo` — verified 2026-08-25): when the documentation
  cannot tell you, a check is the only source of truth.
- Chapter 5 gains **"Verify your globs — they fail silently, in both directions"**, with the
  field-vs-glob split and escaped examples.

---

## [1.3.0] — 2026-08-24

### Added — the third leg: scheduled autonomy

- **`docs/13-STANDING-ROUTINES.md`** — standing routines: narrow agent jobs on a schedule, producing
  small PRs behind review gates. The seven conventions (one charter per routine; PRs only; repro +
  truth table on every fix; one reporting surface; never self-merge; wrong output tunes the routine,
  not just the output; attempt caps + a verified retire path), model/effort tiering, a starter
  catalog (dead-code remover with the log-before-delete step, dup unifier, flaky-test root-causer,
  doc-drift checker, crash fuzzer, workspace janitor, hill-climber), and the three run mechanisms
  (product scheduler / CI cron + headless `claude -p`, which runs the cwd's hooks — verified
  2026-08-22 / chat-channel agent). Distilled from the Claude Code team's public maintenance-fleet
  practice (Aug 2026: 388 PRs opened, 180 merged, ~1-in-50 noise), restated stack- and
  domain-agnostically.
- **`templates/routine.md.template`** — the checked-in routine charter: scope, schedule + kill
  switch, output contract, reporting, gates, budgets/caps with a CHECKED completion write, noise
  budget, append-only tuning log.
- **`templates/hill-climb-skill.md.template`** — the metric-loop skill: "iterate on X with a
  measurement and a dataset until it hits Y" (baseline → one hypothesis per iteration → measure →
  keep/revert → append-only ratchet file; stop on target / plateau / budget).
- **Chapter 12 § Scheduled workspace routines** — the contract guardian on a clock
  (contract-drift-daily, service-map-freshener, workspace janitor); workspace routines stay
  read-only + reports, child writes still delegate to the child's own session.
- **Chapter 6 Mode 6** — standing routines as an invocation mode (Mode 4 on a clock, with
  governance).
- **Pitfall 27** — a context system is a program: bisect misbehavior with `--safe-mode`, weigh the
  install with `/usage`, make every rule earn its tokens.
- **Pitfall 28** — an unattended job without a verified retire path runs forever (the 17-day
  silent-grinder failure class: completion write's error never read, "ran" reported as "worked").
- **REFINEMENT checks 10–11** — context weight pass; routine health pass (noise vs budget,
  tuning-log liveness, caps verified).
- **PDF regenerated** — `Claude-Orchestration-Framework.pdf` is now the v1.3.0 render (97 pages:
  quickstart + all 14 chapters), closing the "still the v1.1.2 render" gap tracked since v1.2.0.

---

## [1.2.1] — 2026-08-22

### Fixed — same-day audit of v1.2.0 (stale text, contradictions, one runtime defect)

- **`templates/workspace/settings.json.snippet` wired a hook script the workspace never receives** (`correction-capture-prompt.mjs`) — every Stop in a workspace session would have errored. Hooks stanza removed; the snippet is the allowlist + deny block only. `bootstrap.sh` now generates it from the manifest.
- **`docs/06-INVOCATION-MODES.md` still said "subagents do not spawn other subagents"**; `templates/SPOONFEEDER.md.template` still said rules are not auto-loaded. Both corrected to the verified facts (Pitfall 18).
- **BOOTSTRAP never created the rule files or the starter skills** the docs said it creates — new Step 12 does, from `rule.md.template` / `skill.md.template`; Step 9 names `CLAUDE.md.template`; Step 10 gitignores `.claude/settings.local.json`; a greenfield-answers block stands in for the inventory on new projects. INVENTORY gains an existing-config scan (Step F2) and the `<repo>-orchestrator` naming rule.
- **`templates/HANDOFF_SCHEMA.md.template` lacked the v1.2 optional fields** chapter 4 documents and the workspace copies — added (`schema_version` unchanged).
- Counts and cross-links: "17 pitfalls" → 26, "10 chapters" → 12 + quickstart, "four hook patterns" → five, chapter 10 pattern/rule order, REFINEMENT output template sections, README tree; PDF marked as the v1.1.2 render everywhere.
- Stack-as-default wording removed (`npm run build`, `develop`, `node_modules`, `.next`, "Auth/RLS") — placeholders or one-of-several examples now; one leftover domain word in Pitfall 25.
- `lint-fix.mjs` skip directories are a `SKIP_PREFIXES` constant; `build-gate.mjs` header comments matched to v1.2 behaviour; `doc-freshness-gate.mjs` gained Rule 12 (a push in another repository is not this repo's push) and `<REPO_NAME_FRAGMENT>`.

### Added

- **`templates/workspace/bootstrap.sh.template`** — creates the whole workspace layer from a filled `workspace.json`: copies every file, fills `<WORKSPACE_*>`/`<REPO_LIST>`, generates `.gitignore` and the `settings.json` deny block from the manifest's paths, fills the service-map rows, and lists what remains to fill by hand. Quickstart Part 3 step 2 uses it; the manual `cp` block stays as the long path.
- Chapter 12: the "scanned recursively" and "subdirectory skills load on touch" claims now cite the docs that state them. README: companion-editions section; FAQ answer for Copilot points at the companion edition and the dual-tool setup.
- `claude agents` (CLI subcommand) verified locally as real; `/agents` is the in-session form.

---

## [1.2.0] — 2026-08-22

### Added — what three more months of production use taught

- **`docs/11-PROJECT-TRUTH-AND-LEARNINGS.md`** — the failure class v1.0 missed: a fresh agent knows only what is in git, and git recorded code, not truth. Specifies three knowledge stores (`PROJECT.md` with a date-stamped "what is live where" table; `LEARNINGS.md` with decisions / failed approaches / bug patterns / agent corrections; per-area backlogs), the "deferred work must be written, not spoken" rule, the "every production push freshens the docs in the same turn" rule, the six-gate engineering playbook, the evidence-confidence taxonomy (`verified-*` / `documented-unverified` / `historical` / `unknown`), the proof ladder, and the multi-client parity rules (one functional source of truth; same heading ⇒ same endpoint; one brain; grep every consumer; never hard-depend on an undeployed route).
- **`docs/12-MULTI-REPO-WORKSPACES.md`** — web + mobile + microservices in separate repos. Three layers (per-repo install → shared plugin → workspace repo), the verified table of what loads from a parent directory (child `CLAUDE.md`/rules on demand; child hooks/settings never), two delegation mechanisms (child's own `claude -p` session for writes — hooks fire; read-only cross-repo specialists for investigation), two additive handoff fields (`repo:`, `contract_impact:` / `contracts_changed:`), the cross-repo contract protocol, and a one-afternoon POC recipe. Answers "do we need a fourth framework?" — no.
- **Nine pitfalls (18–26):** platform drift (re-verify quarterly); a rule file is a claim, not evidence; deferred work in prose vanishes; production push without doc freshening; correction-regex false positives; a killed check is inconclusive; many sessions on one working directory; never report a negative from a capped reader; two words for one thing ships bugs.
- **Templates:** `CLAUDE.md.template` (the router itself — v1.0 only described it), `PROJECT.md.template`, `LEARNINGS.md.template`, `BACKLOG.md.template`, `GLOSSARY.md.template`, `engineering-playbook-skill.md.template`, `hooks/doc-freshness-gate.mjs.template` (Pattern 5), and the whole `templates/workspace/` set (router, manifest, orchestrator + two read-only specialists, contract rule, service map, contracts doc, `sync-repos.sh`, `delegate.sh`, `return-schema.json`, settings).
- **Chapter 10:** Pattern 5 (doc-freshness gate); Rules 9 (a killed check is inconclusive), 10 (strip fences / inline code / heredoc bodies before matching; match commands per top-level segment), 11 (anchor correction regexes).
- **Chapter 6:** Mode 4 headless `claude -p` (loads the directory's hooks/agents/rules unless `--bare`; `--json-schema` for structured returns) and Mode 5 dynamic workflows.
- **Chapter 4:** optional additive fields — `repo`, `contract_impact`, `contracts_changed`, `deferred_work`, and the evidence-confidence class on claims. `schema_version` stays 1.
- **Prompts:** BOOTSTRAP Step 11 generates the project-truth set; REFINEMENT gains §8 platform drift and §9 project-truth freshness.

### Changed

- **`applies_to:` → `paths:`** in the rule template and every doc. `paths:` is Claude Code's native path-scoped rule field (rules load when a matching file is read); the framework-private name is retired. `surface-matching-rules.mjs` reads `paths:` and, for un-migrated files, `applies_to:`. Pattern 1 is now documented as covering only the write-without-read gap.
- **`build-gate.mjs`:** 12-minute cap (was 5) and `timedOut` handling — a run killed by the cap exits 0 (no failure evidence) instead of blocking with a warnings-only tail.
- **`correction-capture-prompt.mjs`:** bare `you already` anchored to a past-tense verb; it had matched "you already have access".
- **`settings.json.snippet`:** Stop order is now correction-capture → build-gate → doc-freshness-gate.
- **Pitfall 4 and Chapter 3:** corrected — subagents can spawn subagents (default depth three below the main conversation). The framework's flat-tree *convention* stands as a design choice, with orchestrator → orchestrator nesting sanctioned for Chapter 12.
- **Chapter 2 Variations:** monorepo guidance now states the verified load rules and points to the official large-codebases page; new "Multiple repositories" section points to Chapter 12.
- **README:** "Where this came from" counts refreshed; every lesson was required to survive the test "any team, any stack, any domain hits this" before inclusion.

### Known gap

- `Claude-Orchestration-Framework.pdf` is still the v1.1.2 render (57 pages). Chapters 11–12 are markdown-only until the PDF is regenerated.

### Provenance

Same production codebase as v1.0/v1.1, now at 12 agents / 14 rules / 10 skills / 5 hooks and two clients of one backend. Platform facts were re-verified against the official Claude Code docs on 2026-08-22 (memory, sub-agents, large-codebases, headless pages); the verified-on date is printed wherever a fact appears, because Pitfall 18 is the lesson that made this release necessary.

---

## [1.1.2] — 2026-05-06

### Fixed (P0 — silent delivery failure)

- **`templates/hooks/correction-capture-prompt.mjs.template`** — the `<system-reminder>` reminder is now written to **stderr**, not stdout. Verified empirically that Claude Code Stop hooks surface stderr verbatim into next-turn context (as `Stop hook feedback:\n[node ...]: <stderr>` blocks) but capture-and-discard stdout. v1.1.0 and v1.1.1 wrote to stdout, so the reminder was correctly produced + exit code was correct, but the model never saw it. **Adopters of v1.1.0 or v1.1.1 must upgrade.**
- **`templates/hooks/build-gate.mjs.template`** — same fix for the build-failure reminder.
- **`templates/hooks/HOOKS.md.template`** — added "Stop-hook IO contract" section documenting the stdout/stderr split and how it differs between hook events. Recommends using stderr for Stop hooks, stdout for PreToolUse hooks.

### Added

- **`docs/10-HOOK-HARDENING.md` Rule 6 — "Stop hooks deliver via stderr, PreToolUse via stdout."** New design rule documenting the IO contract finding so future contributors don't make the same mistake. Includes the table of which channel surfaces for each hook event.

### Why this is a hotfix release

The bug was a synthetic-test blind spot: the harness verified `script returns exit 2 with the right content` (which v1.1.0 + v1.1.1 did correctly), but couldn't observe whether the content actually reached the model in real Claude Code. Only live integration testing surfaced it. The lesson is now Rule 6 in the chapter.

### Note on v1.1.1

There was no public v1.1.1 release — only an internal iteration on Glow Grades that found and fixed the v1.1.0 self-contamination + recall bugs. v1.1.2 incorporates both v1.1.1 changes AND the stderr-channel fix in a single tag.

---

## [1.1.0] — 2026-05-06

### Added — optional hook-based hardening

- **`docs/10-HOOK-HARDENING.md`** — new chapter explaining when documentation discipline isn't enough and how to add mechanical enforcement. Covers the four hook patterns by leverage, eight design rules every hook must follow, the verification recipe (and why it must run in a fresh session), and when to remove hooks again.
- **`templates/hooks/`** — drop-in generic templates for the four patterns:
  - `surface-matching-rules.mjs.template` — Pattern 1: `PreToolUse` rule-surfacing. Auto-injects matching `.claude/rules/*.md` content as a `<system-reminder>` before any `Write|Edit|MultiEdit`. Stack-agnostic.
  - `correction-capture-prompt.mjs.template` — Pattern 2: `Stop` correction-capture. Detects strong correction signals in the user's most recent message and blocks the stop until the model proposes a `.claude/rules/<file>.md` patch. Stack-agnostic.
  - `build-gate.mjs.template` — Pattern 3: `Stop` build-gate. Refuses to end a turn with build-relevant files dirty AND a failing build. Customize `BUILD_RELEVANT_RE`, `BUILD_COMMAND`, `DEPS_SENTINEL` for your stack (Next.js, Python, Go, etc.).
  - `lint-fix.mjs.template` — Pattern 4: `PostToolUse` lint-fix. Auto-runs the project's auto-fix linter on the file just touched. Always exits 0 — never blocks an edit. Customize `LINTABLE_RE` and `LINT_COMMAND_BUILDER` for your stack.
  - `settings.json.snippet` — sample wiring for all four hooks (Stop hooks ordered correctly).
  - `HOOKS.md.template` — orientation note matching the framework's tier-1 docs style.
- **`templates/slash-command.md.template`** — generic template for slash commands (e.g. `/commit-push-pr`-style daily-workflow commands). Codifies the "pre-flight inline-bash precomputes context" pattern.
- **Pitfall 17** in `docs/08-COMMON-PITFALLS.md` — `.claude/settings.json` is loaded once at session start. Hooks installed mid-session don't activate until the next session. Includes verification recipe and PR-author guidance.

### Changed

- **README** — bumped to v1.1.0; refreshed "What's in the box" tree to include chapter 10 + hook templates + slash-command template; refreshed "What this framework does NOT include" to reflect that hook templates now ship as opt-in.
- **`docs/08-COMMON-PITFALLS.md`** — header updated from "15 hard-won lessons" to "Seventeen hard-won lessons" (the file already had Pitfall 16 from a prior iteration; v1.1 adds Pitfall 17).

### Philosophy preserved

- Default install still ships **zero hooks**. Hook hardening is an explicit later-phase decision, not a setup-time default.
- Documentation discipline remains the primary enforcement layer. Hooks add a second layer for cases where the first one demonstrably fails.
- Templates are stack-agnostic at the structural level; stack-specific values are explicitly marked for customization.

### Provenance

The four hook patterns and the slash-command pattern were extracted from a production rollout on a real K-12 SaaS codebase, which itself adapted Boris Cherny's documented Claude Code setup (creator of Claude Code, Anthropic). The "deterministic scaffolding for stochastic outcomes" framing is from Boris's [Lenny's Podcast appearance](https://www.lennysnewsletter.com/p/head-of-claude-code-what-happens) (Feb 2026). The patterns generalize cleanly because they target documented Claude Code surface area (hook events, tool matchers, JSONL transcripts) rather than any single project's specifics.

---

## [1.0.0] — 2026-05-02

### Added

- Initial public release.
- Nine-chapter doc set covering principles, architecture, agents, handoff schema, rules + skills, invocation modes, folder structure, common pitfalls, runbook.
- Templates for orchestrator, specialist, REVIEW-ONLY agent, handoff schema, INDEX, spoonfeeder, rule, skill, archive README.
- Three bootstrap prompts: INVENTORY (read-only proposal), BOOTSTRAP (file generation with pre-flight safety checks), REFINEMENT (post-bootstrap hardening).
- Pre-flight safety pass for brownfield bootstrap on repos with existing Claude Code configuration.
- Battle-tested on a production K-12 codebase with 11 specialist agents, 7 path-globbed rule files, 6 repeatable workflows, and a three-tier doc organization.
