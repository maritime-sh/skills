---
name: migrate-from-fly
description: Migrate agent-shaped workloads from Fly.io to Maritime, including scheduled Machines, fly.toml worker processes, and bots or automations running as ordinary Fly apps. Inventories the Fly organization with a read-only token, redeploys each app's existing Dockerfile as a Maritime agent, verifies it, then hands the human a decommission checklist. Use when someone wants to move automations, bots, or cron jobs off Fly.io onto maritime.sh.
---

# Migrate from Fly.io to Maritime

Move the automation layer of a Fly.io organization onto Maritime: scheduled Machines, worker processes declared in fly.toml, and bots or queue drainers running as ordinary Fly apps. Fly is one of the friendlier sources to migrate from: every Fly app is already a container, so the Dockerfile usually exists in the user's repo and `maritime create <name> --repo <url>` is nearly direct. Maritime hosts each workload as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent.

This skill runs in two places: locally in a terminal agent such as Claude Code (preferred, credentials stay on the user's machine) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **Read-only Fly token only.** Ask the user to mint a scoped read-only org token; that is all inventory needs:
   ```bash
   fly tokens create readonly -x 48h
   ```
   The human runs that with their own login; you only ever receive the resulting token. It can read a single org and its resources, nothing else. The `-x 48h` matters: Fly tokens default to a 20 year expiry, and a short-lived token cleans up after itself even if revocation is forgotten. If the user offers the output of `fly auth token` (a full-access personal token) or a deploy token, refuse it and show the readonly command instead. Remind them to revoke the migration token when the migration is done. If the subcommand errors, trust `fly tokens create --help` over this document.
2. **Credentials never travel through chat.** Chat transcripts are logs. Running locally, the user exports `FLY_API_TOKEN` in their own shell. Running inside a Maritime agent, the user sets `FLY_API_TOKEN` from their own machine with `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation.
3. **Never modify or delete anything on Fly.** No `fly apps destroy`, no `fly machine stop`, no `fly secrets unset`, no `fly scale`, ever, not even if asked mid-flow. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target shape (template or repo), and the price, and get a yes before each `maritime create`.

## Prerequisites

- `fly` CLI (flyctl), authenticated with the read-only token above via `FLY_API_TOKEN`. `fly auth whoami` reports the token's identity rather than the human's email when the env var is set; `fly apps list` returning the org's apps is the real smoke test.
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.
- `jq`.

## Step 1: Inventory

All of these are read operations, safe under the readonly token:

```bash
# Every app in the organization
fly apps list

# Per app: overall state and the machines behind it
fly status -a <app>
fly machine list -a <app>

# The live app config as deployed (JSON by default; --toml prints fly.toml shape).
# Look at [processes] for worker processes beyond the default web process.
fly config show -a <app>

# Secret NAMES and digests only; values are not retrievable, by design
fly secrets list -a <app>
```

Scheduled Machines are Fly's cron analog: created with `fly machine run <image> --schedule hourly|daily|weekly|monthly`, they show up in `fly machine list` and carry a `schedule` field in the machine config (`fly machine list --json` includes the full config; `fly machine status -d <machine-id> -a <app>` prints one machine's config as JSON if you need to double-check a single machine).

Agent-shaped workloads are the ones that wake up, do a job, and go quiet: scheduled Machines, report generators, scrapers, Slack/Telegram bots, queue drainers, sync jobs, and any `[processes]` worker with no public traffic. Present the inventory to the user as a table (app, process or machine, schedule if any, what it appears to do) and let them pick what moves.

## Step 2: Map

| Fly concept | Maritime concept |
|---|---|
| Fly app (container per Machine) | Agent (bring-your-own code) |
| Scheduled Machine (`--schedule`) | Agent + trigger (cron schedule) |
| fly.toml `[processes]` worker | Its own agent, one per process |
| `fly secrets` | Agent env vars, AES-encrypted at rest |
| `fly logs` | Agent logs (`maritime logs`) |
| Machine auto stop/start | Agent auto-sleep (default) or `--always-on` |

Schedules translate up, not across. Fly offers only four coarse intervals (hourly, daily, weekly, monthly) and fires them on a fuzzy, approximate cycle anchored to the Machine's own runs; the user never chose a wall-clock time. Maritime triggers are real five-field cron, so this is an upgrade: ask the user when the job should actually run instead of transcribing blindly. Mechanical defaults if they have no preference: hourly becomes `0 * * * *`, daily `0 0 * * *`, weekly `0 0 * * 0`, monthly `0 0 1 * *`, all UTC.

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest cases:

- **The workload is an LLM-driven bot or assistant.** Recreate it on a Maritime framework template. Enumerate live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
- **The workload is the app's own containerized code, which is the common Fly case.** The repo already has a Dockerfile because Fly deployed from one (hand-written or generated by `fly launch`), so this is close to a direct move:
  ```bash
  maritime create <name> --repo https://github.com/<user>/<repo> --json
  ```
  Before deploying, READ the Dockerfile. Two things bite:
  - Never a shell-string CMD like `CMD ["sh", "-c", "python main.py $PORT"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly, exec form. Fly's builder tolerates shell-string CMDs, so a long-deployed Fly app proves nothing here; check anyway.
  - Install `ca-certificates` in slim images.

  fly.toml `[processes]` command overrides do not travel with the Dockerfile. Give each worker process its own agent, and make its Dockerfile CMD (in a branch or a small dedicated repo) launch that process's command directly.

  If only an image remains because the source repo is lost, `fly image show -a <app>` names the deployed image ref, but prefer recovering the source: `registry.fly.io` images need Fly credentials to pull, and rebuilding from source keeps the agent reproducible.

**Sizing and sleep.** Fly Machines auto stop and start; Maritime's default is the same shape, agents auto-sleep when idle and wake on demand, so most workloads need no sizing flags at all. If a Fly Machine was pinned to a specific VM size (`fly machine list` shows each machine's size), mirror it with the optional create-time overrides:

```bash
maritime create <name> --repo <url> \
  --ram 2048 \        # MB, mirror the Fly Machine memory
  --cpu 1 \           # vCPU cores
  --idle 600 \        # sleep 600s after the last request
  --json
```

Use `--always-on` instead of `--idle` for bots that hold a persistent connection (Telegram, Discord, long-lived queue consumers); scheduled and polling jobs should keep the default auto-sleep. Resources apply at provision time; changing `--ram`/`--cpu`/`--disk` later needs a recreate, while idle settings update live.

**Env vars and secrets.** Say this plainly to the user up front: `fly secrets list` shows names and digests only, and secret values are not retrievable through the secrets API by design; only the running app ever sees them. The values must come from the user's own records: a password manager, local `.env` files, or the dashboards of the upstream services the secrets belong to. One more honest source: since the running app sees its secrets as plain env vars, the human, with their own full-access login, can read them off a live Machine with `fly ssh console -a <app>` and `env`. That is their command to run, never yours, and the readonly token cannot do it. Do not suggest `fly console` for this; it boots a new billable Machine instead of entering the existing one. Build the list of required names from `fly secrets list` and `fly config show` (the `[env]` section holds the non-secret vars), then have the user assemble a local file themselves:

```bash
# The user fills fly.env from their own records, one KEY=value per line.
maritime env import <agent> ./fly.env --reload --json
rm ./fly.env
```

If a value is genuinely unrecoverable, rotate it at the upstream service and use the new value; that is normal and safer than archaeology. Env changes apply on next boot unless you pass `--reload`.

**Schedules.** Recreate each scheduled Machine as a Maritime trigger: dashboard, agent page, Triggers pane, cron type, using the expression agreed in Step 2. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. For bring-your-own-code agents remember that timers inside a sleeping VM do not fire; either serve `GET /schedules`, use the SDK scheduler observer, or mark the agent `--always-on` at create time.

## Step 4: Verify before touching Fly

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For scheduled workloads, wait for one real trigger firing and compare its output against a known-good run from `fly logs -a <app>`. For auto-sleep agents, also confirm the job survives a sleep/wake cycle: let the agent go idle, then trigger it again and check the logs. Both sides can safely run in parallel; that is the point of leaving Fly untouched.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on Fly, so they run them with their own full-access login. You never run them.

1. Pause the old app (reversible): `fly machine stop <machine-id> -a <app>` for each Machine, reversed with `fly machine start <machine-id> -a <app>`. Stopping keeps the Machines and their config in place, which is exactly what a trial cutover wants. `fly scale count 0 -a <app>` also pauses the app (name every process on a multi-process app: `fly scale count web=0 worker=0`), but know what it does: scaling down destroys the Machines, and restoring means recreating them with `fly scale count web=1` (add `-g <process>` variants as needed) or a fresh `fly deploy`. Prefer stop/start for the trial window.
2. Watch Maritime for a few days: `maritime logs <agent> --json`. If anything is wrong, scale Fly back up and keep both running.
3. Only when satisfied, retire the app: `fly apps destroy <app>`. This permanently removes its Machines, volumes, and IPs, so export any volume data first (see below).
4. Revoke the migration token: `fly tokens list --scope org` (the readonly token is org-scoped; the list defaults to app scope), then `fly tokens revoke <token-id>`. If those subcommands have drifted, trust `fly tokens --help` over this document.
5. If the migration ran inside a Maritime agent, remove `FLY_API_TOKEN` from that agent via the dashboard env vars pane.

## What stays on Fly

Be upfront about this list rather than letting the user discover it:

- **Fly Postgres and anything else on the private network.** Data does not move, and reachability is the hard part. `fly proxy` is a WireGuard tunnel on the user's local machine, so a Maritime agent cannot use it. If the database is reachable only over Fly's private 6PN network, the agent cannot connect at all. The honest options: enable Fly's public connection path for that Postgres (external port on a dedicated IP), move the database to a managed provider, or leave that workload on Fly. A private-network-only database is a genuine blocker to weigh, not a footnote.
- **Volumes and LiteFS.** Fly volume data does not migrate; the human exports anything they need before step 3 above deletes it (app deletion removes the volumes and their snapshots together). LiteFS-replicated SQLite is Fly-specific, so an app built around it keeps its data plan on Fly or redesigns storage first.
- **Anycast IPs, certificates, custom domains.** These stay with Fly. If a public web agent on Maritime replaces the app, re-pointing DNS is a separate task after the migration, not part of it.

## Official Fly.io documentation

The exact pages a migrating user needs, verified current as of 2026-08-07:

- [flyctl command reference](https://fly.io/docs/flyctl/): index of every fly command used in this skill.
- [Access tokens](https://fly.io/docs/security/tokens/): token varieties, what the read-only org token can and cannot do, listing and revoking.
- [fly tokens create readonly](https://fly.io/docs/flyctl/tokens-create-readonly/): the one command the human runs for you, including the `-x` expiry flag and the 20 year default.
- [Run a new Machine](https://fly.io/docs/machines/flyctl/fly-machine-run/): `fly machine run`, including the `--schedule` flag and its fuzzy hourly/daily/weekly/monthly cycle.
- [Task scheduling on Fly](https://fly.io/docs/blueprints/task-scheduling/): Fly's own comparison of scheduled Machines against cron alternatives, and why interval buckets are the only granularity.
- [fly.toml reference](https://fly.io/docs/reference/configuration/): the `[processes]` section that defines worker process groups and the `[env]` section holding non-secret vars.
- [Secrets](https://fly.io/docs/apps/secrets/): how secrets reach the app as env vars and why plain-text read-back is not allowed.
- [Private networking](https://fly.io/docs/networking/private-networking/): 6PN, `.internal` addresses, and why nothing outside the org's WireGuard mesh can reach them.
- [Connect to Postgres from outside Fly](https://fly.io/docs/postgres/connecting/connecting-external/): the dedicated-IP-plus-external-port recipe referenced under "What stays on Fly".
- [Delete an app](https://fly.io/docs/apps/delete/): confirms `fly apps destroy` removes Machines, volumes, snapshots, and IPs, and to export data first.

## Troubleshooting

- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.
- A secret nobody can find the value for: the secrets API never returns values, and if no Machine is still running there is nothing for the human to read it from. Rotate it at the upstream service and import the new value.
- A `fly` read command returns 403 under the readonly token: some subcommands want broader scope. Have the human run that single read with their own login and paste the output; do not upgrade the migration token.

Every fly command and flag in this skill was verified against the official Fly.io documentation linked above on 2026-08-07, and the maritime commands against the current maritime CLI source; flyctl itself was not run live during that pass. If a provider-side command errors on a flag, trust `fly <cmd> --help` over this document and continue.
