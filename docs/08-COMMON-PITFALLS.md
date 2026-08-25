# 08 — Common Pitfalls

Twenty-eight hard-won lessons from real-world adoption. Read this before bootstrapping a project.

> Pitfalls 1–17 date from v1.0/v1.1. Pitfalls 18–26 were added in v1.2.0 after three further months of production use; Pitfall 18 also corrects two v1.0 claims that the platform has since made false. Pitfalls 27–28 were added in v1.3.0 alongside Chapter 13 (standing routines).

## Pitfall 1: Making the orchestrator the project default

Setting `agent: "<orchestrator-name>"` in `.claude/settings.json` SEEMS like the right move ("now everyone gets the discipline always-on!"). It is not.

The orchestrator has:
- No Edit/Write tools (delegations required for every change)
- A modest `maxTurns` cap (insufficient for interactive iterative work)
- A body prompt designed for routing, not general work
- A system prompt that REPLACES the default Claude Code system prompt entirely

Daily work (typo fixes, "what does this function do?", quick prototyping) becomes painful.

**Right answer:** Don't set it as default. Document the invocation modes (Chapter 6) in your spoonfeeder. Train the team to use `--agent <orchestrator-name>` for production-sensitive cross-domain work and plain `claude` for everything else.

## Pitfall 2: Enabling memory on REVIEW-ONLY agents

Per the official Claude Code subagents docs, when you set `memory: project` (or `user` / `local`), the harness automatically grants Read/Write/Edit so the agent can manage its memory files.

This **silently bypasses your `tools:` allowlist** — a `legal-compliance` agent you carefully restricted to Read/Grep/Glob suddenly has Write/Edit because you turned on memory.

**Right answer:** Never enable `memory` on REVIEW-ONLY agents. For implementation specialists where memory is useful, enable it explicitly and document that the auto-Write/Edit is intentional.

## Pitfall 3: Treating documentation enforcement as runtime enforcement

The handoff schema, the universal evidence rule, the failure_condition observation, the soft hop limits — these are **documentation discipline**. They work because agents follow their instructions. They do NOT physically prevent a misbehaving agent from emitting a malformed handoff.

What IS runtime-enforced (by the Claude Code harness):
- `tools:` allowlists
- `disallowedTools:` denylists
- `maxTurns:` caps
- `Agent(name1, ...)` allowlists when orchestrator runs as main thread
- `mcpServers:` per-agent MCP scoping
- Subagent context isolation

What is NOT runtime-enforced:
- The handoff YAML block being present
- The evidence rule being followed
- Refusal of vague delegations
- Hop limits ("3rd same-specialist delegation → escalate")
- "Rules read before edit"
- Definition of Done completeness

**Right answer:** Be honest about which layer enforces what. If you need hard enforcement of a documentation rule, write a PreToolUse hook on the `Agent` tool that validates the YAML block before letting the call proceed. Don't pretend documentation is enforcement.

## Pitfall 4: `Agent(name1, name2, ...)` only binds in main-thread mode

The `Agent(specialist-1, specialist-2, ...)` allowlist on the orchestrator's `tools` field restricts which agents the orchestrator can spawn. **This restriction only takes effect when the orchestrator runs as the main thread** via `claude --agent <orchestrator-name>`.

When the orchestrator is spawned AS a subagent from another session (via the Agent tool), the `Agent(...)` allowlist on its `tools` field is **not** what governs it — the spawning session's permissions are. (v1.0 of this chapter said subagents cannot spawn subagents at all; that is no longer true — see Pitfall 18. Nesting is now allowed to a configurable depth, which makes the allowlist question *more* important, not less.)

When the main session reads CLAUDE.md and acts AS-IF the orchestrator (without `--agent`), the allowlist has no enforcement at all.

**Right answer:** Document the limitation in the orchestrator file. For hard enforcement on the main thread without `--agent`, use `permissions.deny` in `.claude/settings.json`:

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

This blocks the main session from spawning built-ins/plugins regardless of which agent it's posing as.

## Pitfall 5: MCP allowlist brittleness — list servers, not individual tools

If your specialist needs Chrome DevTools MCP and Playwright MCP, you might be tempted to list every individual tool:

```yaml
tools: Read, Edit, Bash, mcp__chrome-devtools__navigate_page, mcp__chrome-devtools__click, mcp__chrome-devtools__take_screenshot, mcp__chrome-devtools__list_console_messages, ... (28 more)
```

