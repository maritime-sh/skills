---
name: migrate-from-gcp
description: Migrate agent-shaped workloads from Google Cloud Platform to Maritime, including Cloud Scheduler cron jobs, Cloud Functions (gen1 and gen2), Cloud Run jobs and worker services, and Compute Engine-hosted bots. Inventories the GCP project with a read-only service account, recreates each workload as a Maritime agent, verifies it, then hands the human a decommission checklist. Use when someone wants to move automations, bots, or cron jobs off Google Cloud onto maritime.sh.
---

# Migrate from Google Cloud to Maritime

Move the automation layer of a GCP project onto Maritime: Cloud Scheduler cron jobs, scheduled Cloud Functions (gen1 and gen2), Cloud Run jobs, Cloud Run services acting as workers, and bots or workers squatting on Compute Engine instances. Maritime hosts each one as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent.

This skill runs in two places: locally in a terminal agent such as Claude Code (preferred, credentials stay on the user's machine) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **Read-only GCP credentials only.** Ask the user to mint a dedicated service account with `roles/viewer`. If the user offers an owner or editor key, refuse it and show the read-only setup instead:
   ```bash
   gcloud iam service-accounts create maritime-migration-readonly \
     --display-name="Maritime migration (read-only)"
   gcloud projects add-iam-policy-binding <PROJECT_ID> \
     --member="serviceAccount:maritime-migration-readonly@<PROJECT_ID>.iam.gserviceaccount.com" \
     --role="roles/viewer"
   gcloud iam service-accounts keys create maritime-migration-key.json \
     --iam-account=maritime-migration-readonly@<PROJECT_ID>.iam.gserviceaccount.com
   ```
   The human runs those three with their own credentials; you only ever receive the resulting key file. `roles/viewer` lists secrets but cannot read their values; when step 3 needs a value, the human additionally grants `roles/secretmanager.secretAccessor` on just the secrets the chosen workload reads:
   ```bash
   gcloud secrets add-iam-policy-binding <secret-name> \
     --member="serviceAccount:maritime-migration-readonly@<PROJECT_ID>.iam.gserviceaccount.com" \
     --role="roles/secretmanager.secretAccessor"
   ```
   Remind them to delete this service account when the migration is done.
2. **Credentials never travel through chat.** Chat transcripts are logs. Running locally, the user activates the key themselves (`gcloud auth activate-service-account --key-file=maritime-migration-key.json`). Running inside a Maritime agent, the user sets `GCP_SA_KEY` (the key file contents) and `GCP_PROJECT_ID` from their own machine with `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation; the agent writes the key to a file and activates it the same way.
3. **Never modify or delete anything on GCP.** No `functions delete`, no `scheduler jobs pause`, no `instances stop`, ever, not even if asked mid-flow. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target template, and the price, and get a yes before each `maritime create`.

## Prerequisites

- `gcloud` CLI, authenticated as the read-only service account above, with the project set (`gcloud config set project <PROJECT_ID>`). Verify with `gcloud auth list`.
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.
- `jq`.

## Step 1: Inventory

Sweep every project the user cares about (`gcloud projects list`, then `gcloud config set project` per pass). Collect four kinds of workload:

```bash
# Cloud Scheduler cron jobs (regional; loop over locations)
gcloud scheduler locations list --format="value(locationId)"
gcloud scheduler jobs list --location=<loc>
gcloud scheduler jobs describe <job> --location=<loc>   # schedule, timeZone, target

# Cloud Functions, gen1 and gen2 (the ENVIRONMENT column tells them apart;
# older gcloud versions lack it, so rerun as `gcloud functions list --v2`,
# which returns both generations in v2 format with an `environment` field)
gcloud functions list

# Cloud Run jobs, and services acting as background workers
gcloud run jobs list
gcloud run services list

# Long-running bots on Compute Engine
gcloud compute instances list --filter="status=RUNNING"

# Secrets the workloads read (names only for now; list never returns secret data)
gcloud secrets list
```

Cross-reference each Scheduler job's target (an HTTP URL, a Pub/Sub topic, a Cloud Run jobs `:run` call against `run.googleapis.com`, or on legacy cron.yaml jobs an App Engine handler) against the function and Run lists to find which ones are actually scheduled. Agent-shaped workloads are the ones that wake up, do a job, and go quiet: cron functions, report generators, scrapers, Slack/Telegram bots, queue drainers, sync jobs. Present the inventory to the user as a table (workload, trigger, runtime, what it appears to do) and let them pick what moves.

## Step 2: Map

| GCP concept | Maritime concept |
|---|---|
| Cloud Scheduler job | Agent + trigger (cron schedule) |
| Scheduled Cloud Function (gen1 or gen2) | Agent + trigger |
| Cloud Run job | Agent + trigger |
| GCE-hosted bot or worker | Agent (bring-your-own code, always-on or auto-sleep) |
| Function/Run env vars, Secret Manager | Agent env vars, AES-encrypted at rest |
| Cloud Logging | Agent logs (`maritime logs`) |
| Service account attached to the workload | Scoped keys in env vars for whatever the agent still calls |

Schedule conversion is nearly 1:1. Cloud Scheduler already speaks five-field unix-cron, with the timezone stored separately in the job's `timeZone` field: copy the `schedule` expression as-is and set the Maritime trigger's timezone to the job's `timeZone`. The one thing to watch is legacy App Engine syntax (`every 5 minutes`) on old jobs; rewrite those as standard cron (`*/5 * * * *`).

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest cases:

- **The workload is an LLM-driven bot or assistant.** Recreate it on a Maritime framework template. Enumerate live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
- **The workload is arbitrary code (a function, a container, a script).** This is a port, not a copy, and you should say so. Budget an hour, not a click. The source should come from the user's own repo; failing that, the deployed artifact is readable:
  ```bash
  gcloud functions describe <fn> --region=<region> --format=json
  # gen1: sourceArchiveUrl is a gs:// path; gen2: buildConfig.source.storageSource
  gcloud storage cp gs://<bucket>/<object> ./source.zip
  gcloud run jobs describe <job> --region=<region> --format=json   # container image ref
  ```
  If the `cp` returns 403, that is expected: basic `roles/viewer` carries no `storage.objects.get`, so it can see the archive's path but not download it. The human grants the migration service account `roles/storage.objectViewer` on that one source bucket (still read-only), or downloads the file themselves.

  Wrap the entrypoint in a small always-running program or HTTP server, add a Dockerfile, push to a GitHub repo the user controls, then:
  ```bash
  ID=$(maritime create <name> --json | jq -r '.id')
  maritime deploy "$ID" --source github --repo https://github.com/<user>/<repo> --branch main --json
  ```
  Dockerfile gotcha that will bite: never use a shell-string CMD like `CMD ["sh", "-c", "python main.py $PORT"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly. Install `ca-certificates` in slim images.

**Env vars and secrets.** Pull only the specific secrets the chosen workload reads (this is the one place the `secretAccessor` grant from the safety section is needed):

```bash
gcloud functions describe <fn> --region=<region> --format=json
# env vars live at environmentVariables (gen1) or serviceConfig.environmentVariables (gen2)
gcloud secrets versions access latest --secret=<name>
```

Write them to a local `.env` file, import, then destroy the file:

```bash
maritime env import <agent> ./gcp-migration.env --reload --json
rm ./gcp-migration.env
```

Env changes apply on next boot unless you pass `--reload`.

**Schedules.** Recreate each cron as a Maritime trigger: dashboard, agent page, Triggers pane, cron type, using the copied expression and the job's timezone. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. For bring-your-own-code agents remember that timers inside a sleeping VM do not fire; either serve `GET /schedules`, use the SDK scheduler observer, or mark the agent `--always-on` at create time.

## Step 4: Verify before touching GCP

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For scheduled workloads, wait for one real scheduled firing and check its output against a known-good run from Cloud Logging. Both sides can safely run in parallel; that is the point of leaving GCP untouched.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on GCP, so they run them with their own admin credentials. You never run them.

1. Pause the old schedule (genuinely reversible): `gcloud scheduler jobs pause <job> --location=<loc>`
2. Watch Maritime for a few days: `maritime logs <agent> --json`. If anything is wrong, `gcloud scheduler jobs resume <job> --location=<loc>` restores GCP in one command.
3. Only when satisfied, delete: `gcloud scheduler jobs delete <job> --location=<loc>`, `gcloud functions delete <fn> --region=<region>`, `gcloud run jobs delete <job> --region=<region>`, delete the GCE instance, and remove unused Secret Manager entries.
4. Delete the migration service account made in the safety section: `gcloud projects remove-iam-policy-binding <PROJECT_ID> --member="serviceAccount:maritime-migration-readonly@<PROJECT_ID>.iam.gserviceaccount.com" --role="roles/viewer"`, the same per-secret `remove-iam-policy-binding` for any `secretAccessor` grants, then `gcloud iam service-accounts delete maritime-migration-readonly@<PROJECT_ID>.iam.gserviceaccount.com` (deleting the account kills its keys).
5. If the migration ran inside a Maritime agent, remove `GCP_SA_KEY` and `GCP_PROJECT_ID` from its dashboard env vars pane and delete the key file the agent wrote to disk.

## What stays on Google Cloud

Be upfront about this list rather than letting the user discover it:

- **GCS, Cloud SQL, BigQuery, Firestore**: data services do not move. The Maritime agent keeps calling them over the public endpoint with scoped credentials in its env vars. VPC-only resources are unreachable from Maritime: a Cloud SQL instance with only a private IP is the classic case; flag each one you find and let the user decide (public IP with strict authorized networks, a proxy, or leave that workload on GCP).
- **Cloud DNS**: stays. Nothing on Maritime needs it unless the user points a domain at a public web agent, which is a separate task.
- **Pub/Sub-driven workloads**: Maritime triggers are cron and webhook shaped. A Pub/Sub subscriber can move only if the topic can gain a push subscription aimed at a webhook, or the agent polls the subscription on a schedule. Say which one applies.

## Official Google Cloud documentation

The exact pages a migrating user needs, in the order this skill uses them:

- [gcloud scheduler jobs reference (list, describe, pause, resume, delete)](https://docs.cloud.google.com/sdk/gcloud/reference/scheduler/jobs)
- [Cloud Scheduler cron schedule format (five-field unix-cron, time zones, legacy cron.yaml)](https://docs.cloud.google.com/scheduler/docs/configuring/cron-job-schedules)
- [gcloud functions list (gen1 via v1 API, gen2 via v2 API, --v2 flag)](https://docs.cloud.google.com/sdk/gcloud/reference/functions/list)
- [gcloud functions describe (per-generation output format, --region)](https://docs.cloud.google.com/sdk/gcloud/reference/functions/describe)
- [gcloud run jobs reference (list, describe, delete, --region)](https://docs.cloud.google.com/sdk/gcloud/reference/run/jobs)
- [Execute Cloud Run jobs on a schedule (the Scheduler :run HTTP target)](https://docs.cloud.google.com/run/docs/execute/jobs-on-schedule)
- [Secret Manager access control (why roles/viewer cannot read secret values)](https://docs.cloud.google.com/secret-manager/docs/access-control)
- [gcloud secrets versions access (latest alias, --secret, --out-file)](https://docs.cloud.google.com/sdk/gcloud/reference/secrets/versions/access)
- [gcloud iam service-accounts keys create (key file for the read-only account)](https://docs.cloud.google.com/sdk/gcloud/reference/iam/service-accounts/keys/create)
- [gcloud storage cp (downloading a function's source archive)](https://docs.cloud.google.com/sdk/gcloud/reference/storage/cp)

## Troubleshooting

- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.

Every gcloud command, flag, role claim, and field path in this skill was verified against the official Google Cloud documentation linked above on 2026-08-07, and every maritime command against the current maritime CLI; the gcloud commands were not executed live from this machine, and cosmetic details of table output (column names such as ENVIRONMENT) remain desk-checked. If a provider-side command errors on a flag, trust `gcloud <group> --help` over this document and continue.
