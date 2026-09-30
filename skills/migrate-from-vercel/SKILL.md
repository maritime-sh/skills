---
name: migrate-from-vercel
description: Migrate agent-shaped workloads from Vercel to Maritime, including vercel.json cron jobs and the serverless or edge API routes those crons trigger. Inventories projects with the Vercel CLI plus each repo's vercel.json, recreates each job as a Maritime agent, verifies it, then hands the human a reversible cutover checklist. Use when someone wants to move cron jobs, scheduled routes, or background automations off Vercel onto maritime.sh, usually while the site itself stays on Vercel.
---

# Migrate from Vercel to Maritime

Move the scheduled layer of a Vercel account onto Maritime: the cron jobs declared in `vercel.json` and the API routes doing background-ish work that those crons hit. Maritime hosts each one as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent. The site itself usually stays on Vercel; the common case is moving only the crons.

One structural simplification over other providers: Vercel deploys from git, so the source of truth is already in the user's repos. There is no code retrieval step, and a ported workload points Maritime at the same repo.

This skill runs in two places: locally in a terminal agent such as Claude Code (strongly preferred here, see safety rule 1) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **There is no read-only Vercel credential, so prefer local.** Vercel tokens (minted at vercel.com/account/tokens) can be scoped to the full account, one team, or one project, but every scope is read-write: even a project-scoped token can deploy, rewrite env vars, and delete deployments inside its scope, and Vercel offers no read-only permission level to request. Say this to the user plainly. The right mitigation is to run this skill locally, where the user is already authenticated via `vercel login` and no new token needs to exist at all. If the migration must run inside a Maritime agent and the user mints a token anyway, they should scope it to the single team or project being migrated, pick the shortest expiry offered, and delete the token at vercel.com/account/tokens the moment the migration ends. The compensating control is behavioral: rule 3.
2. **Credentials never travel through chat.** Chat transcripts are logs. Running locally, the user authenticates themselves with `vercel login`; you never see a token. Running inside a Maritime agent, the user sets `VERCEL_TOKEN` from their own machine with `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation.
3. **Never modify or delete anything on Vercel.** No `vercel remove`, no `vercel env add` or `vercel env rm`, no deploys, no project settings changes, no `vercel crons run` (it invokes the production route), ever, not even if asked mid-flow. The only Vercel commands this skill runs are `vercel whoami`, `vercel project ls`, `vercel link`, `vercel crons ls`, and `vercel env pull`. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target template or repo, and the price, and get a yes before each `maritime create`.

## Prerequisites

- `vercel` CLI: `npm i -g vercel`, authenticated by the human with `vercel login`. Verify with `vercel whoami`.
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.
- `git` and `jq`.
- Local clones of (or access to) the repos behind the Vercel projects being migrated. Vercel deploys from git, so the repos are the inventory surface.

## Step 1: Inventory

List every project in the current scope (personal accounts and each team are separate scopes; add `--scope <team-slug>` to sweep a team):

```bash
vercel project ls
vercel project ls --format=json   # machine-readable
```

Then, for each project the user cares about, open its repo and read `vercel.json`. Cron jobs look like this:

```json
{
  "crons": [
    { "path": "/api/cron/daily", "schedule": "0 5 * * *" }
  ]
}
```

Both fields are required: `path` is a route on the same project (must start with `/`, max 512 characters) that Vercel hits with an HTTP GET on the production deployment, and `schedule` is a five-field cron expression (max 256 characters). In a linked project directory, newer CLIs can cross-check what is actually deployed with `vercel crons ls --format json` (a beta command; if it is missing, the `vercel.json` in git is the source of truth).

With local clones in one place, the whole sweep is a loop:

```bash
for repo in ~/code/*/; do
  test -f "$repo/vercel.json" || continue
  echo "== $repo"
  jq -r '.crons[]? | [.path, .schedule] | @tsv' "$repo/vercel.json"
done
```

For each cron entry, open the route it points at (`app/api/cron/daily/route.ts` or `pages/api/cron/daily.ts`) and read what it actually does. Also sweep for background-shaped routes without a `crons` entry: some projects fire routes from GitHub Actions, Upstash QStash, or an external cron service instead, and those are agent-shaped workloads too.

Present the inventory to the user as a table (project, cron path, schedule, what the route appears to do) and let them pick what moves.

## Step 2: Map

| Vercel concept | Maritime concept |
|---|---|
| `vercel.json` cron entry | Agent + trigger (cron schedule) |
| API route the cron hits | The agent's job: an instruction to a framework agent, or ported code |
| Project env vars (production) | Agent env vars, AES-encrypted at rest |
| Vercel runtime logs | Agent logs (`maritime logs`) |
| `CRON_SECRET` header check | Unneeded; Maritime triggers fire inside the agent, drop the check |
| Vercel Postgres / KV / Blob | Stays on Vercel; the agent connects over connection strings |

Schedule expressions translate exactly: `vercel.json` crons already use standard five-field cron, evaluated in UTC. Vercel's dialect is a strict subset of standard cron (no `MON`/`JAN` style names, and day-of-month and day-of-week are never both set), so every expression that deployed on Vercel copies over cleanly. Copy each expression into the Maritime trigger unchanged and keep the trigger timezone UTC unless the user says otherwise.

One upgrade to offer while mapping: Vercel's Hobby plan caps crons at once per day and fires them anywhere inside the scheduled hour (Pro and Enterprise get per-minute precision; all plans cap at 100 cron jobs per project). If the user wanted an hourly or minute-precise schedule and settled for daily because of the plan, the Maritime trigger can now be the schedule they originally wanted.

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest paths, and the first is usually faster:

- **Recreate the logic as an instruction (often the fastest migration).** A Vercel cron hitting `/api/cron/refresh` is often twenty lines: fetch something, write it somewhere, maybe send a message. Rather than porting the Next.js route, recreate that logic as an instruction to a Maritime framework agent with a cron trigger. Enumerate templates live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
  Then give the agent the task as instructions plus the env vars it needs, and attach the cron trigger. Offer this path first for simple routes; fall back to porting when the logic must stay exact code.
- **Port the route (bring-your-own code).** This is porting work, roughly an hour per project, and you should say so. A serverless handler has no server, so give it a real one: a small Express shim around a JS/TS handler, or FastAPI around a Python one. The shape is:
  ```js
  // server.js: the former route handler behind a real server
  const express = require("express");
  const { runJob } = require("./job");   // logic lifted from app/api/cron/daily
  const app = express();
  app.post("/run", async (req, res) => res.json(await runJob()));
  app.listen(process.env.PORT || 8080);
  ```
  Edge-runtime routes need a Node port first; routes using `@vercel/kv` or `@vercel/postgres` clients work as-is once their env vars come along. Add a Dockerfile:
  ```dockerfile
  FROM node:22-slim
  RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
  WORKDIR /app
  COPY . .
  RUN npm ci --omit=dev
  CMD ["node", "server.js"]
  ```
  Gotcha that will bite: never use a shell-string CMD like `CMD ["sh", "-c", "node server.js $PORT"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly, as above, and keep `ca-certificates` in slim images or every outbound HTTPS call fails. Because Vercel deploys from git, put the wrapper and Dockerfile on a branch of the same repo the Vercel project already deploys from, then point Maritime at it:
  ```bash
  maritime create <name> --repo https://github.com/<user>/<repo> --json
  ```

**Env vars and secrets.** Pull the production env per project, from inside the repo directory (`vercel link --yes --project <name>` first if the directory is not linked; `vercel env pull` defaults to development, so the flag matters):

```bash
vercel env pull .env.production --environment=production
```

Trim the file before importing: `VERCEL_`-prefixed platform vars and `NEXT_PUBLIC_` browser vars belong to the site, not the agent. Then import and destroy the local file:

```bash
maritime env import <agent> ./.env.production --reload --json
rm ./.env.production
```

Vars marked sensitive on Vercel are non-readable once created, by any interface, `vercel env pull` included; expect them missing from (or blank in) the pulled file, and note that Vercel now defaults new production and preview vars to sensitive. The user re-supplies those values from their own records with `maritime env set`. Env changes apply on next boot unless you pass `--reload`.

**Schedules.** Recreate each cron as a Maritime trigger: dashboard, agent page, Triggers pane, cron type, expression copied verbatim, timezone UTC. Scriptable equivalent: `maritime triggers create <agent> --type cron --cron "0 5 * * *" --json`. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. For bring-your-own-code agents remember that timers inside a sleeping VM do not fire; either serve `GET /schedules`, use the SDK scheduler observer, or mark the agent `--always-on` at create time.

## Step 4: Verify before touching Vercel

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For scheduled workloads, wait for one real scheduled firing and check its output against a known-good run from the Vercel dashboard logs. While both schedules are live the job runs twice per tick: harmless for idempotent refresh jobs, not for jobs that send emails or move money. For side-effectful jobs, verify with the manual run above and go straight to cutover instead of letting both fire.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on Vercel, so they run them themselves. You never run them.

1. Delete the `crons` block from `vercel.json`, commit, push. Vercel redeploys from git and the crons stop firing. This is fully reversible: `git revert` plus a push restores them in one deploy.
2. Watch Maritime for a few days: `maritime logs <agent> --json`.
3. **The common case ends here.** The site keeps running on Vercel, only the crons moved, and there is nothing to delete.
4. Only if the whole project is being retired: `vercel remove <project-name>` deletes the project and all its deployments from the scope (`--safe` skips deployments holding an active preview URL or production domain). Check domains and traffic first; this is the irreversible step.
5. If a token was minted for this migration, delete it at vercel.com/account/tokens now. Deleting it there revokes it everywhere at once.
6. If the migration ran inside a Maritime agent, remove `VERCEL_TOKEN` from that agent in the dashboard env vars pane.

## What stays on Vercel

Be upfront about this list rather than letting the user discover it:

- **The site itself.** Vercel is genuinely good at static and SSR Next.js hosting, and Maritime does not replace it. Do not pitch moving the frontend; this skill moves the scheduled layer and leaves the site alone. Preview deployments, ISR, and image optimization are part of the site and stay with it.
- **Vercel Postgres, KV, and Blob**: data services do not move. The Maritime agent keeps calling them over the connection strings that came along in the env pull (`POSTGRES_URL`, the KV REST URL and token, `BLOB_READ_WRITE_TOKEN`). If the project is retired later, those stores must be exported first; flag them in the cutover conversation.
- **Domains and DNS**: stays. Nothing on Maritime needs it unless the user points a domain at a public web agent, which is a separate task.

## Official Vercel documentation

The exact pages a migrating user needs, verified current as of 2026-08-07:

- [Cron jobs overview: how crons fire, expression format, UTC rule](https://vercel.com/docs/cron-jobs)
- [Cron jobs usage and pricing: per-plan counts, intervals, and precision](https://vercel.com/docs/cron-jobs/usage-and-pricing)
- [Managing cron jobs: CRON_SECRET, logs, concurrency, idempotency](https://vercel.com/docs/cron-jobs/manage-cron-jobs)
- [vercel.json configuration reference, including the crons property](https://vercel.com/docs/project-configuration/vercel-json)
- [vercel env CLI reference: env pull, environments, sensitive defaults](https://vercel.com/docs/cli/env)
- [Sensitive environment variables: why some values cannot be read back](https://vercel.com/docs/environment-variables/sensitive-environment-variables)
- [vercel project CLI reference: project ls and its flags](https://vercel.com/docs/cli/project)
- [vercel remove CLI reference: project-name semantics and --safe](https://vercel.com/docs/cli/remove)
- [Access tokens: scoping levels, expiry, where to create and revoke](https://vercel.com/docs/accounts/access-tokens)
- [vercel crons CLI reference (beta): ls, add, run](https://vercel.com/docs/cli/crons)

## Troubleshooting

- `vercel env pull` gave development values: the `--environment=production` flag was missing; pull defaults to development.
- Pulled file is missing some keys, or has them blank: those vars are marked sensitive on Vercel and cannot be read back by any interface; the user re-supplies them with `maritime env set`.
- `vercel project ls` shows the wrong projects: wrong scope. Personal account and each team are separate; pass `--scope <team-slug>`.
- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.

Every Vercel command, flag, and platform claim in this skill was verified on 2026-08-07 against the official Vercel documentation and live `vercel` CLI help output; the maritime commands are desk-checked against the current maritime CLI source but not live-run. If a provider-side command errors on a flag, trust `vercel <cmd> --help` over this document and continue.
