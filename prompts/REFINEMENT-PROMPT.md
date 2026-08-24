# REFINEMENT PROMPT

Use this prompt AFTER you've run 2-3 real tasks through the orchestrator (`claude --agent <project-slug>-orchestrator`) and want to harden the setup.

Adjust `<framework path>` to match your install location.

---

The orchestration framework has been live for [N] tasks. Now I want a hardening review. Read `<framework path>/docs/08-COMMON-PITFALLS.md` first.

## Review the following

### 1. Schema compliance

Inspect the most recent 3 orchestrated task transcripts (you can ask me for them or look at recent git history). Check:

- Did the orchestrator emit the outbound `handoff:` YAML block on every delegation?
- Did each specialist return the inbound `return:` YAML block?
- Were `failure_condition` items articulated meaningfully (not just "task fails")?
- Did any specialist refuse a vague delegation? If yes, what triggered it?
- Were `do_not_pass_downstream_without_verification` items honored on subsequent hops?

For each violation found, propose a fix:
- Tighten the orchestrator's outbound discipline
- Tighten a specialist's incoming validation
- Add a missing rule
- Add a missing orientation map

### 2. Tool boundary enforcement

Run `claude agents` and confirm:
- All project specialists still register
- Each agent's `tools:` field is explicit (no missing field = inheriting everything = bad)
- REVIEW-ONLY agents do NOT have Edit/Write
- REVIEW-ONLY agents do NOT have `memory:` (would silently re-enable Edit/Write)
- Browser-testing specialists use the wildcard pattern (`mcp__chrome-devtools__*`) not enumerated tools

### 3. maxTurns sizing

For each agent, check whether `maxTurns` is appropriate based on actual task patterns:
- Did any specialist hit the cap mid-task? (raise it)
- Is any specialist consistently using < 30% of its cap? (lower it slightly to enforce focus)

### 4. Specialist scope drift

For each specialist, check the last 3 tasks it handled:
- Did it routinely refuse work that "almost" fit its domain? → its scope is too narrow
- Did it routinely accept work that the description doesn't really cover? → its scope is too broad
- Did it consistently delegate aspects it could have handled inline? → consider broadening
- Did it consistently get re-delegated to (orchestrator delegating to it 2-3x in a row)? → soft hop limit triggered; investigate root cause

Propose specific edits to agent definitions, NOT broader restructuring.

### 5. Rule rot

For each `.claude/rules/*.md` file:
- Are the `paths:` globs still matching real files in the codebase?
- Are any "hard rules" describing fixed bugs (the constraint no longer exists)? → propose deletion
- Do new rules need to be added based on recent production incidents?

### 6. Doc tier hygiene

Check `docs/`:
- Are there new dated reports / sprint summaries / audit reports at `docs/` root that should move to `_archive/<YYYY-MM>/`?
- Are any orientation maps in `docs/ai-context/` over 200 lines? (Move detail to canonical doc; orientation should be 50-150 lines.)
- Is the `docs/_archive/README.md` current with any new "known link rot" entries?

### 7. Hardening recommendations (yes/no per item)

Based on what you've observed, recommend whether to add:

