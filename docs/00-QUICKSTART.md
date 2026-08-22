# 00 — Quickstart: onboard any project (Claude Code, many repos)

> The whole framework as one step-by-step walk. Every step says **what to do**, **why**, **what to
> paste**, and **how you know it worked**. Written for someone who has never seen the framework;
> the long version of each part is linked. Based on v1.2.0 and the Claude Code docs as verified on
> 2026-08-22 — if a step here disagrees with the chapter it links to, the chapter wins.
>
> **Time:** 15 min once per person · 2–4 h per repo · one afternoon for the workspace.

---

## Part 0 · The words (read once)

Nine words. Everything else in this guide is made of these.

| Word | Means |
|---|---|
| **Orchestrator** | The manager. Reads your task, decides who should do it, writes them a note, checks their work. **Never touches code itself.** |
| **Specialist** | A worker who does one kind of job (the API, the database, the UI, testing…). Touches only its own corner. |
| **Handoff** | The note the manager writes the worker: what to do, what the manager already knows (with proof), what would prove the manager wrong, what not to touch. |
| **Return** | The note the worker writes back: what it checked, what it changed, what it couldn't verify, what the manager must not pass on as fact. |
| **Rule** | A sticky note on a drawer. When Claude reads a file in that drawer, the note appears. Lives in `.claude/rules/`, scoped with `paths:`. |
| **Skill** | A recipe card for a job you do again and again (`/commit-push-pr`, the engineering playbook). Lives in `.claude/skills/`. |
| **Hook** | A tripwire. Runs a script when something happens (before an edit, when Claude tries to stop). Optional — add later. |
| **Workspace** | One desk with every repo's folder on it, plus one manager who only coordinates between repos. Its own small git repo. |
| **Contract** | The promise between a service and everyone who calls it (an API shape, an event payload, a shared package). The thing that breaks when two repos change at different times. |

---

## Part 1 · Before you start (15 minutes, once per person)

### 1. Make sure you have the four tools

**Why:** Claude Code runs the agents; git and `jq` are used by the workspace scripts; node runs hooks.

```bash
# Each of these should print a version, not an error
claude --version       # Claude Code 2.x or later
git --version
jq --version
node --version         # hooks are node scripts
```

Missing `claude`? Install from the Claude Code docs (code.claude.com). Missing `jq`? `brew install jq`
on a Mac, `winget install jqlang.jq` on Windows.

✓ **You know it worked when:** all four print a version, and `claude` in any folder opens a session.

### 2. Get the framework onto your disk

**Why:** the framework is a folder of templates and two copy-paste prompts. You never install it
into your project — you point Claude at it.

```bash
mkdir -p ~/frameworks
git clone https://github.com/abhinavsehgal/claude-orchestration-framework.git ~/frameworks/claude
echo ~/frameworks/claude      # remember this path — you will paste it into the prompts
```

✓ `ls ~/frameworks/claude` shows `docs  prompts  templates`.

### 3. Read one file

**Why:** twenty minutes now saves a confused afternoon later. Skip the rest of the docs for now.

Open `docs/09-RUNBOOK.md`. That is the long version of Part 2 below.

✓ You can say in one sentence what the orchestrator does and does not do. (It coordinates. It never edits.)

---

## Part 2 · One repo (2–4 hours — repeat for every repo)

Every repo gets its own manager and its own workers. Yes, every one — even a tiny service. The
workspace in Part 3 only works if each repo already has this. Start with the repo you know best.

### 1. Open the repo on a fresh branch

**Why:** the setup creates new files. A clean branch means you can throw it all away if you don't like it.

```bash
cd ~/code/<your-repo>
git checkout main && git pull
git checkout -b setup/claude-orchestration
claude
```

✓ `git status` is clean; the session banner shows the repo path.

### 2. Run the INVENTORY prompt (look, don't touch)

**Why:** Claude scans the repo and *proposes* which specialists this repo needs. It writes nothing.
You correct the proposal before anything is created.

Open `~/frameworks/claude/prompts/INVENTORY-PROMPT.md`, copy all of it, paste it into the session.
Replace `<framework path>` with the path from Part 1 step 2. Send.

