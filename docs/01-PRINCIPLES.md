# 01 — Principles

The seven principles this framework rests on. Every other doc, prompt, and template implements these.

## 1. Orchestrator-worker pattern, documented not coded

One coordinator agent reads tasks, classifies them, picks the right specialists, and aggregates findings. Specialists do focused work in isolated context windows.

The orchestration is **prescriptive documentation**, not runtime code. The "orchestrator" is a markdown file with instructions; the harness (Claude Code) provides isolation, tool restrictions, and turn caps. There is no orchestration service, no message queue, no central process.

This matters because:
- It works with any LLM that supports subagents
- It's fully transparent — you can read every rule
- It's reversible — delete `.claude/` and you're back to default
- Hard runtime enforcement (hooks, settings.deny) can be added later as a separate hardening pass

## 2. Specialists run in isolated context

Each specialist agent runs in its own context window. It does not see the orchestrator's full conversation. It receives a focused prompt (the handoff) and returns a focused result (the return).

This is the **single most important defense against context flooding**. A specialist investigating a database bug doesn't need to know about your unrelated UI redesign in another open thread. Isolation makes each specialist's reasoning sharp and their output auditable.

## 3. Universal evidence rule — "no evidence, no claim"

If a claim cannot be tied to a file:line, table:column, rule, doc anchor, test, or command output, it must be marked as an assumption (`confidence: low` outbound, or moved to `unverified_claims` inbound) and cannot be passed downstream as fact.

This rule appears verbatim in:
- The handoff schema (both directions)
- The orchestrator's "outbound discipline"
- Every specialist's "incoming handoff validation"

It is the **core defense against cascading hallucinations** across hops. An orchestrator that hallucinates "the bug is in `foo.ts:42`" must cite that path; the specialist re-verifies before editing; if wrong, the specialist returns `status: claim_rejected` with the evidence of what it actually found.

## 4. Failure condition — articulate what would prove you wrong

Every outbound handoff includes a `failure_condition` field: one sentence stating what observable evidence would prove the orchestrator's hypothesis or delegation premise wrong. If the specialist observes that condition, it stops, returns `claim_rejected`, and includes the triggering evidence.

This is the **inverse of `verify_before_acting`**. Where `verify_before_acting` says "check these before acting," `failure_condition` says "if you observe this, STOP." Together they bracket the work in falsifiable claims.

Forcing the orchestrator to articulate `failure_condition` is itself a thinking discipline — if you can't name what would falsify your hypothesis, you don't have a hypothesis, you have a guess.

## 5. Tools and turns are runtime-enforced; everything else is discipline

The Claude Code harness enforces:
- `tools:` allowlists (specialist physically cannot use tools not listed)
- `disallowedTools:` denylists (subtract from inheritance)
- `maxTurns:` (hard cap on agent execution)
- `Agent(name1, name2, ...)` allowlists (orchestrator can only spawn allowlisted specialists)
- `mcpServers:` per-agent MCP server scoping
- Subagent context isolation

Everything else (handoff schema fields, refusal behavior, evidence rule, hop limits, "rules read before edit") is **documentation discipline** — relies on agents following their instructions. This is acceptable because:
- Subagents are isolated, so a misbehaving agent's damage is bounded
- The orchestrator catches schema violations on receipt of a return
- A poorly-documented expectation can be tightened into a hook later if needed

Don't conflate the two layers. When you say "the orchestrator only delegates to project specialists," that's enforced at runtime via `Agent(name1, name2, ...)`. When you say "specialists refuse vague delegations," that's documented in their instructions and only as enforced as the agent's compliance.

## 6. Three-tier documentation

Project documentation should split into three clearly-labeled tiers:

| Tier | Location | Purpose | Read by |
|---|---|---|---|
| Orientation maps | `docs/ai-context/<topic>.md` | 50-150 lines per area; gotchas; cross-links to canonical | Agents (per task routing) |
| Canonical references | `docs/<UPPERCASE>.md` | Architecture, API, schema, business — full detail | Humans + agents (when deeper) |
| Frozen archive | `docs/_archive/<date>/` | Sprint reports, post-mortems, migration logs, snapshots | Audits only — never linked from active docs |

This separation makes drift impossible. Active docs live in the first two tiers. The archive is append-only and never referenced by agents/rules/active docs. When something is no longer authoritative, it moves to archive — it doesn't sit in `docs/` confusingly mixed with truth.

## 7. Default Claude Code stays default

**Do not make the orchestrator the project default.** Setting `agent: "<orchestrator-name>"` in `.claude/settings.json` would force every session into orchestrator mode, which means:
- No Edit/Write tools (every code change requires `Agent(specialist)` delegation — heavy friction for typos)
- A `maxTurns` cap restrictive for interactive iterative work
- Default Claude Code system prompt replaced entirely by the orchestrator's body prompt

The orchestrator is for **medium/complex tasks** — explicitly invoked via `claude --agent <orchestrator-name>`. Plain `claude` stays available for everyday work. Specialists can also be invoked directly via `claude --agent <specialist-name>` for narrow domain work.

Three everyday invocation modes coexist by design (plus headless `claude -p` and dynamic workflows for scripts and sweeps). See `06-INVOCATION-MODES.md`.

---

## What these principles get you

A team can run dozens of parallel sessions on the same project without context collision. Specialists never inherit baggage from unrelated work. Orchestrator decisions are auditable (every claim cites evidence). Hallucinations don't compound across hops. New engineers see a clean folder layout that tells them where to look.

The cost is upfront discipline: you spend a day writing the agent contracts, rules, and orientation maps. The payback is permanent — every future task benefits.
