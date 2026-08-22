# templates/workspace/ — multi-repo workspace layer (framework Chapter 12)

Copy into a NEW repo (the workspace). **Short path:** copy `workspace.json.template` to `workspace.json`, fill it, then run `bash bootstrap.sh.template <framework path>` in the new repo — it copies every file below into place, fills the per-repo placeholders, generates `.gitignore` and the `settings.json` deny block from the manifest, and lists what is left to fill by hand. **Long path:** the table below.

| File | Goes to |
|---|---|
| `bootstrap.sh.template` | (run it — creates everything below) |
| `CLAUDE.md.template` | `CLAUDE.md` |
| `workspace.json.template` | `workspace.json` |
| `gitignore.template` | `.gitignore` |
| `orchestrator-agent.md.template` | `.claude/agents/<ws>-orchestrator.md` |
| `contract-guardian-agent.md.template` | `.claude/agents/contract-guardian.md` |
| `service-mapper-agent.md.template` | `.claude/agents/service-mapper.md` |
| `cross-repo-contracts.md.template` | `.claude/rules/cross-repo-contracts.md` |
| `sync-repos.sh.template`, `delegate.sh.template`, `return-schema.json` | `.claude/scripts/` |
| `settings.json.snippet` | `.claude/settings.json` |
| `SERVICE_MAP.md.template`, `CONTRACTS.md.template` | `docs/ai-context/` |
| (from the main templates) `HANDOFF_SCHEMA.md.template` | `docs/ai-context/HANDOFF_SCHEMA.md` |

Prerequisite: every child repo already has its own layer-1 install with a working
`<repo>-orchestrator`. The workspace cannot help a repo that has nothing to delegate to.