It comes back with: a list of proposed specialists (usually 4–8), proposed rule files with their
`paths:` globs, and a list of **Open Questions**. Answer the questions. Cross out specialists that
feel wrong. Fewer is better.

> ⚠ **The orchestrator must be named `<repo-name>-orchestrator`** (for example `orders-orchestrator`).
> Not just `orchestrator`. In Part 3 every repo's manager is delegated to by name; names must be unique.

✓ You have a short list of specialist names you agree with, and `git status` is still clean.

### 3. Run the BOOTSTRAP prompt (now it builds)

**Why:** this creates all the files. It first **backs up** anything already in `CLAUDE.md` and
`.claude/` to `.claude-pre-bootstrap-backup/` and asks before overwriting — so a `CLAUDE.md` your
team already wrote is safe (you get a 3-pane diff before the merge).

In the **same session**, paste `~/frameworks/claude/prompts/BOOTSTRAP-PROMPT.md` (path replaced
again). It shows you each file before saving. Say yes to each one you agree with.

What it creates:

| File | What it is |
|---|---|
| `CLAUDE.md` | The router. Short (< 200 lines). Loaded into every session. |
| `.claude/agents/<repo>-orchestrator.md` | The manager. |
| `.claude/agents/<name>.md` ×N | The workers. |
| `.claude/rules/<area>.md` | The sticky notes, each with a `paths:` glob. |
| `.claude/skills/<repo>-engineering/SKILL.md` | The six-gate working method every task follows. |
| `docs/ai-context/PROJECT.md` | "What is true right now" — what's live where, verified commands. |
| `docs/ai-context/LEARNINGS.md` | Decisions, failed approaches, corrections. |
| `docs/ai-context/HANDOFF_SCHEMA.md`, `INDEX.md`, `GLOSSARY.md` | The note format, the map, the one-name-per-thing list. |
| `docs/<AREA>_BACKLOG.md` | Where "we'll do it later" must be written down. |

✓ `ls .claude/agents` lists your orchestrator and specialists. Your normal build still passes.

### 4. Check the manager shows up

**Why:** an agent file with a typo in its `name:` silently doesn't load.

Exit the session (`/exit`) and start a new one: `claude`. Type `/agents`.

✓ You see `<repo>-orchestrator` and every specialist. `/context` lists `CLAUDE.md` under **Memory files**.

> ⚠ Not there? The file must be in `.claude/agents/`, the `name:` line must match what you expect,
> and the YAML between the `---` lines must be valid. Fix, restart the session, look again.

### 5. Give it one real, small job

**Why:** the only proof the setup works is watching a handoff go out and a return come back.

```bash
claude --agent <repo>-orchestrator
```

Type a real bug or tiny feature from your tracker — one that needs maybe 30 minutes of human work.

Watch for three things: it restates the task; it writes a `handoff:` block (YAML) with `claims`,
`failure_condition`, `in_scope`; the specialist answers with a `return:` block that lists
`files_changed` and `tests_run`.

✓ You saw both blocks. The change is small and correct. If the specialist *refused* a vague
handoff — that's the framework working, not failing.

### 6. Commit, open a PR, merge

**Why:** teammates get the agents the moment they pull. The backup folder stays local (gitignored).

```bash
git add CLAUDE.md .claude/ docs/ .gitignore
git commit -m "chore: bootstrap Claude Code orchestration framework"
git push -u origin setup/claude-orchestration
gh pr create --base main --title "Bootstrap Claude orchestration"
```

✓ PR merged. A teammate pulls, starts `claude`, types `/agents`, and sees the same list.

### 7. Later, not now: turn on hooks

**Why:** hooks are tripwires (block a stop while the build is red; force a correction to become a
rule; refuse to end a turn after a production push with stale docs). Add them only after you've
seen people skip the written rules. `docs/10-HOOK-HARDENING.md` is the recipe.

```bash
# when you're ready:
mkdir -p .claude/scripts
for f in ~/frameworks/claude/templates/hooks/*.mjs.template; do cp "$f" ".claude/scripts/$(basename "$f" .template)"; done
# fill the <PLACEHOLDERS> in each script, then merge templates/hooks/settings.json.snippet into .claude/settings.json
```

