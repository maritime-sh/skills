---
name: migrate-from-digitalocean
description: Migrate agent-shaped workloads from DigitalOcean to Maritime, including App Platform workers and jobs, DO Functions with scheduled triggers, and droplet-hosted bots. Inventories the account with a read-only scoped API token, recreates each workload as a Maritime agent, verifies it, then hands the human a decommission checklist. Use when someone wants to move automations, bots, or cron jobs off DigitalOcean onto maritime.sh.
---

# Migrate from DigitalOcean to Maritime

Move the automation layer of a DigitalOcean account onto Maritime: App Platform workers and jobs, DO Functions on cron triggers, and bots squatting on droplets. Maritime hosts each one as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent.

This skill runs in two places: locally in a terminal agent such as Claude Code (preferred, credentials stay on the user's machine) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **Read-only DigitalOcean token only.** DigitalOcean API tokens support custom scopes with a one-click Read Only preset, so there is no excuse for a full-access token here. Ask the user to mint one at cloud.digitalocean.com/account/api/tokens (Account, API, Tokens tab, Generate New Token) and choose **Read Only**, which grants the `api:read` alias scope, or Custom Scopes with only read scopes checked. If the user offers a legacy full-access token, refuse it and show the scoped flow instead; token scopes cannot be edited after creation, so a fresh token is the only path. The human authenticates it themselves:
   ```bash
   doctl auth init --context maritime-migration   # doctl prompts for the token directly
   doctl account get                              # verify the token works
   ```
   Remind them to revoke this token in the control panel when the migration is done.
2. **Credentials never travel through chat.** Chat transcripts are logs. Running locally, the user runs `doctl auth init` themselves and the token never appears in the conversation. Running inside a Maritime agent, the user sets `DIGITALOCEAN_ACCESS_TOKEN` from their own machine with `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation; doctl reads that env var directly.
3. **Never modify or delete anything on DigitalOcean.** No `apps update`, no `apps delete`, no `droplet delete`, no `droplet-action power-off`, no `triggers disable`, ever, not even if asked mid-flow. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target template, and the price, and get a yes before each `maritime create`.

## Prerequisites

- `doctl`, authenticated with the read-only token above. Verify with `doctl account get`.
- For Functions inventory only: `doctl serverless install` once (installs local support), then `doctl serverless connect`. Both work under the read-only token.
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.
- `jq`.

## Step 1: Inventory

Three places agent-shaped work hides on DigitalOcean, plus the data layer:

```bash
# App Platform: list apps, then read each spec
doctl apps list
doctl apps spec get <app-id>        # YAML with services, workers, jobs, functions components

# DO Functions and their cron triggers
doctl serverless functions list
doctl serverless triggers list      # columns: Name, Cron, Function, Enabled, LastRun

# Droplets that might host bots
doctl compute droplet list

# Managed databases the workloads read (inventory only, the data stays)
doctl databases list
```

To pull just the agent-shaped components out of a busy app spec:

```bash
doctl apps spec get <app-id> --format json | \
  jq '{workers: [.workers[]?.name], jobs: [.jobs[]? | {name, kind, schedule}], functions: [.functions[]?.name]}'
```

**App Platform now has a native cron component.** A `workers:` entry is a long-running process. A `jobs:` entry is either a deploy-time hook (`kind: PRE_DEPLOY`, `POST_DEPLOY`, or `FAILED_DEPLOY`) or a real cron job: `kind: SCHEDULED` with a `schedule` block holding a five-field `cron` expression and a tz-database `time_zone`, minimum frequency every 15 minutes. Scheduled jobs are a recent addition, so older apps still fake cron with a worker running its own scheduler loop or an external pinger hitting a service endpoint; read the spec first, then the code, so you find both kinds of schedule. One more wrinkle: `functions` components inside an app spec cannot have scheduled triggers at all (DigitalOcean says so outright), so if an in-app function appears to run on a schedule, something external invokes it.

**Droplets: hand off, do not duplicate.** What runs inside a droplet (systemd units, crontabs, pm2, tmux sessions) is exactly the SSH inventory the companion skill `migrate-from-vps` in this folder family already does. Use `doctl compute droplet list` to find candidate droplets and their IPs, then switch to that skill for everything inside the box. This document covers only the droplet's DigitalOcean-side lifecycle (power off, delete) in the cutover list.

Agent-shaped workloads are the ones that wake up, do a job, and go quiet: cron functions, report generators, scrapers, Slack/Telegram bots, queue drainers, sync jobs. Present the inventory to the user as a table (workload, trigger, runtime, what it appears to do) and let them pick what moves.

## Step 2: Map

| DigitalOcean concept | Maritime concept |
|---|---|
| App Platform worker | Agent (always-on or auto-sleep) |
| App Platform job (deploy-time) | Fold into the agent's own startup or deploy; rarely its own agent |
| App Platform scheduled job (`kind: SCHEDULED`) | Agent + trigger (cron schedule) |
| DO Function + cron trigger | Agent + trigger (cron schedule) |
| Worker running a scheduler loop | Agent + trigger, with the loop removed |
| Droplet-hosted bot | Agent via migrate-from-vps |
| App spec `envs`, Functions env | Agent env vars, AES-encrypted at rest |
| App Platform runtime logs, `doctl serverless activations logs` | Agent logs (`maritime logs`) |

DO Functions triggers and App Platform scheduled jobs both use standard five-field cron, so expressions copy to Maritime triggers unchanged. A scheduled job names an explicit tz-database `time_zone`; recreate the Maritime trigger in that timezone. The Functions docs do not commit to a timezone for trigger expressions, so treat those as UTC and verify the first firing time against a known activation. For scheduler-loop workers the schedule lives in code; extract it and recreate it as a real trigger rather than porting the loop. App Platform instance sizes do not carry over: Maritime picks an applicable size at create, and per-agent `--ram`/`--cpu` overrides exist if one workload genuinely needs them.

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest cases:

- **The workload is an LLM-driven bot or assistant.** Recreate it on a Maritime framework template. Enumerate live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
- **The workload is arbitrary code (a worker, a job script, a function handler).** App Platform already deploys from the user's git repo, so the source of truth is the repo itself; point Maritime at the same one:
  ```bash
  maritime create <name> --repo https://github.com/<user>/<repo> --json
  ```
  If the component built with a native buildpack (no Dockerfile in the repo), add one. That is porting work, not a click; say so and budget an hour. For DO Functions, fetch the source with `doctl serverless functions get <fn> --code` (this does not work for functions deployed as a zip; take the source from the user's repo instead), wrap the handler in a small always-running entrypoint or HTTP server, and push to a repo the user controls.

  Dockerfile gotcha that will bite: never use a shell-string CMD like `CMD ["sh", "-c", "python main.py $PORT"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly. Install `ca-certificates` in slim images.

**Env vars and secrets.** Each component's `envs:` block in the app spec carries plain values inline, but anything typed `SECRET` comes back encrypted from `doctl apps spec get`, never as plaintext; DigitalOcean encrypts secret values on first submission and does not return the originals through the API. Those values must come from the user's own records (password manager, original .env file, the upstream service that issued them). Say that plainly up front instead of discovering it at import time. DO Functions env vars ARE retrievable: `doctl serverless functions get <fn> --save-env <file>` (or `-E`) writes them as key=value lines. Route everything into a local `.env` file, import, then destroy the file:

```bash
maritime env import <agent> ./do.env --reload --json
rm ./do.env
```

Env changes apply on next boot unless you pass `--reload`.

**Schedules.** Recreate each cron as a Maritime trigger: dashboard, agent page, Triggers pane, cron type, using the expression from `doctl serverless triggers list` or the one extracted from the worker's code. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. For bring-your-own-code agents remember that timers inside a sleeping VM do not fire; either serve `GET /schedules`, use the SDK scheduler observer, or mark the agent `--always-on` at create time.

## Step 4: Verify before touching DigitalOcean

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For scheduled workloads, wait for one real scheduled firing and check its output against a known-good run from App Platform runtime logs or `doctl serverless activations logs --function <fn>`. If the workload writes somewhere observable (a database row, a Slack message, a file in Spaces), check the artifact itself, not just the logs. Both sides can safely run in parallel; that is the point of leaving DigitalOcean untouched.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on DigitalOcean, so they run them with their own credentials. You never run them.

1. Back up every app spec before touching it: `doctl apps spec get <app-id> > backup-<app>.yaml`.
2. Pause, reversibly. App Platform: edit the spec to remove the migrated worker or scheduled-job component and apply with `doctl apps update <app-id> --spec edited.yaml`; applying `backup-<app>.yaml` the same way reverses it. Removing the component is the pause: `instance_count` has a documented minimum of 1, so scaling to zero is not an option. Functions: `doctl serverless triggers disable <trigger>` (enable to reverse). Droplets: `doctl compute droplet-action power-off <droplet-id>` (power-on to reverse), always before any destroy.
3. Watch Maritime for a few days: `maritime logs <agent> --json`.
4. Only when satisfied, delete: `doctl apps delete <app-id>` (or keep the app and apply the trimmed spec permanently), `doctl compute droplet delete <droplet-id>`, and undeploy retired functions with `doctl serverless undeploy <function>`.
5. Revoke the migration token at cloud.digitalocean.com/account/api/tokens.
6. If the migration ran inside a Maritime agent, delete `DIGITALOCEAN_ACCESS_TOKEN` from that agent in the dashboard env vars pane, then `maritime env reload <agent> --json`. The revoked token is dead either way; do not leave it sitting in config.

## What stays on DigitalOcean

Be upfront about this list rather than letting the user discover it:

- **Spaces buckets**: S3-compatible object storage does not move and does not need to. The Maritime agent keeps using the bucket over the S3 API with a scoped Spaces access key pair in its env vars.
- **Managed Databases**: keep them. Put the public connection string (`doctl databases connection <db-id>`) in the agent's env vars and have the user add the agent's traffic to the cluster's trusted sources (Network Access tab). A cluster whose trusted sources admit only VPC-internal resources will refuse Maritime's connections, and the private hostname only resolves inside the VPC; flag each one you find and let the user decide (a trusted-source entry for the agent, a proxy inside the VPC, or leave that workload on DigitalOcean).
- **Load Balancers**: stay, serving whatever web services remain on DigitalOcean.
- **Cloud Firewalls, reserved IPs, VPCs**: droplet plumbing. It retires with the droplets it serves, in the cutover list, not before.
- **DNS hosted at DO**: stays. Nothing on Maritime needs it unless the user points a domain at a public web agent, which is a separate task.

## Official DigitalOcean documentation

The exact pages a migrating user needs, each verified 2026-08-07:

- [doctl apps command reference](https://docs.digitalocean.com/reference/doctl/reference/apps/): every apps subcommand, including list, spec, update, delete, and logs.
- [App spec reference](https://docs.digitalocean.com/products/app-platform/reference/app-spec/): the full YAML schema for services, workers, jobs (all five kinds), functions, and envs.
- [How to manage jobs in App Platform](https://docs.digitalocean.com/products/app-platform/how-to/manage-jobs/): deploy-time versus scheduled (cron) jobs, the SCHEDULED spec template, and the 15-minute floor.
- [How to schedule Functions](https://docs.digitalocean.com/products/functions/how-to/schedule-functions/): scheduled triggers, cron syntax, and project.yml trigger blocks.
- [How to create a personal access token](https://docs.digitalocean.com/reference/api/create-personal-access-token/): the Read Only preset, custom scopes, and why scopes are immutable after creation.
- [API token scopes reference](https://docs.digitalocean.com/reference/api/scopes/): every scope, including the `api:read` alias the migration token needs.
- [doctl compute droplet-action reference](https://docs.digitalocean.com/reference/doctl/reference/compute/droplet-action/): power-off, power-on, and the rest of the droplet lifecycle verbs used at cutover.
- [Spaces S3 compatibility reference](https://docs.digitalocean.com/products/spaces/reference/s3-compatibility/): which S3 API operations Spaces supports for the agent's continued bucket access.
- [How to secure PostgreSQL managed databases](https://docs.digitalocean.com/products/databases/postgresql/how-to/secure/): trusted sources, the mechanism that must admit the agent's traffic (the MySQL and Valkey pages mirror it).

## Troubleshooting

- `doctl serverless ...` says serverless support is not installed: run `doctl serverless install` once, then `doctl serverless connect`.
- Account has several Functions namespaces: `doctl serverless connect <namespace-hint>` picks one; re-run connect to switch before listing another namespace's functions.
- Secret env vars come back encrypted (not plaintext) from `doctl apps spec get`: expected, not an error. Values come from the user's records.
- `doctl apps spec get` returns the active spec; a deployment in progress can differ. Pass `--deployment <id>` to read a specific deployment's spec.
- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.

The doctl syntax and DigitalOcean platform claims in this skill were verified against official DigitalOcean documentation on 2026-08-07, with one exception: the `doctl serverless triggers` subcommands are missing from the docs site and were verified against the doctl source instead. Nothing here was run against a live DigitalOcean account, and the maritime CLI side remains desk-checked against the current CLI. If a provider-side command errors on a flag, trust `doctl <cmd> --help` over this document and continue.
