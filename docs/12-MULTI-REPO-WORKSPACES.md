# 12 — Multi-Repo Workspaces (web + mobile + microservices)

> Added in v1.2.0. Every platform fact in this chapter was verified against the official Claude
> Code documentation on 2026-08-22 (`code.claude.com/docs/en/memory`, `/sub-agents`,
> `/large-codebases`, `/headless`). Re-verify before relying on a behaviour — the platform moves.

## The question this chapter answers

> *"We have one repo per microservice, plus a web repo, a mobile repo, and an API gateway. Does
> orchestration need agents at every level, or can a top-level orchestrator understand all the
> services? Should we create a separate workspace repo that holds the agent files and gitignores
> the sub-projects?"*

Short answer: **three layers, each optional, each building on the one below.** The per-repo
framework you already have is layer 1 and stays mandatory. Shared specialists move into a plugin
(layer 2) the moment two repos would otherwise carry copies. A workspace repo (layer 3) exists only
when tasks *routinely* cross repos — and it holds the cross-repo orchestrator and the contract map,
**not** the specialists. The teammate's instinct (separate repo, gitignored clones, no CI) is right;
the part that needs correcting is *what lives there*.

```
Layer 3  workspace repo        ← cross-repo orchestrator, service map, contracts, sync script
Layer 2  shared plugin          ← specialists / skills / hooks that every repo would otherwise copy
Layer 1  each repo (required)   ← CLAUDE.md router, .claude/agents, .claude/rules, hooks — chapters 1-11
```

## What the platform does when you launch from a parent directory

This table is the whole design constraint. Launch `claude` in `workspace/`, with each repo cloned as
a subdirectory:

| Thing | Loaded from the workspace root? | Loaded from a child repo? |
|---|---|---|
| `CLAUDE.md` | At launch | **On demand** — when Claude reads a file inside that child |
| `.claude/rules/*.md` | At launch (unscoped) / on match (`paths:`) | On demand, same as above |
| `.claude/agents/` | Yes (scanned recursively — *Subagents* doc) | **No** — unless the child is added with `--add-dir` |
| `.claude/skills/` | Yes | Yes — once Claude has touched a file in that child (*Monorepos and large repos* doc: "skills from every subdirectory Claude touches during the session") |
| `.claude/settings.json` — hooks, permissions | **Only from the start directory** | **Never** — a child's build-gate / doc-gate / deny rules do not fire in a workspace session |
| Auto-memory | Keyed to the workspace's git repo | Not the child's |

Two consequences drive everything below:

1. **A child's rules and CLAUDE.md follow its files into the workspace session** — the discipline
   layer survives. Good.
2. **A child's enforcement layer (hooks, permission deny rules) does not.** If the child relies on
   a Stop hook to gate its build or its docs, a workspace session editing that child bypasses it.
   So any *write* to a child either runs inside the child's own session, or the workspace
   re-implements the gate. We choose the former.

Also verified, and it changes a v1.0 claim: **subagents can now spawn subagents** (default depth
three below the main conversation, `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` to change). A
two-tier orchestrator → orchestrator → specialist tree is possible in one process. This chapter
still prefers process-level delegation for writes, for reason 2 above — not because nesting is
impossible.

## Layer 1 — every repo keeps its own framework install

Nothing in this chapter removes the per-repo install. Each service, the web app and the mobile app
keep their own `CLAUDE.md`, `.claude/agents/<repo>-orchestrator.md` + specialists, `.claude/rules/`,
hooks and `docs/ai-context/`. Reasons:

- Most tasks are single-repo. A service team opening `claude` in their service must get the full
  experience with zero knowledge that a workspace exists.
- Hooks only fire from the start directory, so the gates *must* live with the code they gate.
- The child's orchestrator is what the workspace delegates to. No per-repo orchestrator, nothing to
  delegate to.

A small repo (a 3-endpoint microservice) may need only an orchestrator plus two specialists. The
framework's sizing table (`03-AGENTS-GUIDE.md`) applies per repo, not to the fleet.

## Layer 2 — shared specialists become a plugin, not copies

Twelve services will share `backend-api`, `database`, `observability`, `release-devops`,
`security-privacy` almost verbatim. Copying the files into twelve repos guarantees drift. Claude
Code's own guidance for this is a **plugin** — a versioned bundle of agents, skills, hooks and
commands, published in an internal marketplace and enabled per repo with `enabledPlugins`.

What goes in the plugin:

- The generic specialists (`backend-api`, `database`, …) with the framework's handoff validation
  and return schema blocks baked in.
- The shared skills (`investigate-bug`, `build-feature`, `engineering-playbook`, `commit-push-pr`).
- The generic hooks (rule-surfacing, correction-capture, build-gate, doc-freshness-gate) with their
  stack parameters left for the repo to set.

What stays in each repo:

- `CLAUDE.md`, `.claude/rules/` (the hard-won, repo-specific gotchas), `docs/ai-context/`,
  `PROJECT.md`, `LEARNINGS.md`, the backlogs.
- The repo's orchestrator (it names *this* repo's specialists and docs).
- Any specialist unique to that repo.

Plugin agents are namespaced (`plugin-name:agent-name`), so they never collide with a repo's own
agents — and a repo can still override by shipping a same-purpose local agent and listing only that
one in its orchestrator's `Agent(...)` allowlist.

Skip this layer while you have fewer than three repos. Copy-and-diverge is fine at that size.

## Layer 3 — the workspace repo

Create it when a task description regularly contains more than one repo name ("add the field to
the orders service *and* show it on web *and* mobile"). Until then, the per-repo orchestrators plus a
human carrying context between two terminals is cheaper than the layer.

### Layout

`templates/workspace/bootstrap.sh.template` creates all of this from a filled `workspace.json` (it fills the per-repo placeholders, generates `.gitignore` and the `settings.json` deny block from the manifest's paths, fills the service-map rows, and lists what is left for you to fill by hand). The layout it produces:

```
workspace/                              ← its own git repo
├── CLAUDE.md                           ← workspace router (templates/workspace/CLAUDE.md.template)
├── workspace.json                      ← manifest: repos, contracts, commands (templates/workspace/workspace.json.template)
├── .gitignore                          ← every child clone directory
├── .claude/
│   ├── agents/
│   │   ├── <ws>-orchestrator.md        ← cross-repo coordinator (templates/workspace/orchestrator-agent.md.template)
│   │   ├── contract-guardian.md        ← REVIEW-ONLY: API/event/schema contract changes across producers + consumers
│   │   └── service-mapper.md           ← read-only: "which repo owns X", builds/refreshes SERVICE_MAP.md
│   ├── rules/
│   │   └── cross-repo-contracts.md     ← the contract-change protocol (unscoped: loads every session)
│   ├── scripts/
│   │   ├── sync-repos.sh               ← clone/fetch every repo in the manifest
│   │   ├── delegate.sh                 ← run a child repo's orchestrator in the child's own session
│   │   └── return-schema.json          ← the return block as JSON Schema, for `claude -p --json-schema`
│   └── settings.json                   ← Bash allowlist for the two scripts + the deny block on every child path (hooks optional)
├── docs/ai-context/
│   ├── SERVICE_MAP.md                  ← repo → owns → produces → consumes → orchestrator name
│   ├── CONTRACTS.md                    ← every cross-repo contract: producer, consumers, spec location, versioning rule
│   └── HANDOFF_SCHEMA.md               ← the standard schema + the two cross-repo fields below
├── web/                                ← gitignored clone
├── mobile/                             ← gitignored clone
└── services/
    ├── orders/                         ← gitignored clone
    └── billing/                        ← gitignored clone
```

Plain clones (not submodules) in gitignored directories, driven by `workspace.json` and
`sync-repos.sh`. Submodules pin commits, which is exactly wrong for active development — you want
each child on whatever branch its own team is on. The manifest is the source of truth for *which*
repos belong; the clones are disposable.

**No CI at the workspace level.** Nothing deploys from here. The one optional job is a contract lint
(run `contract-guardian` headlessly on a schedule and open an issue on drift).

### The workspace orchestrator's contract

It is a coordinator, like every orchestrator in this framework — **no Edit/Write**. Its job:

1. Restate the cross-repo task. Read `SERVICE_MAP.md` and `CONTRACTS.md`. Identify which repos the
   task touches and which contracts sit between them.
2. **Order the work by the contract.** Producer first (additive change, deployed), then consumers.
   A consumer must never depend on a producer change that is not yet deployed — see "probe or
   degrade" below.
3. Delegate **one handoff per repo**, each to that repo's own orchestrator, using the standard
   schema plus the two cross-repo fields. Sequence them; do not fan out writes to two repos that
   share a contract in parallel.
4. Use the workspace's own read-only specialists (`contract-guardian`, `service-mapper`) for
   anything that needs to *see* several repos at once.
5. Aggregate the returns. Refuse to declare the task done while any `contract_impact: breaking`
   handoff lacks a matching consumer handoff.

### Two delegation mechanisms — pick by whether the child is written to

**Mechanism A — delegate to the child's orchestrator in the child's own session** (default for any
write). `delegate.sh <repo> <handoff.yaml>` runs:

```bash
cd "<repo path from workspace.json>" && claude -p "$(cat handoff.yaml)" \
  --agent <repo>-orchestrator \
  --permission-mode acceptEdits \
  --allowedTools "Bash(git *),Bash(<build command> *),Bash(<test command> *)" \
  --max-turns 40 \
  --output-format json --json-schema "$(cat return-schema.json)"
```

Why this and not a nested subagent: `claude -p` started *in the child* loads the child's
`CLAUDE.md`, rules, agents, skills **and runs the child's hooks** (it is documented to do so unless
`--bare` is passed — never pass `--bare` here). The child's build-gate and doc-gate fire. The child's
orchestrator spawns the child's specialists with the child's allowlists. Every layer-1 guarantee
holds. `--json-schema` forces the `return:` block into the `structured_output` field, so the
workspace orchestrator parses a contract, not prose; `session_id` in the same JSON lets it
`--resume` that child session for a follow-up instead of starting cold.

Costs: a full child session per delegation (slower than a subagent), and you must pre-approve the
tools the child needs because `-p` never prompts. Treat the allowlist as the child's permission
budget and keep it narrow.

**Mechanism B — workspace-level specialists that read across repos** (default for anything
read-only). `contract-guardian` and `service-mapper` run as ordinary subagents of the workspace
orchestrator. They have file access to every child (the children are under the start directory),
and each child's `CLAUDE.md` + rules load as the specialist reads its files. They are REVIEW-ONLY
or read-mostly by contract and by `tools:`; the `permissions.deny` block in the workspace
`settings.json` additionally denies `Edit`/`Write` under every child path, so a workspace session
*cannot* write to a child except through Mechanism A.

**Mechanism C (optional) — pull a child's agents into the session with `--add-dir`.** Documented
for added directories; it makes the child's specialists spawnable from the workspace session and, with
nesting, even a child orchestrator → child specialists tree inside one process. Two catches: every
child ships a `backend-api`, so names collide (identity comes from the `name` field only), and the
child's hooks still do not fire. Use it for fast cross-repo *investigation* when B is too coarse,
never for writes. Confirm what actually loaded with `/agents` in your first session.

### Two new handoff fields (additive — `schema_version` stays 1)

Outbound:

```yaml
  repo: <repo name from workspace.json — the ONLY repo this handoff may edit>
  contract_impact:
    level: <none | additive | breaking>
    contracts: [<names from CONTRACTS.md>]
    consumers_to_update: [<repo names — required when level != none>]
```

Inbound:

```yaml
  contracts_changed:
    - contract: <name>
      change: <one line — field added / field renamed / event payload changed …>
      backward_compatible: <true | false>
      consumers_grepped: [<repo>:<path>, …]   # evidence the consumer search happened
```

A specialist that finds itself needing to edit a second repo returns `status: blocked` with
`recommended_next_agent` naming that repo's orchestrator. It never reaches across.

### The cross-repo contract protocol (`.claude/rules/cross-repo-contracts.md`)

These are the multi-client parity rules from Chapter 11, applied across repository boundaries:

1. **Producer first, additive only, deployed before any consumer depends on it.** A breaking change
   is two additive changes plus a removal after every consumer has moved.
2. **Grep every consumer in the same turn as the contract change**, and record the search in
   `consumers_grepped`. Clients pin shapes; a renamed field fails silently (request succeeds, screen
   empty).
3. **Never ship a consumer that hard-depends on a producer change that might not be deployed.**
   Probe or degrade, and say so on screen when degraded. Deploy clocks differ per repo, and an
   installed mobile client can be older *or newer* than the server.
4. **Same heading ⇒ same endpoint.** Before building a consumer panel that mirrors another
   consumer's panel, open that consumer's code and find what it actually fetches. Record the pairing
   in `CONTRACTS.md`.
5. **One brain.** Logic lives in the producer; consumers render. A consumer that needs derived data
   asks for an endpoint, never ports the computation.
6. **One session for a whole cross-repo change.** Hand the workspace orchestrator the producer edit
   and every consumer edit together, plan first, then delegate in order. Splitting a cross-repo change
   across days re-derives decisions per repo and is where drift starts.

### `workspace.json`

```json
{
  "name": "acme",
  "repos": [
    { "name": "web",     "path": "web",             "url": "git@…/web.git",     "default_branch": "develop", "orchestrator": "web-orchestrator",     "build": "npm run build" },
    { "name": "mobile",  "path": "mobile",          "url": "git@…/mobile.git",  "default_branch": "develop", "orchestrator": "mobile-orchestrator",  "build": "npm run typecheck" },
    { "name": "orders",  "path": "services/orders", "url": "git@…/orders.git",  "default_branch": "main",    "orchestrator": "orders-orchestrator",  "build": "make build" }
  ],
  "contracts": [
    { "name": "orders-api", "producer": "orders", "consumers": ["web", "mobile"], "spec": "services/orders/openapi.yaml", "versioning": "additive-only; breaking = new major path" }
  ]
}
```

`sync-repos.sh` clones what is missing and fetches what exists; it never checks out or resets a
branch (each child's team owns that). `delegate.sh` reads `path`, `orchestrator` and `build` from
here so the orchestrator prompt never hard-codes a path.

## What NOT to do

- **Don't put the specialists in the workspace repo.** They belong with the code they edit, or in
  the plugin. A workspace that holds `backend-api` has re-created the one-agent-for-everything
  problem one level up.
- **Don't write to a child from a workspace session.** Its hooks and deny rules are not loaded.
  Delegate (Mechanism A). The workspace `settings.json` deny block exists to make this a runtime
  fact, not a documented wish.
- **Don't use git submodules for the children.** Pinned SHAs fight active development. Manifest +
  gitignored clones.
- **Don't give the workspace a deploy pipeline.** Each repo deploys itself. The workspace has
  nothing to build.
- **Don't fan out writes to two repos that share a contract in parallel.** Producer first, then
  consumers, sequentially.
- **Don't pass `--bare` to a child delegation.** It skips the child's hooks, CLAUDE.md and agents —
  the exact things you delegated to get.
- **Don't let a `-p` delegation run against a repo you don't trust.** `-p` runs that repo's hooks
  and MCP servers with no trust prompt. Delegate only to your own repos.
- **Don't assume a child's agents are visible.** They are not, unless `--add-dir`ed. Check
  `/agents`.
- **Don't carry a production approval across repos.** Approval to push *orders* to production is not
  approval to push *web*. Per repo, per incident, current turn.

## POC recipe (one afternoon)

1. Pick two repos that share one contract — a service and one of its consumers. Confirm each has
   a layer-1 install with a working `<repo>-orchestrator` (`claude --agent <repo>-orchestrator` on
   a single-repo bug). If not, do that first; the workspace cannot help a repo that has no
   orchestrator to delegate to.
2. Create the workspace repo from `templates/workspace/`. Fill `workspace.json` with the two repos
   and the one contract. Run `sync-repos.sh`.
3. Launch `claude --agent <ws>-orchestrator` in the workspace. Ask for something additive that
   crosses the contract: *"add `estimated_delivery` to the order response and show it on the web
   order page."*
4. Verify, in this order: the orchestrator read `CONTRACTS.md` and ordered producer → consumer; the
   producer delegation ran in the child (`session_id` in the JSON result, the child's hooks visible
   in its output); the consumer handoff carried `contract_impact.level: additive` and the return
   carried `consumers_grepped`; the workspace session itself wrote nothing under either child
   (`git status` in both).
5. Break it on purpose: ask for a *rename* of a field. The correct outcome is a refusal to proceed
   until the handoff is re-issued as add-then-remove with both consumers listed.
6. Only then decide on layer 2. Count how many agent files the two repos have in common; three or
   more identical files is the signal to package the plugin.

## Cross-links

- `docs/02-ARCHITECTURE.md` § Variations — the monorepo and mobile+web notes now point here.
- `docs/04-HANDOFF-SCHEMA.md` — the base schema the two cross-repo fields extend.
- `docs/11-PROJECT-TRUTH-AND-LEARNINGS.md` § Multi-client parity — the origin of the contract protocol.
- `templates/workspace/` — every file in the layout above.
- Official references (verified 2026-08-22): *Set up Claude Code in a monorepo or large codebase*,
  *Subagents → Let subagents spawn their own subagents*, *Run Claude Code programmatically*,
  *Memory → Load from additional directories*.