> ⚠ `settings.json` is read when a session starts. After installing: `/exit`, start a new `claude`,
> then test. Stop hooks talk on **stderr**; a build killed by the cap is *inconclusive*, not failed.

**Now do Part 2 again for the next repo.** Two repos done with working orchestrators is the minimum
before Part 3.

---

## Part 3 · The workspace (one afternoon, once per team)

Only when tasks keep saying "change the orders service *and* the web app". The workspace is a small
extra repo that holds one coordinating manager, a map of who owns what, and the rules for changing
contracts. It holds **no workers** — they stay in their repos. Long version:
`docs/12-MULTI-REPO-WORKSPACES.md`.

> Two Claude Code facts shape everything below (verified 2026-08-22): when you start `claude` in
> the workspace folder, a child repo's `CLAUDE.md` and rules load as Claude reads its files — but a
> child's **hooks and permission rules never load**, and a child's agents are not visible. So the
> workspace never edits a child itself; it hands each job to the child's own orchestrator in the
> child's own session, where everything the child's team built still applies.

### 1. Create an empty repo and clone it

**Why:** the workspace is its own repo so the map and the rules are versioned and shared. It has no
CI and deploys nothing.

```bash
gh repo create <team>-workspace --private --clone
cd <team>-workspace
```

✓ You're inside an empty folder with a `.git`.

### 2. Copy the workspace templates into place

**Why:** thirteen files, each with a fixed home. `templates/workspace/README.md` has the same table.

```bash
T=~/frameworks/claude/templates/workspace
mkdir -p .claude/agents .claude/rules .claude/scripts docs/ai-context
cp $T/CLAUDE.md.template                       CLAUDE.md
cp $T/workspace.json.template                  workspace.json
cp $T/gitignore.template                       .gitignore
cp $T/orchestrator-agent.md.template           .claude/agents/<team>-orchestrator.md
cp $T/contract-guardian-agent.md.template      .claude/agents/contract-guardian.md
cp $T/service-mapper-agent.md.template         .claude/agents/service-mapper.md
cp $T/cross-repo-contracts.md.template         .claude/rules/cross-repo-contracts.md
cp $T/sync-repos.sh.template                   .claude/scripts/sync-repos.sh
cp $T/delegate.sh.template                     .claude/scripts/delegate.sh
cp $T/return-schema.json                       .claude/scripts/return-schema.json
cp $T/settings.json.snippet                    .claude/settings.json
cp $T/SERVICE_MAP.md.template                  docs/ai-context/SERVICE_MAP.md
cp $T/CONTRACTS.md.template                    docs/ai-context/CONTRACTS.md
cp ~/frameworks/claude/templates/HANDOFF_SCHEMA.md.template docs/ai-context/HANDOFF_SCHEMA.md
chmod +x .claude/scripts/*.sh
```

✓ `find . -type f -not -path './.git/*'` matches the layout in chapter 12.

### 3. Fill in the blanks — mostly in one file

**Why:** `workspace.json` is the source of truth: which repos, where they live, what each one's
manager is called, every contract between them, and how much a delegated session may do. The
scripts read it; you never hard-code a path anywhere else.

Open `workspace.json`. For each repo fill `name`, `path` (e.g. `services/orders`), `url`,
`default_branch`, `orchestrator` (exactly the name from Part 2 — `orders-orchestrator`), `build`,
`test`. For each contract fill `producer`, `consumers`, `spec`, and a one-line `versioning` rule.
Under `delegation`, keep `allowed_tools` narrow — it is the child's entire permission budget.

Then search every copied file for `<` and replace the remaining placeholders (`<WORKSPACE_SLUG>`,
`<REPO_1_PATH>`…). The `deny` block in `.claude/settings.json` must list every child path — that is
what makes "the workspace never writes to a child" a fact, not a wish.

✓ `grep -rn "<[A-Z_0-9]*>" . --exclude-dir=.git` prints nothing. `jq . workspace.json` and
`jq . .claude/settings.json` both print valid JSON.

### 4. Pull every repo onto the desk

