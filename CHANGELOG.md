# Changelog

All notable changes to the Claude Orchestration Framework. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.1.0] — 2026-05-06

### Added — optional hook-based hardening

- **`docs/10-HOOK-HARDENING.md`** — new chapter explaining when documentation discipline isn't enough and how to add mechanical enforcement. Covers the four hook patterns by leverage, six design rules every hook must follow, the verification recipe (and why it must run in a fresh session), and when to remove hooks again.
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
