# 08 — Common Pitfalls

Hard-won lessons from real-world adoption. Read this before bootstrapping a project.

## Pitfall 1: Making the orchestrator the project default

Setting `agent: "<orchestrator-name>"` in `.claude/settings.json` SEEMS like the right move ("now everyone gets the discipline always-on!"). It is not.

The orchestrator has:
- No Edit/Write tools (delegations required for every change)
- A modest `maxTurns` cap (insufficient for interactive iterative work)
- A body prompt designed for routing, not general work
- A system prompt that REPLACES the default Claude Code system prompt entirely

Daily work (typo fixes, "what does this function do?", quick prototyping) becomes painful.

**Right answer:** Don't set it as default. Document the three invocation modes in your spoonfeeder. Train the team to use `--agent <orchestrator-name>` for production-sensitive cross-domain work and plain `claude` for everything else.

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

When the orchestrator is spawned AS a subagent from another session (via the Agent tool), it cannot spawn further subagents at all (subagents do not spawn subagents per Claude Code semantics) — so the allowlist is moot in that mode.

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

## Pitfall 11: Direct push to develop bypassing PR review

Direct pushes feel faster. They skip:
- CI / staging deploy verification
- Vercel / Cloudflare / Netlify preview URLs
- Code review by humans
- Auditable PR history

For trivial doc changes the cost of a PR is low. For multi-file reorgs, the cost of a broken merge to develop is high (other engineers' worktrees, broken builds, lost time).

**Right answer:** Use direct push to develop ONLY when:
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

**Right answer:** Refactor a long CLAUDE.md into the framework's tiers. Move per-area gotchas to `.claude/rules/<domain>.md` (with `applies_to` globs). Move orientation to `docs/ai-context/<area>.md`. Move full detail to `docs/<UPPERCASE>.md`. Keep CLAUDE.md as the routing index — golden rules + workflow + tables + cross-links.

We did this on a real codebase. The pre-refactor CLAUDE.md was 2,928 lines. The post-refactor one is ~150. Every existing rule survived; nothing was lost. The routing pattern made it dramatically easier for both Claude and humans to navigate.