**Why:** the children are plain clones in gitignored folders — disposable. The script clones what's
missing and fetches what exists. It **never** switches or resets a branch; each repo's team owns that.

```bash
.claude/scripts/sync-repos.sh
```

✓ One `==` line per repo, no `⚠` warnings. A warning means that repo has no `<repo>-orchestrator`
yet — go back to Part 2 for it.

### 5. Start the workspace session

**Why:** the only intended way to use the workspace is *as* its orchestrator.

```bash
claude --agent <team>-orchestrator
```

✓ `/agents` shows `<team>-orchestrator`, `contract-guardian`, `service-mapper` — and **not** the
children's agents. That is correct: children are reached through `delegate.sh`, not spawned here.

### 6. Commit the workspace

```bash
git add -A
git commit -m "chore: bootstrap cross-repo workspace"
git push -u origin main
```

✓ `git status` shows no child repo files — the clones are ignored, the map and rules are committed.

### 7. The first cross-repo job — add something

**Why:** the workspace's whole job is ordering changes across a contract: the service that
*produces* a field changes first, is deployed, and only then do the apps that *read* it change.

In the workspace session, ask for something additive that crosses one contract:

```text
Add an `estimated_delivery` field to the orders API response and show it on the web order page.
```

Watch for: it reads `CONTRACTS.md`; it says "producer first: orders, then web"; it writes one
handoff per repo to `.claude/handoffs/`, each with `repo:` and `contract_impact: additive`; it runs
`.claude/scripts/delegate.sh orders <handoff>` — which starts `claude -p --agent orders-orchestrator`
*inside* the orders repo, so that repo's rules and hooks apply — and reads the `return` from the JSON
it prints.

✓ Two delegations, in that order. Each return lists `consumers_grepped`. `git status` in the
workspace root is clean — the workspace itself changed nothing. `.claude/returns/` holds the two
JSON results, each with a `session_id` you could `--resume`.

### 8. Break it on purpose — rename something

**Why:** a rename is a breaking change. The correct answer is a refusal.

```text
Rename `total` to `grand_total` in the orders API response.
```

✓ It refuses to run it as one change and proposes three: add `grand_total` → move every consumer →
remove `total`. If it just did the rename, the contract rule is not loading — check
`.claude/rules/cross-repo-contracts.md` has no `paths:` (so it loads every session) and `/context`
lists it.

---

## Part 4 · Every day (the whole thing in one table)

