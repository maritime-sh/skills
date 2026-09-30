---
name: migrate-from-heroku
description: Migrate agent-shaped workloads from Heroku to Maritime, including worker dynos, Heroku Scheduler jobs, and clock processes with in-code schedules. Inventories apps and Procfiles with the Heroku CLI, recreates each workload as a Maritime agent with real cron triggers, verifies it, then hands the human a reversible cutover checklist. Use when someone wants to move workers, bots, or scheduled jobs off Heroku onto maritime.sh.
---

# Migrate from Heroku to Maritime

Move the automation layer of a Heroku account onto Maritime: worker dynos, Heroku Scheduler jobs, and clock processes. These are exactly agent-shaped, a process that wakes up, does a job, and goes quiet, so Heroku is one of the best-fit sources. Maritime hosts each one as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent. The common outcome: the web dyno stays on Heroku serving the site, and only the workers and schedulers move.

This skill runs in two places: locally in a terminal agent such as Claude Code (preferred, credentials stay on the user's machine) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **Mint a read-only, expiring token; never hand over a full-access credential.** Heroku's default credentials are full-access: the CLI login token and the account API key carry the `global` scope. But OAuth authorizations accept scopes, and `read-protected` is read-only while still covering config vars, which the env export in Step 3 needs (plain `read` excludes config vars). The honest posture is still: prefer running locally where the user is already `heroku login`-ed and no credential ever moves. If the migration must run inside a Maritime agent, the human mints a scoped, expiring token on their own machine and revokes it the moment the migration ends:
   ```bash
   heroku authorizations:create -d maritime-migration -s read-protected -e 86400   # human runs; note the Id and Token
   # ...after the migration:
   heroku authorizations:revoke <id>
   ```
   `-e` is seconds, so this token dies on its own after a day; revoke it anyway. Two caveats: account-scope commands fail under it (`heroku auth:whoami` reads account info, which the scope excludes), so sanity-check the token with `heroku apps` instead; and the scope is Heroku's server-side backstop, not a license to try writes, so rule 3 below is still absolute.
2. **Credentials never travel through chat.** Chat transcripts are logs. Locally, the Heroku CLI holds its own auth. Inside a Maritime agent, the user sets `HEROKU_API_KEY` from their own machine with `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation.
3. **Never run a mutating heroku command.** No `ps:scale`, no `config:set`, no `addons:destroy`, no `apps:destroy`, ever, not even if asked mid-flow. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target template, and the price, and get a yes before each `maritime create`.

## Prerequisites

- `heroku` CLI, logged in. Verify with `heroku auth:whoami`. Inside a Maritime agent, `HEROKU_API_KEY` holds the scoped token from the safety section instead, and the sanity check is `heroku apps` (whoami fails by design under a scoped token).
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.
- `jq`.

## Step 1: Inventory

Three read-only sweeps, plus one honest ask.

```bash
# Every app on the account
heroku apps --json | jq -r '.[].name'

# Dyno formation per app: worker and clock lines are the targets, web usually stays
heroku ps -a <app>

# Add-ons: look for scheduler, heroku-postgresql, heroku-redis
heroku addons -a <app>

# Recent releases help spot dead apps not worth migrating
heroku releases -a <app>
```

**Procfile.** The repo's Procfile names the process types. `web:` serves HTTP and usually stays on Heroku. `worker:` and `clock:` lines are the agent-shaped workloads; record each one's exact command.

**Heroku Scheduler, the honest part.** Scheduler jobs cannot be listed from the CLI; there is no command for it. Ask the user to open `https://dashboard.heroku.com/apps/<app>/scheduler` (or run `heroku addons:open scheduler -a <app>`) and read each job back: the command, the frequency (every 10 minutes, hourly, or daily), and the UTC time. That dictated list is the inventory; write it down verbatim.

**Clock processes.** A `clock:` dyno keeps its schedules in code, not in Heroku. Grep the repo for the scheduler library (APScheduler, `whenever`, node-cron, celery beat) and extract each schedule expression. Each one becomes its own Maritime trigger.

Present the inventory to the user as a table (app, process or job, schedule, what it appears to do) and let them pick what moves.

## Step 2: Map

| Heroku concept | Maritime concept |
|---|---|
| Worker dyno (`worker:` in Procfile) | Agent, likely `--always-on` |
| Heroku Scheduler job | Agent + cron trigger, auto-sleep |
| Clock process (in-code schedules) | Agent + one trigger per schedule |
| Config vars | Agent env vars, AES-encrypted at rest |
| `heroku logs` | Agent logs (`maritime logs`) |
| Heroku Postgres / Redis add-ons | Stay on Heroku; the agent connects over the public URL |

**Schedules convert upward.** Heroku Scheduler supports only three shapes, all at fixed UTC times, so the mapping to real cron is mechanical and strictly an upgrade:

| Heroku Scheduler setting | Maritime cron |
|---|---|
| Every 10 minutes | `*/10 * * * *` |
| Hourly at :30 | `30 * * * *` |
| Daily at 05:00 UTC | `0 5 * * *` |

Keep the trigger timezone UTC so behavior matches on day one. Then, if the user wants, express what Scheduler never could: weekday-only runs, every 5 minutes, monthly reports.

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest cases:

- **The workload is an LLM-driven bot or assistant.** Recreate it on a Maritime framework template. Enumerate live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
- **The workload is arbitrary code (most worker dynos and Scheduler jobs).** This is a port, not a copy, and you should say so. The code is usually already in the user's own git repo, since that is how it reached Heroku. If the only copy lives on Heroku:
  ```bash
  heroku git:clone -a <app>
  ```
  Heroku itself calls this a last resort; it only works for apps deployed with `git push heroku`, not the GitHub integration or the container registry (for those, the user's own repo is the source). Run the clone locally under the user's own login; the git endpoint is not guaranteed to accept a scoped token.

  Buildpack apps have no Dockerfile, so write one: base image matching the runtime, copy the app, install dependencies, and make the Procfile's worker command the CMD in exec form. Dockerfile gotcha that will bite: never use a shell-string CMD like `CMD ["sh", "-c", "python worker.py"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly. Install `ca-certificates` in slim images. Push to a repo the user controls, then:
  ```bash
  maritime create <name> --repo https://github.com/<user>/<repo> --json
  ```

**Env vars.** Config vars export cleanly in shell format. Write them to a local file, review it (keep `DATABASE_URL`, `REDIS_URL`, and the API keys the worker reads; drop web-only vars), import, then destroy the file:

```bash
heroku config -a <app> --shell > ./heroku-migration.env
maritime env import <agent> ./heroku-migration.env --reload --json
rm ./heroku-migration.env
```

Env changes apply on next boot unless you pass `--reload`.

**Schedules.** Recreate each Scheduler job and clock entry as a Maritime trigger: dashboard, agent page, Triggers pane, cron type, using the converted expression. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. Remember that timers inside a sleeping VM do not fire; a bring-your-own-code agent must serve `GET /schedules`, use the SDK scheduler observer, or be created `--always-on`. Rule of thumb: a former always-running worker dyno wants `--always-on`; a Scheduler job wants a cron trigger with auto-sleep, which is usually cheaper than the dyno hours it replaces.

## Step 4: Verify before touching Heroku

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For scheduled workloads, wait for one real scheduled firing and check its output against a known-good run from `heroku logs --process-type scheduler -a <app>` (or `--process-type worker`). Older heroku CLIs called this flag `--ps`; if either spelling errors, `heroku logs --help` settles it. Both sides can usually run in parallel; that is the point of leaving Heroku untouched. One caution: a worker draining a shared queue or writing non-idempotent output will double-process during the overlap. If that matters, keep the overlap window short or point the Maritime agent at a staging target first.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on Heroku, so they run them with their own credentials. You never run them.

1. Scale the moved processes to zero (reversible): `heroku ps:scale worker=0 -a <app>`, and `clock=0` if there is one. `worker=1` restores Heroku in seconds.
2. Remove the Scheduler jobs in the dashboard at `https://dashboard.heroku.com/apps/<app>/scheduler`. No CLI for this either; re-adding a job by hand is the rollback.
3. Watch Maritime for a few days: `maritime logs <agent> --json`.
4. The common case ends here: the web dyno keeps serving the site, the app keeps running with `worker=0`, and the bill drops to web only. Stop; do not destroy anything.
5. Only when the entire app is retired: `heroku apps:destroy -a <app> --confirm <app>`. This deletes the app's add-ons and their data with it, including Heroku Postgres. Capture and download a final backup first (`heroku pg:backups:capture -a <app>`, then `heroku pg:backups:download -a <app>`), and only destroy an app whose database the agent no longer uses.
6. If a migration token was minted in the safety section, revoke it: `heroku authorizations` to find the id, then `heroku authorizations:revoke <id>`. If the migration ran inside a Maritime agent, also `maritime env remove <agent> HEROKU_API_KEY`.

## What stays on Heroku

Be upfront about this list rather than letting the user discover it:

- **Heroku Postgres and Key-Value Store add-ons** (the latter is Heroku Redis's current name): keep them. Copy `DATABASE_URL` and `REDIS_URL` into the Maritime agent's env vars and keep using them over the public connection string, which is exactly how the dynos reached them anyway. One catch: Heroku states plainly that add-on config var values can change at any time (provider maintenance, failover, or a manual `heroku pg:credentials:rotate`), so pull fresh values at cutover with `heroku config:get DATABASE_URL -a <app>` and expect to update the agent's env if a rotation happens later.
- **The web dyno**: if it serves the site, it stays, along with its custom domains and SSL. Moving a public website is a different project from moving its background jobs; do not conflate them.
- **Other add-ons (SendGrid, Mailgun, Papertrail, and friends)**: the services keep working. Any add-on-provisioned key the worker used is already in the config vars export from Step 3 and travels with the env import.

## Official Heroku documentation

The exact Dev Center pages a migrating user needs, in the order this skill touches them:

- [Heroku Scheduler](https://devcenter.heroku.com/articles/scheduler): the three frequencies, UTC times, dashboard-only job management, and the `scheduler.X` dyno naming.
- [Procfile](https://devcenter.heroku.com/articles/procfile): process types, why `web` is the only type that receives HTTP traffic, and how `worker` and `clock` lines are declared.
- [Configuration and config vars](https://devcenter.heroku.com/articles/config-vars): how config vars work, plus the warning that add-on-set values can change at any time.
- [OAuth authorizations and scopes](https://devcenter.heroku.com/articles/oauth): the scope table behind the `read-protected` recommendation in the safety rules.
- [Heroku CLI authentication](https://devcenter.heroku.com/articles/authentication): `HEROKU_API_KEY` and how it overrides the stored login.
- [Dyno types](https://devcenter.heroku.com/articles/dyno-types): what each dyno tier costs, for the before-and-after bill comparison.
- [Heroku CLI commands reference](https://devcenter.heroku.com/articles/heroku-cli-commands): every command and flag in one page, including `apps:destroy` and `authorizations:create`.
- [Heroku Postgres credentials](https://devcenter.heroku.com/articles/heroku-postgresql-credentials): credential rotation and how a rotation rewrites config vars and restarts apps.
- [Heroku Key-Value Store](https://devcenter.heroku.com/articles/heroku-redis): the Redis add-on's current name and its explicitly-unstable connection strings.
- [Cloning an existing Heroku app](https://devcenter.heroku.com/articles/git-clone-heroku-app): why `git:clone` is unsupported-last-resort and which deploy methods leave it empty.

## Troubleshooting

- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.
- `heroku git:clone` yields an empty or stale repo: the app was deployed via the GitHub integration or container registry; the user's own repo is the real source.
- The scheduler dashboard page 404s: the app has no Scheduler add-on (`heroku addons -a <app>` confirms); the schedules are probably in a clock process, so go back to the Procfile and grep the code.
- `heroku ps` shows transient `scheduler.N` dynos: those are Scheduler runs in flight, not a process type to migrate; the jobs behind them are the migration targets.

Heroku syntax in this skill was verified on 2026-08-07 against live `heroku --help` output (CLI v10.8.0) and the Dev Center pages linked above; maritime syntax is checked against the current maritime CLI source. What remains desk-checked is behavior against a live Heroku account, since verification ran logged out. If a provider-side command errors on a flag, trust `heroku <cmd> --help` over this document and continue.
