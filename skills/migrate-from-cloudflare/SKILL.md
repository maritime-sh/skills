---
name: migrate-from-cloudflare
description: Migrate agent-shaped workloads from Cloudflare to Maritime, including Workers with cron triggers, Queues consumers, Durable Object alarms, and data held in KV, R2, and D1. Inventories the account with a read-only API token, recreates each workload as a Maritime agent, verifies it, then hands the human a decommission checklist. Use when someone wants to move Workers crons, bots, or scheduled automations off Cloudflare onto maritime.sh.
---

# Migrate from Cloudflare to Maritime

Move the automation layer of a Cloudflare account onto Maritime: Workers running on cron triggers, Queues consumers, Durable Object alarms, and the KV, R2, and D1 data behind them. Maritime hosts each one as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent.

This skill runs in two places: locally in a terminal agent such as Claude Code (preferred, credentials stay on the user's machine) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **Read-only Cloudflare API token only.** Ask the user to create a custom token at dash.cloudflare.com/profile/api-tokens (Create Token, then Custom Token) with only these permissions: Workers Scripts:Read, Workers KV Storage:Read, D1:Read, Account Settings:Read (add Queues:Read and Workers R2 Storage:Read only if those products are in use). If the user offers their Global API Key, refuse it outright: Global API Keys cannot be scoped and carry full account write access. The human creates the token in their own dashboard; you only ever receive the resulting token. Remind them to delete it when the migration is done. Both wrangler and the REST API read it from the environment:
   ```bash
   export CLOUDFLARE_API_TOKEN=<token>        # the user sets this themselves
   export CLOUDFLARE_ACCOUNT_ID=<account-id>  # shown in the dashboard sidebar
   ```
2. **Credentials never travel through chat.** Chat transcripts are logs. Running locally, the user exports `CLOUDFLARE_API_TOKEN` in their own shell. Running inside a Maritime agent, the user sets `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` from their own machine with `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation.
3. **Never modify or delete anything on Cloudflare.** No `wrangler delete`, no `wrangler deploy`, no trigger edits, no KV or R2 writes, ever, not even if asked mid-flow. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target template, and the price, and get a yes before each `maritime create`.

## Prerequisites

- Node 22+. Current wrangler (4.x) refuses to run on anything older. Run wrangler as `npx wrangler` so you always get the current version; verify auth with `npx wrangler whoami`.
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.
- `jq`. If R2 data will actually move (usually it should not, see Step 3), the `aws` CLI or `rclone`.

## Step 1: Inventory

Workers are account-scoped, so no region sweep is needed. Collect the scripts, their cron triggers, and the data stores behind them:

```bash
# Every Worker script in the account
curl -s "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/workers/scripts" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | jq -r '.result[].id'

# What is actually live for one Worker
npx wrangler deployments list --name <worker>

# Cron triggers. If the source is in git, read the [triggers] crons block in
# wrangler.toml (or wrangler.jsonc). For a deployed-only Worker, ask the API:
curl -s "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/workers/scripts/<worker>/schedules" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | jq '.result.schedules'

# Data stores the Workers read
npx wrangler kv namespace list
npx wrangler d1 list --json
npx wrangler r2 bucket list
npx wrangler queues list
```

Agent-shaped workloads are the ones that wake up, do a job, and go quiet: cron Workers, report generators, scrapers, webhook bots, sync jobs. While reading source, also record every Queues consumer (`queue()` handler) and Durable Object alarm (`alarm()` handler) you find; they need the rewrite treatment in Step 2. Present the inventory to the user as a table (worker, trigger, bindings, what it appears to do) and let them pick what moves.

**Pulling a Worker whose source is not in git.** Cloudflare's documented recovery path for dashboard-created Workers:

```bash
npx wrangler init --from-dash <worker-name>
```

This is a one-time download: it does not sync later dashboard edits, so treat the result as a snapshot and keep working from git. Expect the Worker script only; do not count on bundled static assets coming down with it. If your wrangler version rejects the flag (some wrangler 3.x builds deprecated it; current wrangler 4.x supports it), the alternative form is `npm create cloudflare@latest <dir> -- --type pre-existing --existing-script <worker-name>`; in general trust `npx wrangler init --help` over this document.

## Step 2: Map

| Cloudflare concept | Maritime concept |
|---|---|
| Worker with cron trigger | Agent + trigger (cron schedule) |
| Worker serving HTTP traffic | Public web agent (code ported to a container server) |
| Queues consumer | Rewrite: webhook trigger or scheduled poll (below) |
| Durable Object alarm | Rewrite: Maritime cron trigger or in-agent scheduler |
| Worker vars and secrets | Agent env vars, AES-encrypted at rest |
| KV namespace | Whatever store the ported code uses (file, Redis, Postgres) |
| R2 bucket | Stays on R2; the agent uses it over the S3 API |
| D1 database | SQLite file on the agent volume, or hosted Postgres |
| `wrangler tail` / dashboard logs | Agent logs (`maritime logs`) |

Schedule expressions copy over verbatim: Cloudflare cron triggers are five-field cron evaluated in UTC, so keep the Maritime trigger timezone UTC unless the user says otherwise. One exception: Cloudflare also accepts non-standard extensions (L, W, and # in the day fields). If a trigger uses them, translate it to plain five-field cron before copying.

**Queues consumers and Durable Object alarms are rewrites; say so plainly.** Both are Cloudflare-runtime constructs with no portable equivalent. A Queues consumer is invoked by the runtime with message batches; on Maritime either the producer changes (it posts to the agent's webhook trigger instead of the queue) or the agent polls the queue on a cron via the Queues pull API (the queue needs an HTTP pull consumer enabled first, and the pulling token needs Queues write scope as well as read, because a pull consumes messages). A Durable Object alarm is a per-object timer; a coarse fixed schedule maps to a Maritime cron trigger, but dynamic per-entity timers mean the ported code needs its own scheduler and almost certainly `--always-on`. Neither case is a migration of the trigger, it is a rewrite of the trigger model. Tell the user which case applies before quoting effort.

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest cases:

- **The workload is an LLM-driven bot or assistant.** Recreate it on a Maritime framework template. Enumerate live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
- **The workload is arbitrary Worker code.** This is a port, not a copy, and you should say so. Worker code runs on the Workers runtime, not Node: `env.KV.get()`, `env.DB.prepare()`, the `scheduled()` and `fetch()` handlers, and every binding must be replaced with direct calls (KV REST or a new store, R2 over the S3 API, D1 data via its export) inside a normal long-running process. Wrap it in a small entrypoint or HTTP server, add a Dockerfile, push to a GitHub repo the user controls, then:
  ```bash
  # Cron-shaped Worker: create, then add a Maritime cron trigger (below)
  maritime create <name> --repo https://github.com/<user>/<repo> --json

  # HTTP-serving Worker: it becomes a public web agent, and the code must
  # become a container-run server listening on the port
  maritime create <name> --repo https://github.com/<user>/<repo> --public --port 8080 --json
  ```
  Budget hours, not a click. Dockerfile gotcha that will bite: never use a shell-string CMD like `CMD ["sh", "-c", "python main.py $PORT"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly. Install `ca-certificates` in slim images.

**Env vars and secrets.** Plain vars are readable from `[vars]` in wrangler.toml or the dashboard settings pane. Worker secret values are write-only on Cloudflare: no API or wrangler command can read them back, so the user must re-supply each secret from wherever it originally came from (password manager, the upstream provider's console). Write everything to a local `.env` file, import, then destroy the file:

```bash
maritime env import <agent> ./cf-migration.env --reload --json
rm ./cf-migration.env
```

Env changes apply on next boot unless you pass `--reload`.

**Data.**

- **KV**: enumerate and bulk-read with the read-only token, then load into whatever store the ported code now uses. Wrangler v4 runs KV data commands against local simulation storage by default, so `--remote` is mandatory here; without it you list an empty local store:
  ```bash
  npx wrangler kv key list --namespace-id=<id> --remote > keys.json
  npx wrangler kv bulk get keys.json --namespace-id=<id> --remote   # open beta; accepts the key-list JSON as-is
  # or per key over REST:
  curl -s "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/storage/kv/namespaces/<ns-id>/values/<key>" \
    -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
  ```
- **R2**: the default is to not move it at all. R2 speaks the S3 API, so the Maritime agent keeps using the bucket directly with a scoped R2 API token in its env vars and endpoint `https://<account-id>.r2.cloudflarestorage.com`. If the user insists on moving the objects: `aws s3 sync s3://<bucket> <dest> --endpoint-url https://<account-id>.r2.cloudflarestorage.com` (the Access Key ID and Secret Access Key shown once at R2 API token creation are the AWS credentials), or rclone.
- **D1**: export the full database, then load the dump into a SQLite file on the agent volume or into a hosted Postgres (expect small SQLite-to-Postgres dialect edits):
  ```bash
  npx wrangler d1 export <db> --remote --output=./dump.sql
  ```

**Schedules.** Recreate each cron as a Maritime trigger: dashboard, agent page, Triggers pane, cron type, expression copied verbatim. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. For bring-your-own-code agents remember that timers inside a sleeping VM do not fire; either serve `GET /schedules`, use the SDK scheduler observer, or mark the agent `--always-on` at create time.

## Step 4: Verify before touching Cloudflare

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For scheduled workloads, wait for one real scheduled firing and check its output against a known-good run from `npx wrangler tail <worker>` or the Workers dashboard logs. Both sides can safely run in parallel; that is the point of leaving Cloudflare untouched.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on Cloudflare, so they run them with their own credentials. You never run them.

1. Pause the old schedule (reversible): remove the `[triggers]` crons block from wrangler.toml and `npx wrangler deploy`. Restoring the block and redeploying restores Cloudflare in one command. For a dashboard-only Worker, remove the cron trigger in the dash under the Worker's Settings, Triggers.
2. Watch Maritime for a few days: `maritime logs <agent> --json`.
3. Only when satisfied, delete the Worker: `npx wrangler delete --name <worker>`. Remove KV namespaces and D1 databases only once nothing else reads them. Do not delete R2 buckets the agent now uses.
4. Delete the migration API token at dash.cloudflare.com/profile/api-tokens.
5. If the migration ran inside a Maritime agent, remove the Cloudflare env vars from it: `maritime env remove <agent> CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID`.

## What stays on Cloudflare

Be upfront about this list rather than letting the user discover it:

- **DNS zones**: keep them. Cloudflare DNS is best-in-class and nothing about this migration requires touching it. Pointing a domain at a Maritime public web agent later is one record in the same zone, a separate task.
- **Pages static sites**: stay. Maritime hosts agents, not static sites.
- **R2 buckets**: stay. The agent reaches them over the S3 API with scoped credentials in its env vars; moving object storage is almost never worth it.
- **Turnstile and Zero Trust configuration**: stay. They are account-level products with no Maritime equivalent and no reason to move.
- **Queues**: the queue itself stays as long as anything still produces to it; only the consumer role moves, per the rewrite note in Step 2.

## Official Cloudflare documentation

The exact pages a migrating user needs, each verified current as of 2026-08-07:

- [Wrangler commands reference (index of every command)](https://developers.cloudflare.com/workers/wrangler/commands/)
- [Wrangler Workers commands: init --from-dash, deployments list, tail, delete](https://developers.cloudflare.com/workers/wrangler/commands/workers/)
- [Wrangler KV commands: namespace list, key list, bulk get](https://developers.cloudflare.com/workers/wrangler/commands/kv/)
- [Wrangler v3 to v4 migration guide, including the local-by-default --remote change](https://developers.cloudflare.com/workers/wrangler/migration/update-v3-to-v4/)
- [Cron Triggers: wrangler.toml/wrangler.jsonc syntax, UTC evaluation, extensions](https://developers.cloudflare.com/workers/configuration/cron-triggers/)
- [Create a scoped API token in the dashboard](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
- [API token permission names (Workers Scripts Read, Workers KV Storage Read, D1 Read, and the rest)](https://developers.cloudflare.com/fundamentals/api/reference/permissions/)
- [D1 import and export](https://developers.cloudflare.com/d1/best-practices/import-export-data/)
- [R2 S3 API compatibility and the account endpoint format](https://developers.cloudflare.com/r2/api/s3/api/)
- [Queues pull consumers (HTTP pull, ack, token scopes)](https://developers.cloudflare.com/queues/configuration/pull-consumers/)

## Troubleshooting

- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.
- Cloudflare API answers 403 or `authentication error` (code 10000): the token is missing a read permission. Edit the token at dash.cloudflare.com/profile/api-tokens and add the scope; never fall back to the Global API Key.

Every wrangler command and flag in this skill is verified against live `wrangler 4.120.0` help output, and the REST API paths, token permission names, R2 endpoint format, and cron trigger syntax against official Cloudflare documentation, all on 2026-08-07; the maritime side is checked against the current CLI source. Nothing here was run against a live Cloudflare account, so behavior that only shows up with real credentials remains desk-checked. If a provider-side command errors on a flag, trust `npx wrangler <cmd> --help` over this document and continue.
