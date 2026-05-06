# 10 — Hook-Based Hardening (Optional)

The framework's documentation discipline (handoff schema, evidence rule, path-globbed rules, refusal contracts) covers ~95% of the value at 5% of the complexity. **Most projects should never add hooks.**

But Pitfall 3 was honest: *"If you need hard enforcement of a documentation rule, write a PreToolUse hook on the `Agent` tool that validates the YAML block before letting the call proceed. Don't pretend documentation is enforcement."* This chapter is the runbook for *when* you need that hard enforcement, *which* hooks are worth adding, and *how* to keep them small.

This chapter is **optional**. Skip it on adoption. Come back when you have evidence that documentation discipline is being skipped under real-world pressure.

---

## When to add hooks

Add a hook only when you have **concrete evidence** that a documentation rule is being skipped — not pre-emptively, and not because hooks "feel rigorous."

Concrete evidence looks like:

| Signal | Indicates |
|---|---|
| The same correction repeats across multiple sessions ("you used the wrong helper again") | Rule exists but isn't being read |
| Auto-memory or PR review surfaces the same pattern repeatedly | Documentation discipline is failing |
| Bugs ship to production despite a rule that explicitly forbade them | Honor-system enforcement isn't working |
| New team members regularly miss the same gotchas after onboarding | The rule exists but the trigger to read it doesn't |
| `npm run build` / `pytest` / equivalents are skipped on commits ~10%+ of the time | Pre-commit human discipline is unreliable |

If none of these apply, stay on documentation discipline. The hook layer adds:
- Per-session-start startup cost (settings.json must be loaded)
- A small surface area of fail-paths (if a hook script throws, you need to fail-silent or you block all edits)
- Maintenance: scripts must be kept generic enough to survive code reorganization
- Cognitive load: contributors have to know hooks exist and what they do

That cost is justified ONLY when documentation enforcement has demonstrably failed.

---

## The four hook patterns worth knowing

These are tested patterns, ordered by leverage-per-line-of-code.

### Pattern 1 — `PreToolUse` rule-surfacing (highest leverage)

**Closes:** the gap where agents skip "before editing, read the matching `.claude/rules/*.md`." This rule appears in every framework deployment and is the most-skipped one under deadline pressure.

**Mechanism:** before any `Write|Edit|MultiEdit` tool call, a hook script reads the `applies_to:` glob frontmatter of every `.claude/rules/*.md`, matches it against the path being edited, and emits the matching rule body to stdout. Claude Code injects that stdout as a `<system-reminder>` into the model's context before the edit fires.

**Why it's the highest leverage:** 
- Already-written rules become unmissable
- Zero extra discipline required from agents
- One hook covers all rule files automatically
- Failures are silent (the edit still proceeds)

### Pattern 2 — `Stop` correction-capture (medium leverage)

**Closes:** the gap where user corrections die in conversation instead of becoming permanent rules.

**Mechanism:** when the agent tries to stop, a hook reads the transcript, scans the user's most recent message for explicit correction signals (regex on phrases like "that's wrong", "you forgot", "I told you", "stop doing", "instead of … use", "you keep", "you always"), and on match emits a `<system-reminder>` blocking the stop and asking the model to draft a one-line patch to the right `.claude/rules/<file>.md`.

**Why it's medium leverage:** highly effective at hardening repeated lessons, but requires conservative regex to avoid false positives (firing on benign messages would erode trust quickly).

### Pattern 3 — `Stop` build-gate (medium leverage, narrow scope)

**Closes:** the gap where commits land with a broken build because "run `npm run build` before commit" was procedural and skipped.

**Mechanism:** when the agent tries to stop, a hook checks `git status --porcelain` for changes to build-relevant files (TypeScript / Python / Go / etc., depending on stack). If any are dirty, it runs the build. On non-zero exit, it emits the build error tail and exits 2 (blocks stop). If the build passes or no relevant files are dirty, it exits 0.

**Why it's narrow:** the build is slow (often 30+ seconds), so the hook MUST gate on whether build-relevant files are dirty. Otherwise it punishes documentation-only turns with full-stack builds.

### Pattern 4 — `PostToolUse` lint-fix (low leverage, very low cost)

**Closes:** the "last 10% formatting drift" gap where small lint-fixable issues leak into CI and produce noise PR comments.

**Mechanism:** after every `Write|Edit|MultiEdit`, a hook runs the project's auto-fix linter (e.g. `eslint --fix`, `ruff --fix`, `gofmt`) on the file just touched. **Always exits 0** — lint problems must NEVER block an edit; build correctness is enforced separately by Pattern 3.

**Why it's worth it despite low leverage:** zero behavioural surface for the model, eliminates a class of CI noise. But also: skip if your project already runs auto-fix on save in IDEs or has a pre-commit hook outside Claude Code.

---

## Design rules for any hook

### Rule 1 — Fail silent

A hook bug must NEVER block a tool call (or worse, lock the session). Wrap your script in try/catch around `main()`, exit 0 on any unexpected error, optionally write to stderr for debugging.

```js
try { main(); } catch (e) {
  process.stderr.write(`[my-hook] ${e?.message ?? e}\n`);
  process.exit(0);
}
```

### Rule 2 — Scope narrowly via path filters

A `PreToolUse` hook on `Write|Edit|MultiEdit` fires on **every** edit, including hook-owned scripts and `.md` files. Filter the file path early — exit 0 immediately if the path doesn't match the hook's purpose.

