 # INVENTORY PROMPT

Paste this into Claude Code at the root of the project you want to bootstrap. Adjust `<framework path>` to match your install location.

---

I want to adopt the Claude Orchestration Framework on this project. Before we generate any files, do a read-only inventory.

The framework lives at `<framework path>` (default: `/Users/<you>/Desktop/claude-orchestration-framework/`). Read `docs/01-PRINCIPLES.md`, `docs/02-ARCHITECTURE.md`, and `docs/03-AGENTS-GUIDE.md` from there before answering. Do not modify any files in this project yet.

## ⚠ Two universal rules for this entire pass

1. **Evidence-first.** Every claim you make must cite a real file path, command output, or filename. The phrase "I infer" or "appears to be" without a specific file:line citation is not acceptable.
2. **Ask, don't guess.** If a fact cannot be confirmed by reading a specific file or running a specific command, mark it `<NEEDS USER CONFIRMATION: <one-line question for me>>` and EXPLAIN what you tried before giving up. Examples:
   - `<NEEDS USER CONFIRMATION: Is this project named "acme" or "acme-store"? package.json says "@acme/store-monorepo" — slug is ambiguous.>`
   - `<NEEDS USER CONFIRMATION: Are EU users in scope? No GDPR-related env vars or consent UI found, but README mentions "international expansion".>`

Surface ambiguity. Never invent a fact to fill a gap.

---

## Discovery commands — run these in order

Before writing anything, execute this discovery sequence and capture the results in your context. If a command/file is not present, note it as missing and move on.

### Step A — Top-level inventory

```bash
pwd
ls -la
find . -maxdepth 1 -type f | sort
find . -maxdepth 2 -type d | sort
git remote -v
git branch -a
git log --oneline -5
```

### Step B — Project name + tech stack manifest discovery

Read whichever of these exist (in this order); stop after the first one found that has a clear `name` field:

| File | Looks for |
|---|---|
| `package.json` | `name`, `description`, `dependencies`, `scripts` (Node/JS/TS) |
| `pyproject.toml` | `[project] name`, `dependencies` (Python) |
| `setup.py` | `name=`, `install_requires=` (legacy Python) |
| `Cargo.toml` | `[package] name`, `[dependencies]` (Rust) |
| `go.mod` | `module <path>`, `require` (Go) |
| `Gemfile` + `*.gemspec` | gem name, deps (Ruby) |
| `composer.json` | `name`, `require` (PHP) |
| `pubspec.yaml` | `name:`, `dependencies:` (Flutter/Dart) |
| `*.podspec` / `Podfile` | iOS native dependencies |
| `build.gradle` / `build.gradle.kts` | Android / Kotlin |
| `*.csproj` | .NET project name |
| `package.swift` | Swift Package Manager |
| `mix.exs` | Elixir |

If MULTIPLE manifests exist (monorepo), list all of them and ask:
`<NEEDS USER CONFIRMATION: Which manifest is the canonical project name source?>`

If NONE exist:
`<NEEDS USER CONFIRMATION: No standard manifest found. Should I use the directory name "<dirname>" as the project name?>`

### Step C — Framework + service detection

