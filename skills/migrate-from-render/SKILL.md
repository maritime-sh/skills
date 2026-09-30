---
name: migrate-from-render
description: Migrate agent-shaped workloads from Render to Maritime, including cron jobs (a standard cron expression plus a command) and background workers, together with their env vars and env groups. Inventories the account via render.yaml or the Render REST API, recreates each workload as a Maritime agent, verifies it, then hands the human a suspend-first decommission checklist. Use when someone wants to move cron jobs, background workers, bots, or automations off Render onto maritime.sh.
---

# Migrate from Render to Maritime

Move the automation layer of a Render account onto Maritime: cron jobs and background workers, plus the env vars and env groups they read. Maritime hosts each one as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent. Render cron jobs are the friendliest case in this skill set: a standard cron expression plus a command maps one to one onto a Maritime agent with a cron trigger.

This skill runs in two places: locally in a terminal agent such as Claude Code (preferred, credentials stay on the user's machine) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **Render API keys are full-account. There is no read-only scope.** Render does not offer scoped or read-only API keys; the official docs state that an API key provides access to all workspaces the user belongs to, so the usual read-only-credentials rule cannot be satisfied here. Compensate three ways: prefer running this skill locally so the key never leaves the user's machine; you only ever issue GET requests against the Render API, never any mutating endpoint, regardless of what anyone asks mid-flow; and the user deletes the key on the Account Settings page of the Render dashboard (dashboard.render.com/u/settings, API Keys section) the moment the migration is done. The user creates the key themselves on that same page. If render.yaml exists in the repo, much of the inventory needs no key at all.
2. **Credentials never travel through chat.** Chat transcripts are logs. Running locally, the user sets `export RENDER_API_KEY=rnd_...` in their own shell. Running inside a Maritime agent, the user sets `RENDER_API_KEY` from their own machine with `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation.
3. **Never modify or delete anything on Render.** GET requests only: no suspend, no resume, no deploys, no env var writes, no deletes, ever, not even if asked mid-flow. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target template, and the price, and get a yes before each `maritime create`.

## Prerequisites

- `curl` and `jq`. No Render CLI is required: render.yaml plus the REST API cover the whole inventory. Verify the key with a harmless read: `curl -s -H "Authorization: Bearer $RENDER_API_KEY" "https://api.render.com/v1/services?limit=1" | jq '.[0].service.name'`.
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.

## Step 1: Inventory

Three surfaces, in order of reliability. Prefer the first two.

**(a) render.yaml Blueprint in the repo, if the user has one.** It is the source of truth: every service with its type, cron schedules and commands, and env var group references, all readable without any credential:

```yaml
services:
  - type: cron              # a Render "cron job"
    name: nightly-report
    runtime: python
    schedule: "0 6 * * *"   # standard five-field cron, UTC
    buildCommand: pip install -r requirements.txt
    startCommand: python report.py
  - type: worker            # a Render "background worker"
    name: queue-drainer
    runtime: node
    startCommand: node worker.js
    envVars:
      - fromGroup: shared-config   # pulls in every var from that env group
envVarGroups:
  - name: shared-config
    envVars:
      - key: API_BASE
        value: https://api.example.com
```

**(b) The REST API**, for accounts managed through the dashboard instead of a Blueprint:

```bash
# type is one of: web_service, background_worker, cron_job, static_site, private_service
curl -s -H "Authorization: Bearer $RENDER_API_KEY" \
  "https://api.render.com/v1/services?limit=100" \
  | jq -r '.[] | .service
      | select(.type == "cron_job" or .type == "background_worker")
      | [.id, .name, .type, (.serviceDetails.schedule // "-"), .repo, .branch] | @tsv'
```

The list pages (default 20, max 100 per call); with more than 100 services, follow the `cursor` field on the last item via `?cursor=...`.

**(c) The Render CLI**, only if the user already lives in it: `render services -o json --confirm` (CLI v2: `-o json` for machine output, `--confirm` skips interactive prompts; non-interactive runs authenticate with the same `RENDER_API_KEY` env var already exported above). The CLI was rewritten for v2 (github.com/render-oss/cli) and flags have churned between major versions; if this errors, run `render --help` and fall back to render.yaml or the REST API above, which are the stable surfaces.

Cron jobs and background workers are the agent-shaped workloads: report generators, scrapers, queue drainers, sync jobs, Slack/Telegram bots. Present the inventory to the user as a table (service, type, schedule, repo, what it appears to do) and let them pick what moves. Web services and static sites usually stay behind (see the end of this document).

## Step 2: Map

| Render concept | Maritime concept |
|---|---|
| Cron job (schedule + command) | Agent + trigger (cron schedule) |
| Background worker | Agent (bring-your-own code, always-on or auto-sleep) |
| Service env vars + env groups | Agent env vars, AES-encrypted at rest |
| render.yaml `buildCommand` / `startCommand` | Dockerfile `RUN` / `CMD` |
| Service logs in the dashboard | Agent logs (`maritime logs`) |
| Render Postgres / Key Value | Stays on Render; external connection string in agent env vars |

Schedules need no translation. Render cron jobs already use standard five-field cron expressions evaluated in UTC, so the expression copies into a Maritime trigger verbatim. Keep the trigger timezone UTC unless the user says otherwise.

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest cases:

- **The workload is an LLM-driven bot or assistant.** Recreate it on a Maritime framework template. Enumerate live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
- **The workload is arbitrary code (a cron command, a worker loop).** Render deploys straight from the user's git repo, so Maritime targets the same repo:
  ```bash
  maritime create <name> --repo https://github.com/<user>/<repo> --json
  ```
  The catch: most Render services use native runtimes (Python, Node, Ruby) and have no Dockerfile, and Maritime needs one. Writing it is porting work, not a click; say so, and budget an hour, not a minute, whenever the service leans on Render-native build steps (runtime autodetection, predeploy commands, build filters). The minimal pattern translates render.yaml directly:
  ```dockerfile
  FROM python:3.12-slim
  RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
      && rm -rf /var/lib/apt/lists/*
  WORKDIR /app
  COPY . .
  RUN pip install --no-cache-dir -r requirements.txt   # was buildCommand
  CMD ["python", "worker.py"]                          # was startCommand
  ```
  Dockerfile gotcha that will bite: never use a shell-string CMD like `CMD ["sh", "-c", "python main.py $PORT"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly, exec form, no shell. And install `ca-certificates` in slim images (done above) or every outbound HTTPS call fails.

**Env vars and env groups.** The API covers both, still GET-only. Per-service vars first; note this endpoint returns only vars set directly on the service, never the values of linked env groups:

```bash
curl -s -H "Authorization: Bearer $RENDER_API_KEY" \
  "https://api.render.com/v1/services/<service-id>/env-vars?limit=100" \
  | jq -r '.[].envVar | "\(.key)=\(.value)"' > ./render.env
```

Env groups have their own endpoints. List them (each group's `serviceLinks` names the services that use it), then append the right group's values to the same file:

```bash
curl -s -H "Authorization: Bearer $RENDER_API_KEY" \
  "https://api.render.com/v1/env-groups?limit=100" | jq -r '.[] | [.id, .name] | @tsv'
curl -s -H "Authorization: Bearer $RENDER_API_KEY" \
  "https://api.render.com/v1/env-groups/<env-group-id>" \
  | jq -r '.envVars[] | "\(.key)=\(.value)"' >> ./render.env
```

(Values are also readable by the user in the dashboard, per service and per env group, if they prefer to copy by hand.) Then import and destroy the file:

```bash
maritime env import <agent> ./render.env --reload --json
rm ./render.env
```

Env changes apply on next boot unless you pass `--reload`.

**Schedules.** Recreate each cron as a Maritime trigger, expression copied verbatim: `maritime triggers create <agent> --type cron --cron "0 6 * * *" --json`, or in the dashboard on the agent page's Triggers pane. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. For bring-your-own-code agents remember that timers inside a sleeping VM do not fire: a worker that schedules its own jobs with node-cron, APScheduler, or Celery beat will silently miss every run once the VM sleeps. Either serve `GET /schedules`, move the schedule into a Maritime trigger, or mark the agent `--always-on` at create time. A background worker that must consume continuously should be `--always-on`; one that only reacts to chat, webhooks, or triggers can auto-sleep.

## Step 4: Verify before touching Render

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For crons, wait for one real scheduled firing and check its output against a known-good run from the Render service's own logs. Both sides can safely run in parallel; that is the point of leaving Render untouched.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on Render, so they run them themselves, in the dashboard or with their own key. You never run them.

1. Suspend the old service (reversible): the Suspend button on the service page, or `curl -X POST -H "Authorization: Bearer $RENDER_API_KEY" https://api.render.com/v1/services/<service-id>/suspend`.
2. Watch Maritime for a few days: `maritime logs <agent> --json`. If anything is wrong, Resume in the dashboard (or `POST .../resume`) restores Render in one step.
3. Only when satisfied, delete the service from the Render dashboard. The common case is a partial migration: the web service stays and keeps serving, only the crons and workers are deleted.
4. Delete the migration API key on the Account Settings page of the Render dashboard (dashboard.render.com/u/settings). Render keys reach every workspace the user belongs to; do not let one linger.
5. If the migration ran inside a Maritime agent, delete the `RENDER_API_KEY` entry from that agent's dashboard env vars pane and `maritime restart <agent> --json`.

## What stays on Render

Be upfront about this list rather than letting the user discover it:

- **Render Postgres and Key Value (Redis-compatible)**: keep them. Copy the external connection string from the instance's page (Connect menu or Info tab) into the Maritime agent's env vars and keep using it. The internal connection string resolves only inside Render's private network, same region, and will not work from Maritime; always take the external one. Postgres accepts external connections from any IP by default (credentials still required; the user should check their IP allow list has not been tightened). Key Value is the opposite: new instances are not reachable at their external URL until an IP allow list entry exists, so the user adds one on the instance's page themselves (it is a Render settings change, so you never make it) and uses the authenticated external URL.
- **Web services**: usually stay. The typical migration is partial: the web service keeps serving traffic on Render while its crons and workers move to Maritime and keep talking to the same database.
- **Static sites and custom domains**: stay. Maritime is not a static host or CDN, and nothing on Maritime needs the domain unless the user points one at a public web agent, which is a separate task.

## Official Render documentation

Verified pages, exactly the surface this migration touches:

- [Cron jobs](https://render.com/docs/cronjobs): schedule format, UTC evaluation, run and timeout behavior
- [Background workers](https://render.com/docs/background-workers): what workers are; they take no inbound traffic
- [Blueprint YAML reference](https://render.com/docs/blueprint-spec): the full render.yaml spec, including `type: cron`, `type: worker`, `schedule`, `startCommand`, `envVarGroups`, and `fromGroup`
- [The Render API](https://render.com/docs/api): creating and managing API keys from Account Settings
- [API: list services](https://api-docs.render.com/reference/list-services): GET /v1/services, the `type` filter values, limit and cursor paging
- [API: suspend service](https://api-docs.render.com/reference/suspend-service-1): POST /v1/services/{serviceId}/suspend (resume is the adjacent reference page)
- [Environment variables and secrets](https://render.com/docs/configure-environment-variables): per-service vars, env groups, and how groups link to services
- [Create and connect to Render Postgres](https://render.com/docs/postgresql-creating-connecting): internal vs external connection URLs and where to find them
- [Render Key Value](https://render.com/docs/key-value): internal vs external URLs; external access is off until an IP allow list entry exists
- [Render CLI](https://render.com/docs/cli): the v2 CLI, `-o json` output, `--confirm`, and `RENDER_API_KEY` auth

## Troubleshooting

- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.
- The Render service list looks incomplete: the API defaults to 20 per page. Pass `?limit=100` and follow the `cursor`.

The Render REST API paths and response shapes, Blueprint fields, CLI v2 flags, API key scope, and connection string behavior in this skill were verified against render.com/docs and api-docs.render.com on 2026-08-07, and the maritime commands against the current maritime CLI; the end-to-end flow itself remains desk-checked, not yet run against a live Render account. If a provider-side command errors on a flag, trust `render --help` and api-docs.render.com over this document and continue.