- **Hop limit hook** — PreToolUse hook on `Agent` that counts invocations per turn and warns past N hops. (Recommend if you've seen the same specialist delegated to 3+ times in a row in a recent task.)
- **Schema enforcement hook** — PreToolUse hook on `Agent` that validates the outbound YAML block before letting the call proceed. (Recommend if specialists have been silently emitting malformed returns.)
- **`permissions.deny` for built-ins/plugins on main thread** — `.claude/settings.json` rules blocking Explore/Plan/general-purpose/plugin agents from being spawned by the default `claude` session. (Recommend if team members keep accidentally invoking these in non-orchestrator mode when they should be in orchestrator mode.)
- **`memory: project` on implementation specialists** — for specialists where institutional knowledge accumulates (debugging recipes, schema gotchas). NEVER recommend for REVIEW-ONLY specialists. (Recommend if a specialist has handled 10+ tasks and you can identify recurring patterns it should remember.)
- **CronCreate scheduled context-refactor** — automatic quarterly cleanup pass. (Recommend if `docs/` accumulates stale material between manual cleanups.)
- **One of the five shipped hook patterns** (`docs/10-HOOK-HARDENING.md`: rule-surfacing, correction-capture, build-gate, lint-fix, doc-freshness gate) — recommend only the one that closes a rule §9 below shows was violated despite being written down.

For each recommendation, give a one-paragraph justification.

### 8. Platform drift (v1.2.0)

The platform moves under the framework's conventions (Pitfall 18). Re-read the official pages for
subagents, memory/rules, hooks and headless mode and diff them against what this project's agents,
rules and `docs/ai-context/HOOKS.md` assume. Specifically check: rules use native `paths:` (not a
private field); nothing claims subagents cannot nest; hook IO contracts still hold; any
`claude -p` delegation still loads hooks without `--bare`. Report each stale assumption with the doc
URL that contradicts it.

### 9. Project-truth freshness (v1.2.0)

- `docs/ai-context/PROJECT.md` §3: is every row's environment state still true? Diff against the
  changelog since the header's verified-on date; re-stamp the header after fixing.
- `docs/ai-context/LEARNINGS.md`: any §D correction that has been violated in the last month
  despite being written down? That is the signal to promote it to a hook (Chapter 10).
- Backlogs: any `docs/*_BACKLOG.md` item older than a quarter with no revisit signal → propose
  archive or delete. Any "later" in recent PR descriptions that never reached a backlog?
- Glossary: any new name for an existing concept introduced since the last pass?

### 10. Context weight (v1.3.0)

Measure what the install injects per session. Run `/usage` on a representative session (and on any
long-running loop) and attribute: how much goes to the router + auto-surfaced rules + skills versus
the task itself? Flag: any rule file that has not changed an outcome since the last pass (candidate
for trimming or merging), any skill/plugin that dominates the breakdown, any hook that injects a
whole file where a section would do. If misbehavior was reported this quarter (rabbit-holing,
fixation), confirm the `--safe-mode` bisection was run (Pitfall 27) before anyone blames the model.

### 11. Routine health (v1.3.0 — only if standing routines are installed)

For each routine charter (Chapter 13): noise rate vs its budget (merged vs closed PRs since the
last pass); tuning log growing whenever PRs get closed (a closed routine PR with no tuning entry
means the loop is broken); attempt caps and the checked completion write still in place; budget vs
actual spend; any routine whose reporting went silent — silence must mean broken, not idle, so
verify which it was. Propose retiring any routine still over its noise budget after two tuning
passes.

## Output format

Produce a structured report:

```
# Refinement Review — <date>

## Tasks reviewed
1. <task 1 description, outcome>
2. <task 2 description, outcome>
3. <task 3 description, outcome>

## Schema compliance
[findings + proposed fixes]

## Tool boundary enforcement
[findings + proposed fixes]

## maxTurns sizing
[findings + proposed fixes]

## Specialist scope drift
[findings + proposed fixes]

## Rule rot
[findings + proposed fixes]

## Doc tier hygiene
[findings + proposed fixes]

## Hardening recommendations
- [ ] Hop limit hook — recommended? Why?
- [ ] Schema enforcement hook — recommended? Why?
- [ ] permissions.deny — recommended? Why?
- [ ] Memory on implementation specialists — recommended for which?
- [ ] Scheduled context-refactor — recommended?
- [ ] Shipped hook pattern(s) — which, and which violated rule each closes?

## Platform drift
[each stale assumption + the doc URL that contradicts it, or "none found — re-verified <date>"]

## Project-truth freshness
[PROJECT.md §3 rows corrected + new verified-on stamp · LEARNINGS §D items to promote · stale backlog items · glossary drift]

## Context weight
[injection breakdown + rules/skills flagged]

## Routine health
[per-routine: noise vs budget, tuning-log state, caps verified — or "no routines installed"]

## Proposed PR
- Files to update (paths)
- Risk assessment per file
- Verification steps
```

## Constraints

- **Do not modify any files in this pass.** Read-only review.
- **Wait for my approval per finding.** I'll say which fixes to land vs. defer.
- **Do not propose runtime hooks unless documentation enforcement is demonstrably failing.** The framework's design is documentation-first; hooks are for proven gaps, not preemptive overengineering.

---

(End of prompt.)