| You want to… | Where | Do this |
|---|---|---|
| Fix a typo, ask a question | That repo | Plain `claude`. The framework stays out of your way. |
| Do a medium/complex job inside one repo | That repo | `claude --agent <repo>-orchestrator`. Describe the job. Read the handoff and the return. |
| Work on two things at once without collisions | That repo | `claude --worktree` — an isolated checkout per session; never run two builds in one directory. |
| Ship what you did | Any | `/commit-push-pr` — it refuses to push to `main`, builds first, never stages secrets. |
| Change something in two or more repos | The workspace | `claude --agent <team>-orchestrator`. It orders the work producer → consumer and delegates one repo at a time. |
| Ask "is this API change safe?" | The workspace | Ask the orchestrator to run `contract-guardian`. It greps every consumer and answers with file:line, never an opinion. |
| Correct Claude ("no, use X") | Any | With hooks: it is stopped and asked to draft a rule patch. Without: say "make that a rule". Never accept "I'll remember" — memory is machine-local. |
| Say "we'll do that later" | Any | It must be written into `docs/<AREA>_BACKLOG.md` before the turn ends. Spoken follow-ups vanish. |
| Push to production | That repo | Same turn: update `CHANGELOG.md`, the affected `docs/ai-context/` map, and `PROJECT.md` "what is live where". With hooks, the doc-freshness gate refuses to end the turn until you do. |
| Run a job from a script or CI | That repo | `claude -p "<task>" --agent <repo>-orchestrator --output-format json` (no `--bare` if you want the repo's hooks and rules). |

---

## Part 5 · When it goes wrong

| You see | It usually means | Fix |
|---|---|---|
| The agent isn't in `/agents` | Wrong folder or frontmatter | Must be in `.claude/agents/`; `name:` must match; valid YAML between the `---` lines. Restart the session. |
| A rule never shows up | Its `paths:` glob doesn't match, or the file was never *read* | Run `find . -path "<the glob>"`. Rules load when Claude reads a matching file; a brand-new file written blind matches nothing — that is what the optional rule-surfacing hook covers. |
| From the workspace, the children's agents are missing | By design | Children are reached through `delegate.sh`. To spawn a child's agents in-session (investigation only), start with `claude --add-dir <child>`. |
| `delegate.sh` exits with "handoff does not carry repo:" | The handoff file is missing its `repo:` line | Add `repo: <name>` exactly as in `workspace.json`. A note for one repo must never run in another. |
| `delegate.sh` hangs or asks nothing and fails | `-p` never prompts; a tool wasn't pre-approved | Widen `delegation.allowed_tools` in `workspace.json` *one tool at a time*; check the child's result JSON under `.claude/returns/`. |
| The workspace session tried to edit a child and was denied | The deny block did its job | Correct behaviour. Delegate instead. |
| A hook never fires | Installed mid-session, or wrote to the wrong channel | `/exit` and start a new session. Stop hooks must write to **stderr** and exit 2; PreToolUse hooks write to stdout. |
| The build-gate blocks on a "failed" build that was just slow | The cap is too low | Raise `BUILD_TIMEOUT_MS`; a killed run must exit 0 (the v1.2 template does). |
| The specialist refused the task | The handoff was vague | That's correct behaviour. Add the missing field it named (usually `failure_condition` or evidence on a claim) and re-send. |
| Something worked last month and now doesn't | The platform moved | Re-run `prompts/REFINEMENT-PROMPT.md`; its "platform drift" section re-checks the Claude Code docs. Two framework claims went stale in three months once already (Pitfall 18). |

---

## The checklist (print this)

**Per person, once**
- [ ] `claude`, `git`, `jq`, `node` all print a version
- [ ] Framework cloned to `~/frameworks/claude`

**Per repo**
- [ ] Branch `setup/claude-orchestration`
- [ ] INVENTORY prompt → specialist list agreed
- [ ] BOOTSTRAP prompt → files created; orchestrator named `<repo>-orchestrator`
- [ ] New session → `/agents` lists them; `/context` lists `CLAUDE.md`
- [ ] One real small job via `claude --agent <repo>-orchestrator` → saw `handoff:` and `return:`
- [ ] PR merged; a teammate sees the agents too
- [ ] (later) hooks installed in `.claude/settings.json`, session restarted, tested

**Per team, once — after ≥ 2 repos are done**
- [ ] Workspace repo created; 13 template files in place; no `<PLACEHOLDER>` left
- [ ] `workspace.json`: every repo + every contract filled; `allowed_tools` narrow
- [ ] `.claude/settings.json` deny block lists every child path
- [ ] `.claude/scripts/sync-repos.sh` → no warnings
- [ ] `claude --agent <team>-orchestrator` → `/agents` shows the three workspace agents only
- [ ] Additive cross-repo job → producer first, two delegations, consumers grepped, workspace tree clean
- [ ] Rename job → refused and split into add / move / remove

---

## Cross-links

- https://abhinavsehgal.github.io/claude-orchestration-framework/ (GitHub Pages) or `docs/00-QUICKSTART.html` offline — this guide plus the other two editions as one page with tabs (open in a browser; generated from the three `00-QUICKSTART.md` files on 2026-08-22).

- `docs/09-RUNBOOK.md` — the long version of Part 2.
- `docs/10-HOOK-HARDENING.md` — hooks (Part 2 step 7).
- `docs/12-MULTI-REPO-WORKSPACES.md` — the long version of Part 3, with the verified platform behaviour behind every step.
- `docs/11-PROJECT-TRUTH-AND-LEARNINGS.md` — why the backlog, `PROJECT.md` and "push = freshen docs" rules exist.
- `docs/06-INVOCATION-MODES.md` — `claude` vs `--agent` vs `-p` vs worktrees.
- `docs/08-COMMON-PITFALLS.md` — Part 5, in full.