This is brittle. When the MCP server adds a new tool, your specialist won't get it. When tool names change, your specialist breaks silently.

**Right answer:** Use the documented wildcard syntax (per Claude Code permissions docs):

```yaml
tools: Read, Edit, Bash, mcp__chrome-devtools__*, mcp__playwright__*
```

The wildcard covers all current AND future tools from that MCP server. Other MCP servers (e.g. Supabase MCP, your custom MCP) remain blocked because they're not in the allowlist.

## Pitfall 6: Dropping `tools:` for "convenience"

It's tempting to omit `tools:` on a specialist so it inherits everything from the parent. Don't.

Missing `tools:` means the specialist has full access to:
- Every MCP server you've installed (database access, deploy access, third-party APIs)
- Every built-in tool

For specialists that legitimately need broad MCP access (e.g. `qa-functional` needs browser MCP), use the wildcard pattern from Pitfall 5. For specialists that don't need MCP, list only the standard tools.

**Right answer:** Always declare `tools:` explicitly. Defense-in-depth via least privilege.

## Pitfall 7: Ignoring `.env*` history

If env files with secrets are already in git history (you committed `.env.local.bak` once, or `.env.staging_tmp` slipped through), removing the file from the working tree doesn't remove the secrets from history.

`git log -p -- .env*` will show every secret that ever leaked. So will any clone of the repo. So will GitHub's commit history.

**Right answer:**
1. Rotate every credential that was in any leaked env file (treat them as compromised).
2. Use `git-filter-repo` or BFG Repo-Cleaner to scrub the files from history.
3. Force-push the scrubbed history (this is destructive — coordinate with the team).
4. Add broad gitignore patterns to prevent recurrence: `.env*.bak*`, `.env*staging_tmp`, `.env*tmp`.

## Pitfall 8: Reorganizing docs without a link sweep

Moving docs is mechanically easy (`git mv`). Verifying links isn't. If you move `docs/AUDIT_2025-12.md` → `docs/_archive/2025-12/AUDIT_2025-12.md` and don't check, you'll discover the broken refs weeks later when someone follows a link in CHANGELOG and gets a 404.

**Right answer:** Before any doc reorg, capture a baseline of all `docs/*.md` and `.claude/*.md` references:

```bash
grep -rohE "(docs|\.claude)/[A-Za-z0-9_/-]+\.md" CLAUDE.md docs/ .claude/ | sort -u > /tmp/refs-before.txt
```

After the move:

```bash
for path in $(cat /tmp/refs-before.txt); do
  [ -f "$path" ] || echo "BROKEN: $path"
done
```

Fix every broken ref before committing the move. Or do the reorg in phases with the sweep between each phase (this is what we did).

## Pitfall 9: Treating Claude's training data as authoritative

Claude's training data has a cutoff. Library APIs, SDK signatures, and platform conventions change. Even for things Claude "knows" — verify against current docs before writing code.

**Right answer:** When writing or refactoring code that touches a library/SDK/platform, instruct the agent to verify against current docs first. Pin doc URLs in the orientation map for that area. Add a rule: "Before writing X-SDK code, fetch the current docs at <URL>."

## Pitfall 10: Letting the orchestrator do all the work

If the orchestrator finds itself doing the actual implementation, you've drifted from the pattern. The orchestrator should:
- Read enough to delegate well
- Issue handoffs
- Aggregate returns
- Decide if more delegation is needed

The orchestrator should NOT:
- Edit code (it has no Edit/Write tool by design)
- Run long verifications itself (delegate to qa-functional)
- Make architecture decisions without consulting product-flow

**Right answer:** If the orchestrator is doing implementation, your specialists are too narrow. Either broaden a specialist's scope or add a missing one.

## Pitfall 11: Direct push to the integration branch bypassing PR review

Direct pushes feel faster. They skip:
- CI / staging deploy verification
- Vercel / Cloudflare / Netlify preview URLs
- Code review by humans
- Auditable PR history