A build-gate hook should only run the build if files matching the build's input glob are dirty. Otherwise the hook turns every documentation-only turn into a build wait.

### Rule 3 — Cap output

Hook stdout becomes part of the model's context. A 32 KB rule file dumped twice fills a meaningful slice of the context window. Cap each injection (e.g. 12 KB per rule, 40 KB total).

### Rule 4 — Loop guards

`Stop` hooks that block (exit 2) get re-fired when the agent next tries to stop. Use `payload.stop_hook_active === true` as a "this hook already fired in this stop cycle, allow stop" signal. Without a loop guard you can pin the agent in a stop loop forever.

### Rule 5 — Order matters in `Stop` arrays

If you have two `Stop` hooks (e.g. correction-capture + build-gate), order them with the lighter / more-likely-to-fire one first. The first hook to exit 2 wins; the second never runs in that cycle. Putting build-gate (slow) first means correction-capture only runs on green builds.

### Rule 6 — Settings.json is loaded once at session start

Hook entries added to `.claude/settings.json` mid-session do **not** activate until a new session starts. This is the most surprising fact about hook deployment. After installing hooks: `/exit` and re-run `claude` for them to take effect. Verify in a brand-new session, not the one that installed them.

### Rule 7 — Team-shared hooks live in `.claude/settings.json`, not `.claude/settings.local.json`

`.local.json` is per-machine (gitignored). Hooks meant to enforce team rules belong in the shared `settings.json` so every contributor and CI run gets them. Leave `.local.json` for personal MCP allowlists.

---

## Where to put the scripts

```
.claude/
  scripts/                              ← team-shared hook scripts (committed)
    surface-matching-rules.mjs
    correction-capture-prompt.mjs
    build-gate.mjs
    lint-fix.mjs
  settings.json                         ← team-shared hook wiring (committed)
  rules/                                ← unchanged
    *.md
```

Plus an orientation note at `docs/ai-context/HOOKS.md` explaining each hook's purpose, behaviour, test command, and rollback.

---

## Verification — how to know it's working

You cannot verify hooks in the session that installed them (Rule 6). The verification recipe:

1. **Direct script tests** — run each script with synthetic stdin:
   ```bash
   echo '{"tool_name":"Edit","tool_input":{"file_path":"src/foo.ts"}}' \
     | node .claude/scripts/surface-matching-rules.mjs
   ```
   If output is reasonable, the script logic works.

2. **Fresh-session integration test** — `/exit` and start a new Claude Code session. Edit a file matching a rule glob. Confirm the rule body appears as a `<system-reminder>` before the edit fires. If it doesn't appear:
   - Check `.claude/settings.json` is valid JSON (`node -e "JSON.parse(require('fs').readFileSync('.claude/settings.json'))"`)
   - Check the script runs standalone without errors
   - Check the path you're editing actually matches some rule's `applies_to`

3. **Negative test** — edit a file that should NOT trigger. Confirm no spurious output. Hooks that fire on every turn quickly become noise.

4. **Failure-mode test** — for the build-gate, deliberately introduce a TypeScript error. Confirm the agent can't end the turn until the error is fixed.

5. **Loop-guard test** — for any `Stop` hook, introduce the trigger condition, fix it, ensure the second stop attempt succeeds (not blocked again).

If any of these fail in a real-world session, treat it as P0 — broken hooks are worse than no hooks because they erode trust.

---

## When to remove hooks

Hooks have a lifecycle. Remove them when:

- The underlying problem they solved no longer exists (e.g. the recurring correction stopped happening for a quarter)
- They produce false positives more than 1× per week
- They've added > 30 seconds of latency to typical turns
- Maintaining them costs more than the bugs they prevent

Removal is one stanza-deletion in `.claude/settings.json`. Don't sentimentally keep dead hooks "just in case."

---

## Templates

The framework ships generic templates for all four hook patterns at `templates/hooks/`:

| Template | Purpose |
|---|---|
| `templates/hooks/surface-matching-rules.mjs.template` | Pattern 1 — generic rule-surfacing script |
| `templates/hooks/correction-capture-prompt.mjs.template` | Pattern 2 — generic correction-capture script |
| `templates/hooks/build-gate.mjs.template` | Pattern 3 — generic build-gate script (parameterized for stack) |
| `templates/hooks/lint-fix.mjs.template` | Pattern 4 — generic lint-fix script (parameterized for stack) |
| `templates/hooks/settings.json.snippet` | Sample wiring for all four hooks |
| `templates/hooks/HOOKS.md.template` | Orientation note for `docs/ai-context/HOOKS.md` |
| `templates/slash-command.md.template` | Generic slash-command template (e.g. `/commit-push-pr`) |

Each template has placeholders marked `<UPPERCASE_LIKE_THIS>` for stack-specific values (build script name, lint command, file globs, etc.).

---

## Cross-links

- `docs/01-PRINCIPLES.md` § Principle 5 — runtime-enforced vs documented
- `docs/05-RULES-AND-SKILLS.md` — what rule files look like (the input to Pattern 1)
- `docs/08-COMMON-PITFALLS.md` § Pitfall 3 — why documentation isn't enforcement
- `docs/08-COMMON-PITFALLS.md` § Pitfall 17 — settings.json loaded once at session start
- `templates/hooks/` — drop-in scripts and configs