Read these files if present (don't skip — each tells Claude something specific):

| File | Tells you |
|---|---|
| `next.config.js` / `next.config.ts` | Next.js (web framework) |
| `nuxt.config.ts` | Nuxt (Vue web framework) |
| `vite.config.ts` / `vite.config.js` | Vite (build tool — usually Vue/React/Svelte) |
| `svelte.config.js` | SvelteKit |
| `astro.config.mjs` | Astro |
| `remix.config.js` | Remix |
| `gatsby-config.js` | Gatsby |
| `angular.json` | Angular |
| `vue.config.js` | Vue CLI |
| `app.json` / `metro.config.js` | React Native / Expo |
| `pubspec.yaml` (lib: flutter) | Flutter |
| `tsconfig.json` | TypeScript target/strict mode |
| `prisma/schema.prisma` | Prisma ORM (database) |
| `drizzle.config.ts` | Drizzle ORM (database) |
| `knexfile.js` | Knex (DB query builder) |
| `*.sql` files in `migrations/` or `db/` | Raw SQL migrations |
| `Dockerfile` / `docker-compose.yml` | Container setup |
| `vercel.json` | Vercel deploy |
| `netlify.toml` | Netlify deploy |
| `wrangler.toml` | Cloudflare Workers |
| `serverless.yml` | Serverless framework |
| `fly.toml` | Fly.io deploy |
| `app.yaml` | Google App Engine |
| `Procfile` | Heroku / similar PaaS |
| `firebase.json` | Firebase |
| `amplify.yml` / `amplify/` | AWS Amplify |
| `terraform/*.tf` / `cdk.json` | IaC (Terraform / AWS CDK) |
| `.github/workflows/*.yml` | GitHub Actions CI/CD |
| `.gitlab-ci.yml` | GitLab CI |
| `.circleci/config.yml` | CircleCI |
| `playwright.config.ts` | Playwright E2E tests |
| `cypress.config.ts` | Cypress E2E tests |
| `vitest.config.ts` / `jest.config.js` | Unit test runner |
| `pytest.ini` / `tox.ini` | Python test config |

### Step D — External service detection (.env / dependencies)

Read `.env.example` (or `.env.sample` / `.env.template` — whichever exists). For each environment variable, infer the service:

| Env var pattern | External service |
|---|---|
| `STRIPE_*`, `STRIPE_SECRET_KEY` | Stripe (payments) |
| `RESEND_*`, `SENDGRID_*`, `MAILGUN_*`, `POSTMARK_*` | Transactional email |
| `TWILIO_*` | Twilio (SMS / voice) |
| `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_*` | LLM provider |
| `SUPABASE_*`, `NEXT_PUBLIC_SUPABASE_*` | Supabase (backend) |
| `FIREBASE_*` | Firebase |
| `CLERK_*`, `NEXTAUTH_*`, `AUTH0_*`, `WORKOS_*` | Auth provider |
| `POSTHOG_*`, `MIXPANEL_*`, `SEGMENT_*`, `AMPLITUDE_*` | Analytics |
| `SENTRY_*` | Error monitoring |
| `DATADOG_*`, `NEW_RELIC_*` | APM / observability |
| `CLOUDFLARE_*`, `R2_*`, `S3_*`, `GCS_*` | Storage / CDN |
| `ALGOLIA_*`, `MEILISEARCH_*`, `ELASTICSEARCH_*` | Search |
| `REDIS_URL`, `UPSTASH_*` | Cache / queue |
| `DAILY_*`, `LIVEKIT_*`, `AGORA_*`, `PUSHER_*`, `ABLY_*` | Real-time / video |
| `STRIPE_*` (multiple) | Subscription billing or marketplace |

Cross-check by reading the `dependencies` section of the manifest discovered in Step B. The two should agree.

### Step E — Source-tree shape

```bash
find . -maxdepth 4 -type d \
  -not -path './node_modules/*' \
  -not -path './.git/*' \
  -not -path './.next/*' \
  -not -path './dist/*' \
  -not -path './build/*' \
  -not -path './target/*' \
  -not -path './venv/*' \
  -not -path './__pycache__/*' \
  | sort | head -100
```

Then for the apparent source root (`src/`, `app/`, `lib/`, `internal/`, `cmd/`, etc.):
```bash
find <source-root> -maxdepth 3 -type d | sort
ls <source-root>/<api-or-server-dir> 2>/dev/null | head
ls <source-root>/<components-or-ui-dir> 2>/dev/null | head
```

### Step F — Documentation discovery

```bash
find docs -type f -name "*.md" 2>/dev/null | sort
ls docs/ai-context/ 2>/dev/null
find . -maxdepth 1 -type f -name "*.md" | sort   # README, CONTRIBUTING, etc.
```

For each doc found, read its first 20 lines to categorize (canonical / orientation / archive / workflow).

### Step G — Hygiene scan

```bash
ls .copilot-audit .playwright-mcp test-results logs supabase/.temp 2>/dev/null
find . -maxdepth 1 -type f -name "_*" -o -name "check-*" -o -name "test-*" 2>/dev/null
find . -maxdepth 1 -type f \( -name "*.png" -o -name "*.jpg" -o -name "*.html" \) 2>/dev/null
find . -maxdepth 2 -type f -name ".env*" 2>/dev/null
git ls-files | grep -E "(\\.env|secret|credential)" | head
```

### Step H — Branch and remote check

```bash
git branch -a | head -20
git remote -v
git log --all --pretty=format:"%h %s" --since="3 months ago" --grep="protected\|deploy\|main\|develop" | head
cat .github/workflows/*.yml 2>/dev/null | grep -E "branches:|push:" | head
```

This identifies protected branches by what CI runs against.

---

## Now produce the structured inventory in 7 sections

Use the discovery results above. Cite SPECIFIC files for every claim.

### 1. Project identity
- Project name (one phrase) — cite which manifest you read it from
- Project slug (lowercase-hyphenated) — derived from the name; if ambiguous, ASK
- One-sentence description — from manifest `description` or README first paragraph
- Primary tech stack — list each component with the file that confirmed it
  - Language: `<language>` (from `<file>`)
  - Framework: `<framework + version>` (from `<file>`)
  - Database: `<db>` (from `<file>` or `.env.example`)
  - Auth: `<provider>` (from deps or env vars)
  - Payments: `<provider or "none">` (from deps or env vars)
  - Email: `<provider or "none">`
  - Analytics: `<provider or "none">`
  - Real-time: `<provider or "none">`
  - LLM: `<provider or "none">`
  - Other notable services: `<list>`
- Deployment target — cite the deploy config file
- CI/CD — cite the workflow file(s)
- Protected branches — list with confirmation source
- Test runners — unit + E2E (cite config files)

### 2. Roles and audiences
List every distinct user type the project serves. Infer from:
- Route group structure (`/admin`, `/(auth)`, `/(customer)`, etc.)
- Permission-model files (RBAC config, middleware)
- README or PRD if present

For each role:
- Name
- One-sentence description of what they do
- Evidence (file path or doc that confirms)

If unsure whether a role exists:
`<NEEDS USER CONFIRMATION: Is "<proposed-role>" a real user type, or is it a code abstraction?>`

### 3. Domain boundaries (proposed specialists)
Propose specialists. For each:
- `name:` (lowercase-hyphenated, domain-shaped not technology-shaped)
- One-sentence description of what it owns
- Whether it should be REVIEW-ONLY or implementation
- Path globs it would primarily edit (must match real directories — verify with `ls`)
- Evidence: which signals from Steps C/D/E led you to propose this specialist

Aim for 5-10 specialists. Always include:
- An orchestrator named `<project-slug>-orchestrator`
- A `qa-functional` if there's any user-facing surface (test config exists in Step C)
- A `release-devops` if there's any deploy automation (deploy config + CI exist)
- A `security-privacy` (REVIEW-ONLY) if there's any user data
- A `legal-compliance` (REVIEW-ONLY) ONLY if regulatory exposure is confirmed (children's data, health, finance, EU users) — NOT speculatively. If unsure: `<NEEDS USER CONFIRMATION: Is this project subject to <regulation>? I see <evidence or lack thereof>.>`

For each specialist that DOESN'T have obvious need-to-exist evidence in the codebase, MARK it with `<NEEDS USER CONFIRMATION>` rather than including it speculatively.

### 4. Path-globbed rules (proposed)
For each specialist's domain, propose 1-2 rule files. For each rule file:
- File name (`<.claude/rules/<name>.md>`)
- `applies_to:` glob list (verify globs match real files via `find` or `ls`)
- Top 3-5 hard rules — each WITH source-confirmed evidence (cite file:line where the gotcha CURRENTLY lives in code or where a recent commit/PR fixed it)

If you cannot find evidence for a proposed rule:
- Either: don't propose it
- Or: mark `<NEEDS USER CONFIRMATION: I think rule X applies because Y, but I can't find a code example. Is this rule real?>`

NEVER include rules that are just generic best practices. The framework's value is project-specific gotchas, not lint config.

### 5. Existing documentation tier
Categorize every file in `docs/` (or wherever the project keeps docs):
- **Canonical** — full-detail, authoritative, currently accurate. Stays at `docs/<UPPERCASE>.md`.
- **Orientation candidate** — domain-specific, would benefit from being condensed into `docs/ai-context/<area>.md`. Suggest the new filename.
- **Archive candidate** — dated reports, sprint snapshots, post-mortems. Suggests the archive subdir (`docs/_archive/<YYYY-MM>/`).
- **Active workflow** — currently-in-progress plans. Keep but flag for re-categorization.

Cite each file's location and your categorization rationale.

### 6. Clutter / hygiene findings
Surface (don't fix yet) — for each, cite the path and the specific concern:
- Tracked tool caches (`.playwright-mcp/`, `.copilot-audit/`, `test-results/`, build artifacts) — list each with file count
- Orphan scripts at root level — list each with one-line description of what it appears to do
- Empty stub directories — list
- Loose screenshots / images at root — list
- Env file backups (`.env.local.bak*`, `.env.staging_tmp`, etc.) — **flag CRITICAL if tracked in git** (potential secrets in history)
- Files with stale paths (e.g. VSCode launch.json pointing to a renamed project) — list with diff of expected vs actual path
- Any obvious security concerns:
  - Env files in git history (`git log -p -- .env*` returns content)
  - Hardcoded secrets in source (grep for `sk_live_`, `password=`, common patterns)
  - Bind addresses set to `0.0.0.0` in non-server code

### 7. Invocation guidance for the team
Propose a one-paragraph snippet for the team's onboarding doc explaining when to use:
- `claude` (default — no flags)
- `claude --agent <project-slug>-orchestrator` (cross-domain / production-sensitive)
- `claude --agent <specialist>` (narrow single-domain)

Customize for THIS project's actual specialist list and protected-branch names.

---

## Output format rules

- Use the section headings above EXACTLY. Number them 1-7.
- Cite a specific file path or command output for every claim. "I see" without a citation is not enough.
- For anything you can't verify, mark `<NEEDS USER CONFIRMATION: <specific question>>` — do NOT guess silently.
- Do NOT create any files in this pass. This is read-only investigation.
- After producing the inventory, end with a section called `## Open questions` listing every `<NEEDS USER CONFIRMATION>` from above as a numbered list — so I can answer them in one pass.
- Then ask which sections I want to adjust before bootstrapping.

## What success looks like

After this prompt, I should be able to:
1. Read your inventory and immediately know what specialists you propose, with rationale per specialist
2. Answer all open questions in one message
3. Give you the green light (or adjustments) before any file is generated

If your output is missing any of: discovery command results, evidence citations per claim, or an Open Questions section — start over. The discipline of this prompt is the whole point.

---

(End of prompt.)