For trivial doc changes the cost of a PR is low. For multi-file reorgs, the cost of a broken merge to the integration branch (`develop`, `main`, whatever your base branch is) is high (other engineers' worktrees, broken builds, lost time).

**Right answer:** Use direct push to the integration branch ONLY when:
- The change is < 5 files AND
- The change is doc-only OR adds purely-additive code AND
- You have explicit authorization in the current session

For everything else: push the branch, open a PR, let the preview deploy run, merge.

## Pitfall 12: Not verifying after the work is "done"

A specialist's `status: completed` is the agent's claim, not a guarantee. The orchestrator's job before merging is to verify:
- `verified_claims` covers everything in the original `claims` list
- `unverified_claims` is empty OR explicitly accepted as future-work
- `do_not_pass_downstream_without_verification` items are flagged for the next hop
- `tests_run` actually includes the relevant tests
- `files_changed` matches `expected_artifacts`

**Right answer:** Treat the return schema as a contract. If a specialist returns `status: completed` but `verified_claims` is empty, push back: "What did you actually verify?" Specialists should not be allowed to claim done without showing work.

## Pitfall 13: Skipping the `failure_condition` field

`failure_condition` feels redundant with `verify_before_acting` — until a specialist mid-task discovers the orchestrator's whole premise was wrong and doesn't know whether to push through or stop.

Without `failure_condition`, specialists tend to push through (sunk-cost on partial work). With `failure_condition`, they have an explicit STOP signal.

**Right answer:** Make `failure_condition` required in your handoff schema. Reject delegations that omit it. The discipline of articulating "what would prove me wrong" is itself valuable — if the orchestrator can't name a failure condition, the task is too vague.

## Pitfall 14: Letting `docs/` rot without a librarian

Without active maintenance, `docs/` accumulates:
- Sprint reports from 6 months ago
- Audit reports referenced by no one
- Outdated architecture decisions
- HTML/PNG artifacts
- Dated migration plans

After a year, `docs/` has 50+ files and engineers can't tell which are authoritative.

**Right answer:** Have a `context-librarian` specialist whose job is exactly this. Schedule a quarterly cleanup pass (or more often). Use the `context-refactor` skill to guide the work. Move stale material to `docs/_archive/<YYYY-MM>/`. Never delete (you might need it for an audit).

## Pitfall 15: One huge CLAUDE.md instead of a router

If your CLAUDE.md grows past 200 lines, you're using it wrong. CLAUDE.md is the router; detail goes elsewhere.

A 2000-line CLAUDE.md means:
- Every Claude Code session loads 2000 lines into context up front
- The router and the detail are mixed (you can't tell what's truth vs. what's a passing note)
- Updates require touching the same file from many directions
- You lose the ability to scope rules to file paths (everything applies everywhere)

**Right answer:** Refactor a long CLAUDE.md into the framework's tiers. Move per-area gotchas to `.claude/rules/<domain>.md` (with `paths:` globs). Move orientation to `docs/ai-context/<area>.md`. Move full detail to `docs/<UPPERCASE>.md`. Keep CLAUDE.md as the routing index — golden rules + workflow + tables + cross-links.

We did this on a real codebase. The pre-refactor CLAUDE.md was 2,928 lines. The post-refactor one is ~150. Every existing rule survived; nothing was lost. The routing pattern made it dramatically easier for both Claude and humans to navigate.

## Pitfall 16: Bootstrap on a repo with existing Claude Code config

If you ran `/init` previously, or your team has been adding to `CLAUDE.md` for months, BOOTSTRAP-PROMPT.md must NOT silently overwrite that work.

**Right answer:** the prompt now includes mandatory pre-flight checks at the top — snapshot to `.claude-pre-bootstrap-backup/`, naming-collision detection, `paths:` glob conflict detection, drift detection on existing CLAUDE.md, and a decision gate that STOPS if any pre-flight raised a `<NEEDS USER CONFIRMATION>` flag.

**Specific risks:**
- Existing `CLAUDE.md` with team rules → Pre-flight 4 (drift detection) flags stale content; Step 9 (merge step) shows a 3-pane diff before writing.
- Existing `.claude/agents/<name>.md` with same name as a proposed specialist → Pre-flight 2 (naming collision check) STOPS for explicit user decision per file.
- Existing `.claude/rules/` with overlapping `paths:` → Pre-flight 3 detects glob overlap; both rule files would be referenced creating contradictory guidance.
- Existing `.claude/agents/` with personas that overlap proposed specialists → Pre-flight 5 surfaces the parallel system; user decides migrate vs coexist.

**The pre-flight workflow is non-optional** — even on apparent greenfield projects, run it. Cost is 30 seconds; benefit is never silently destroying team work.

## Pitfall 17: Hooks installed mid-session don't activate until the next session

When you add hook entries to `.claude/settings.json` (per `docs/10-HOOK-HARDENING.md`), the running Claude Code session does **not** pick them up. `settings.json` is loaded once at session start.

This is the single most surprising property of hook deployment. We watched a session install a `PreToolUse` rule-surfacing hook, manually verify the script worked end-to-end with synthetic stdin, then edit a file matching a rule glob — and the hook didn't fire. The hooks were correctly written and wired; they just weren't loaded.

**Right answer:**
1. After installing hooks, **end the current session** (`/exit` or close the tab).
2. Start a fresh Claude Code session.
3. Verify the hook fires by intentionally triggering it (e.g. edit a file matching a rule glob and look for the `<system-reminder>` block in context).
4. Only declare the hook deployed after a fresh-session smoke test passes.

If you're shipping hooks in a PR, write the verification recipe into the PR description so reviewers don't have to re-derive it. The `templates/hooks/HOOKS.md.template` includes a "Verifying hooks are live" section that's safe to copy to your project's `docs/ai-context/HOOKS.md`.

A side effect of this: never auto-merge a PR that adds hooks. The author should restart their session and verify before approving.

## Pitfall 18: The platform moves under your conventions — re-verify the docs every quarter

Two claims this framework made in v1.0 became false within three months, and a third convention
became redundant:

- **"Subagents cannot spawn subagents."** They can — by default up to three layers below the main
  conversation (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` changes it; at the limit the `Agent` tool is
  withheld). The framework's *design* choice — specialists return `recommended_next_agent` rather
  than chaining — still stands, because a flat tree is auditable and a deep one is not. But it is a
  choice now, not a platform limit, and an orchestrator that nests must carry the handoff schema
  across every hop.
- **"Rules need a custom `applies_to:` convention plus a hook."** `.claude/rules/*.md` with a
  `paths:` frontmatter field is native. Rules without `paths:` load at launch; rules with it load
  when Claude *reads* a matching file; nested `.claude/rules/` in subdirectories load on demand;
  symlinked rule files are supported. The framework's rule template now uses `paths:`. The
  rule-surfacing hook (Pattern 1) survives for one gap only — a brand-new file that is *written*
  without ever being *read* matches no `paths:` rule — and reads the same field.
- **Dynamic workflows.** Script-driven fan-out of many subagents (`agent()` / `parallel()` /
  `pipeline()`) is a documented feature, triggered by the `ultracode` keyword. It does not replace the
  orchestrator (it is for repeatable, many-agent sweeps), but an orchestrator that needs twenty
  parallel readers should use it rather than hand-spawning.

**Right answer:** every framework claim about the platform carries a *verified-on* date. The
REFINEMENT prompt now includes a "platform drift" pass: re-read the subagents, memory, hooks and
headless pages and diff them against Chapters 3, 5, 6, 10 and 12. Treat a claim older than a quarter
as `documented-unverified` (Chapter 11) until re-checked.

## Pitfall 19: A rule file is a claim, not evidence

A rule file documented the wrong HTTP verb for a route. Three agents trusted it, shipped a client
that called the wrong verb, and every user of that flow saw a generic "check your connection"
message for a week. The fourth agent read the route's `export` line.

Rules are written by people and agents at a point in time. They rot like any other document — faster,
because they are short and confident.

**Right answer:** a rule is `documented-unverified` until you have looked at the thing it describes.
Before *relying* on a documented fact to build something, check it at the source (the route, the
schema, the config). When a rule is found wrong, fix it in the same turn — never leave a known-false
rule standing because "it's not my task".

## Pitfall 20: Deferred work that lives only in prose vanishes

"We'll do the retry logic in a follow-up" was said at the end of a session. No follow-up happened.
Two weeks later the missing retry was reported as a bug, investigated from scratch, and fixed — with
none of the context the first session had.

**Right answer:** a turn that names deferred work may not end until that work is appended to
`docs/<AREA>_BACKLOG.md` with what / why / effort / revisit-when (Chapter 11). The backlog id is
checked against the remote and open PRs first — two concurrent sessions took the same id on the same
day. Verbal follow-ups are not follow-ups.

## Pitfall 21: A production push that does not freshen the docs strands the next agent

A feature went to production on Tuesday. Its orientation map still described it as staging-only
behind a flag. On Thursday a fresh agent "enabled" it — and re-opened a migration that had already
been applied.

**Right answer:** every production push updates the full doc set *in the same turn*:
`docs/CHANGELOG.md` (the anchor), the affected orientation maps, the affected rules, and
`PROJECT.md` §3 if the production *state* changed. Documentation discipline missed this about one
push in five; it is now a Stop hook (`doc-freshness-gate`, Chapter 10 Pattern 5). A future agent sees
only what is in git.

## Pitfall 22: Correction-capture regexes false-fire on benign phrases — and the cost is trust

The correction-capture hook (Pattern 2) fired on *"You already have access to it"* — a perfectly
polite sentence — because its regex matched `you already`. It also fired on a test plan that
*quoted* a correction phrase, and on a framework doc that *explained* the hook.

**Right answer:** three guards, all now in the template. (1) Anchor the frustration verbs to what
follows them (`you already (did|changed|broke)`), never bare. (2) Strip code fences, inline code and
heredoc bodies before matching — quoted text is data, not a correction. (3) Keep the loop guard
(`stop_hook_active`) so a false fire costs one reply ("not a correction"), never a trapped session.
When a false fire does happen, tighten the pattern the same day; a hook that cries wolf gets disabled
by the team within a week.

## Pitfall 23: A killed check is inconclusive, not failed

The build-gate Stop hook capped the build at five minutes. On a machine also running two other
builds, a legitimate four-minute build was killed every time — and reported as *failed*, with a
warnings-only tail the agent then tried to "fix". An alarm loop with nothing to fix.

**Right answer:** distinguish `status === null` (killed by timeout or signal) from a non-zero exit.
A killed run has produced **no failure evidence**; exit 0 silently and let CI (which builds every PR
anyway) be the authority. Only a real non-zero exit blocks the stop. Size the cap to the slowest
honest build on a busy machine, not to the fastest one on an idle machine.

## Pitfall 24: Many sessions, one working directory

Three agent sessions shared one checkout. One was launched into a stale, detached snapshot of the
tree and "fixed" code that had already been rewritten. Two ran the build at once and corrupted
each other's output directory — every route 500'd with an error that read like a code bug and was
pure directory collision. Two picked the same migration number.

**Right answer:** `git fetch` and compare to the remote base *before the first edit* — the tree you
were launched in is not trustworthy. Do feature work in an isolated worktree branched from the
fresh-fetched base (`claude --worktree` creates one; symlink the dependency directory rather than
duplicating it). Never run a build and a dev server in the same directory at once. Before taking any
sequential id (migration number, backlog id, schema version), check the remote *and* other sessions'
open PRs.

## Pitfall 25: Never report a negative from a reader you have not verified can see the whole set

"The generator refused these 336 records" turned out to mean "the reader that listed the records was
silently capped at 1,000 rows and never showed the generator 25% of them." A confident negative
result, wrong, because the *reader* had a ceiling nobody checked.

Most data-access layers cap a bare query (ORMs, REST layers, search APIs — 1,000 is a common
default), and an explicit `limit` above the cap often does not raise it. No error, no truncation
flag.

**Right answer:** before reporting "none found", "it refused", or "zero exist", establish the reader's
ceiling. Page with a stable order, aggregate in the database, or use a count query. A capped read is
worse than a failed one: it looks like a confident answer.

## Pitfall 26: Two words for one thing ships bugs

The same concept was called `domain` in the database, `topic` on one component and `chapter` on the
page — and `topic` *already meant something else* in a legacy subsystem. A gate took the new
meaning, looked it up where the old meaning lived, matched nothing, and an empty match silently
widened a filter to the whole dataset. Every user of that gate got off-target results for a day.

Vocabulary drift is not cosmetic. It is how a correct-looking lookup returns the wrong set.

**Right answer:** one glossary (`docs/ai-context/GLOSSARY.md` — a rule file, not a nicety) naming
each concept exactly once, with the DB column, the type field and the UI label that carry it. Adding
a fifth name for an existing concept is a defect. Rename only while already editing the file that
carries the wrong name, and say so in the commit.

## Pitfall 27: A context system is a program — profile it and bisect it

The rules, router and hooks you build with this framework are injected into sessions as tokens, and
token cost compounds invisibly: a mature install can front-load thousands of lines into every
session and nobody notices until the bill or the drift does. Worse, misbehavior gets misattributed —
when the agent rabbit-holes or fixates, teams blame the model while a stale rule or an over-broad
skill is doing the steering.

**Right answer:** treat context like code, with two operational habits. **Bisect:** reproduce the
misbehavior with project context disabled (`claude --safe-mode`); if it disappears, the fault is in
your CLAUDE.md/rules/skills — find it by re-enabling halves. **Weigh:** once a quarter, measure what
the install injects per session (`/usage` breaks down where tokens go; runaway loops, extreme
parallelism and one inefficient skill are the usual culprits) and make every rule earn its tokens —
REFINEMENT check 10 makes this a standing pass. A rule that has never changed an outcome is not
free; it is paid for on every session.

## Pitfall 28: An unattended job without a verified retire path runs forever

A scheduled pipeline ran for 17 days without a single job completing. The completion write violated
a database CHECK constraint, its error return was never read, the job stayed "running", and a reaper
re-queued it — so the system burned its full capacity re-doing satisfied work, while every dashboard
showed green because runs *were happening*. Nothing a person watched distinguished "ran" from
"worked".

**Right answer:** for any unattended loop (a cron, a queue drainer, a standing routine — Chapter 13):
the completion write's error is READ, and a failed completion is a loud failure, not a silent retry;
attempt caps park a grinding job instead of letting it spin; and the reporting surface states what
was *verified done*, never just that the process exited. "The routine ran" is a claim about the
scheduler; only the checked completion write is a claim about the work.

## Pitfall 29: A scoping field the tool does not recognise fails SILENTLY — and two tools fail in opposite directions

Path-scoped instruction files carry a frontmatter field naming the globs they apply to. Get that
field wrong — a typo, a name invented before the platform shipped its own, one copied from a
different tool — and nothing errors. The file is still valid. The frontmatter still parses. What
changes is invisible, and it is **not the same failure on every tool**:

| Tool | Scoping field | Wrong or missing field | What you get |
|---|---|---|---|
| Claude Code | `paths:` | loads **unconditionally** | fails **OPEN** — every file's rules in every session |
| GitHub Copilot | `applyTo:` | **not applied automatically** | fails **CLOSED** — the guidance never arrives |

Same mistake. Opposite damage. On one you drown: a real project's fourteen rule files, ~101k
tokens, entered *every* session — a docs edit carrying the payment rules — and the cost was not
just tokens, because oversized instruction files are precisely what makes a model start ignoring
the instructions that matter. On the other you starve: the rules simply never load, the agent
works without them, and the output looks like a model that ignored your standards.

Neither prints a warning. Both look exactly like a working setup.

**Right answer:** confirm the field name against the tool's current documentation, not against
another tool's convention or your own memory (Pitfall 18). Then confirm the *globs resolve*, which
the next pitfall covers, because a correct field name pointed at a glob that matches nothing is the
same silence wearing different clothes.

## Pitfall 30: Glob metacharacters in directory names — the scoped rule that matches nothing

A framework that puts punctuation in directory names hands you globs that are silently invalid.
The characters that matter are the ones glob syntax has already claimed:

- **`[` opens a character class.** A directory literally named `[id]` written as `[id]` matches a
  single character — `i` or `d` — and never the folder.
- **`(` opens a group.** A directory named `(admin)` written as `(admin)` is read as an extglob
  group and matches nothing at all.

This is not exotic. It is the default routing convention of several mainstream web frameworks
(dynamic segments in brackets, route groups in parentheses), and any project using one will write
these paths naturally into a rule's globs and never learn they are dead.

Measured on a real repository during exactly this migration: **5 globs** broken by brackets, and
**13 more across 7 rule files** broken by parentheses. Every one looked correct in review. The
parenthesis case was not even predicted by the person doing the migration — it surfaced only
because a check ran before the commit.

**Right answer:** escape the metacharacter — `\[id\]`, `\(admin\)` — and **verify, do not
review.** These patterns cannot be validated by reading them; both failures look like ordinary
paths. Ship a checker alongside the rules (`templates/verify-rule-globs.mjs`) that asserts every
scoped file matches at least one real tracked file, and run it whenever a glob changes. A rule that
matches nothing never loads, and nothing anywhere will tell you.

Two corollaries worth keeping:

- **If a rule and a hook both read these globs, they must agree on one pattern.** A hand-written
  matcher that treats `[` as a literal and a native matcher that treats it as a class will disagree
  about the same file, so the checker compares both engines and fails on a disagreement.
- **Where the tool's own documentation does not specify its glob implementation** — as VS Code's
  does not for `applyTo` (verified 2026-08-25) — a check is not belt-and-braces, it is the only
  source of truth available.
