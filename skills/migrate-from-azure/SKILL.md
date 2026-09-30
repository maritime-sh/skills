---
name: migrate-from-azure
description: Migrate agent-shaped workloads from Microsoft Azure to Maritime, including Azure Functions timer triggers, Container Apps scheduled jobs, VM-hosted bots, and Logic Apps automations. Inventories the subscription with a read-only service principal, recreates each workload as a Maritime agent, verifies it, then hands the human a decommission checklist. Use when someone wants to move automations, bots, or cron jobs off Azure onto maritime.sh.
---

# Migrate from Azure to Maritime

Move the automation layer of an Azure subscription onto Maritime: Azure Functions on timer triggers, Container Apps jobs on a cron schedule, bots or workers squatting on VMs, and Logic Apps running scheduled automations. Maritime hosts each one as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent.

This skill runs in two places: locally in a terminal agent such as Claude Code (preferred, credentials stay on the user's machine) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **Read-only Azure credentials only.** Ask the user to mint a dedicated service principal with the built-in `Reader` role at the subscription scope. If the user offers Owner or Contributor credentials, refuse them and show the read-only setup instead:
   ```bash
   az ad sp create-for-rbac --display-name maritime-migration-readonly \
     --role Reader --scopes /subscriptions/<subscription-id>
   # Only if Key Vault secret values must be read (see Step 3):
   az role assignment create --assignee <appId> --role "Key Vault Secrets User" \
     --scope $(az keyvault show --name <vault> --query id -o tsv)
   ```
   The human runs those with their own credentials; you only ever receive the resulting appId, password, and tenant. Remind them to delete this service principal when the migration is done.
2. **Credentials never travel through chat.** Chat transcripts are logs. Running locally, the user signs in themselves (`az login --service-principal -u <appId> -p <password> --tenant <tenant>`). Running inside a Maritime agent, the user sets `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, and `AZURE_TENANT_ID` from their own machine with `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation.
3. **Never modify or delete anything on Azure.** No `az functionapp stop`, no `az vm deallocate`, no `delete` of any kind, ever, not even if asked mid-flow. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target template, and the price, and get a yes before each `maritime create`.

## Prerequisites

- `az` CLI 2.55 or newer, signed in as the read-only service principal above. Verify with `az account show`. The `logic` command group is an extension that az installs automatically on first use (2.55 is its documented minimum, which is where the version floor comes from). The `containerapp` job commands ship in the core CLI on current versions; only older installs need the `containerapp` extension, which also auto-installs.
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.
- `jq`.

## Step 1: Inventory

Azure resources are subscription-wide but often spread across resource groups; the list commands below sweep the whole subscription. Collect four kinds of workload:

```bash
# Function apps, then the functions inside each one
az functionapp list --query "[].{name:name, rg:resourceGroup, state:state}" -o table
az functionapp function list -g <rg> -n <app> -o table
az functionapp function show -g <rg> -n <app> --function-name <fn> --query "config.bindings"
# Timer triggers have a binding of type "timerTrigger"; the "schedule" value is NCRONTAB,
# whether it came from function.json or a code attribute. A value like "%MySchedule%"
# resolves through app settings:
az functionapp config appsettings list -g <rg> -n <app> -o table

# Container Apps jobs with a Schedule trigger
az containerapp job list --query "[].{name:name, rg:resourceGroup, trigger:properties.configuration.triggerType, cron:properties.configuration.scheduleTriggerConfig.cronExpression}" -o table

# Long-running bots on VMs
az vm list -d --query "[].{name:name, rg:resourceGroup, power:powerState, size:hardwareProfile.vmSize}" -o table

# Logic Apps doing scheduled automations (Consumption tier; Standard-tier Logic Apps
# run on the Functions runtime and already appear in the functionapp list above)
az logic workflow list --query "[].{name:name, rg:resourceGroup, state:properties.state}" -o table
az logic workflow show -g <rg> -n <wf> --query "properties.definition.triggers"

# Key Vault secrets the workloads read (names only for now)
az keyvault list --query "[].name" -o tsv
az keyvault secret list --vault-name <vault> --query "[].name" -o tsv
```

Agent-shaped workloads are the ones that wake up, do a job, and go quiet: timer functions, report generators, scrapers, Slack/Telegram bots, queue drainers, sync jobs. Present the inventory to the user as a table (workload, trigger, runtime, what it appears to do) and let them pick what moves.

## Step 2: Map

| Azure concept | Maritime concept |
|---|---|
| Function app timer trigger | Agent + trigger (cron schedule) |
| Container Apps scheduled job | Agent + trigger |
| VM-hosted bot or worker | Agent (bring-your-own code, always-on or auto-sleep) |
| Logic App recurrence workflow | Agent + trigger (rebuild connector steps as agent instructions or code) |
| App settings, Key Vault secrets | Agent env vars, AES-encrypted at rest |
| Application Insights / Log Analytics | Agent logs (`maritime logs`) |
| Managed identity on the workload | Scoped keys in env vars for whatever the agent still calls |

Azure Functions timer schedules are NCRONTAB, canonically SIX fields with seconds first: `{second} {minute} {hour} {day} {month} {day-of-week}`. Maritime triggers use standard five-field cron, so drop the leading seconds field when it is `0`: `0 0 12 * * *` becomes `0 12 * * *`, and `0 */5 * * * *` becomes `*/5 * * * *`. Count the fields before converting: NCRONTAB also accepts plain five-field expressions, so a schedule that already has five fields is already standard cron and must be copied unchanged, never stripped. A schedule that actually uses the seconds field (`*/30 * * * * *`, every 30 seconds) fires sub-minute and cannot be expressed in five-field cron; flag it and let the user pick a minutely compromise. NCRONTAB runs in UTC unless the app sets `WEBSITE_TIME_ZONE`, so keep the trigger timezone UTC unless that setting exists. Container Apps job cron expressions are already five fields; copy them verbatim.

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest cases:

- **The workload is an LLM-driven bot or assistant.** Recreate it on a Maritime framework template. Enumerate live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
- **The workload is arbitrary code (a function handler, a container, a script).** This is a port, not a copy, and you should say so. Budget an hour, not a click. The Reader role cannot download deployed function code (Kudu zip and publish-profile downloads are write-scoped operations), so the source of truth is the user's own repo; ask for it. For a Container Apps job, `az containerapp job show -g <rg> -n <job> --query "properties.template.containers[].image"` names the image, but the source should still come from the user's repo. Wrap the handler in a small always-running entrypoint or HTTP server, add a Dockerfile, push to a GitHub repo the user controls, then:
  ```bash
  ID=$(maritime create <name> --json | jq -r '.id')
  maritime deploy "$ID" --source github --repo https://github.com/<user>/<repo> --branch main --json
  ```
  Dockerfile gotcha that will bite: never use a shell-string CMD like `CMD ["sh", "-c", "python main.py $PORT"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly. Install `ca-certificates` in slim images.

**Env vars and secrets.** Function app settings are the workload's env vars; settings whose value looks like `@Microsoft.KeyVault(SecretUri=...)` resolve through Key Vault:

```bash
az functionapp config appsettings list -g <rg> -n <app> -o json
az keyvault secret show --vault-name <vault> --name <secret> --query value -o tsv
```

Reading a secret's value needs the `Key Vault Secrets User` role from the safety section (or a get/list access policy on legacy non-RBAC vaults); the subscription `Reader` role only lists names. Pull only the specific secrets the chosen workload reads.

Write them to a local `.env` file, import, then destroy the file:

```bash
maritime env import <agent> ./azure-migration.env --reload --json
rm ./azure-migration.env
```

Env changes apply on next boot unless you pass `--reload`.

**Schedules.** Recreate each cron as a Maritime trigger: dashboard, agent page, Triggers pane, cron type, using the converted expression. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. For bring-your-own-code agents remember that timers inside a sleeping VM do not fire; either serve `GET /schedules`, use the SDK scheduler observer, or mark the agent `--always-on` at create time.

## Step 4: Verify before touching Azure

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For scheduled workloads, wait for one real scheduled firing and check its output against a known-good run from Application Insights or the function's Monitor tab. Both sides can safely run in parallel; that is the point of leaving Azure untouched.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on Azure, so they run them with their own admin credentials. You never run them.

1. Pause the old workload (reversible): `az functionapp stop -g <rg> -n <app>`. For a Logic App: `az logic workflow update -g <rg> -n <wf> --state Disabled`.
2. Watch Maritime for a few days: `maritime logs <agent> --json`. If anything is wrong, `az functionapp start` (or `--state Enabled`) restores Azure in one command.
3. Only when satisfied, delete: `az functionapp delete -g <rg> -n <app>`, `az containerapp job delete -g <rg> -n <job> --yes`, then for VMs `az vm deallocate -g <rg> -n <vm>` followed by `az vm delete -g <rg> -n <vm> --yes`, and remove unused Key Vault secrets.
4. Remove the migration service principal's role assignments, then delete it: `az role assignment delete --assignee <appId>`, then `az ad sp delete --id <appId>`. Deleting the principal alone leaves orphaned assignments that show up later as "Identity not found".
5. If the migration ran inside a Maritime agent, clear `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, and `AZURE_TENANT_ID` from that agent in the dashboard env vars pane.

## What stays on Azure

Be upfront about this list rather than letting the user discover it:

- **Blob Storage, Azure SQL, Cosmos DB**: data services do not move. The Maritime agent keeps calling them over the public endpoint with scoped credentials (connection strings, SAS tokens) in its env vars. Anything VNet-only, including every private endpoint you find in the inventory, is unreachable from Maritime; flag each one and let the user decide (public endpoint with a strict firewall rule, a proxy, or leave that workload on Azure).
- **Azure DNS**: stays. Nothing on Maritime needs it unless the user points a domain at a public web agent, which is a separate task.
- **Service Bus-driven workloads**: Maritime triggers are cron and webhook shaped. A queue or topic consumer can move only if the producer can hit a webhook instead, or the agent polls on a schedule. Say which one applies.

## Official Azure documentation

The exact pages a migrating user needs, all on learn.microsoft.com:

- [az functionapp command reference](https://learn.microsoft.com/en-us/cli/azure/functionapp) covers `list`, `stop`, `start`, `delete`, and the subgroups used here.
- [Timer trigger for Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer) is the NCRONTAB reference: field order, examples, UTC default, and `WEBSITE_TIME_ZONE`.
- [Jobs in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/jobs) documents trigger types, the five-field UTC cron format, and the `scheduleTriggerConfig` ARM properties.
- [az containerapp job command reference](https://learn.microsoft.com/en-us/cli/azure/containerapp/job) covers `list`, `show`, and `delete`.
- [az logic workflow command reference](https://learn.microsoft.com/en-us/cli/azure/logic/workflow) covers `list`, `show`, and `update --state`, and carries the extension note.
- [az keyvault secret command reference](https://learn.microsoft.com/en-us/cli/azure/keyvault/secret) covers `list` and `show`.
- [Azure RBAC for Key Vault data plane](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide) defines Key Vault Secrets User and the RBAC-vs-access-policy permission models.
- [az ad sp command reference](https://learn.microsoft.com/en-us/cli/azure/ad/sp) covers `create-for-rbac` and `delete`.
- [Sign in with a service principal](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli-service-principal) is the `az login --service-principal` reference.
- [Azure built-in roles: General](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/general) is the Reader role definition (`*/read`, control plane only, no data actions).

## Troubleshooting

- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.
- `az` says a `logic` subcommand does not exist: `logic` is an extension that normally auto-installs; `az extension add --name logic` (or `--upgrade`) fixes it. If a `containerapp` subcommand is missing you are on an old CLI; `az extension add --name containerapp` covers it there (current CLI versions have the job commands in core).
- A `--query` path in this skill returns nulls: inspect one resource with the group's `show` command and adapt the path.

Every `az` command, flag, RBAC role claim, and cron-format claim in this skill was verified against the official documentation on learn.microsoft.com on 2026-08-07 (the links above are the pages used). Two things remain desk-checked rather than doc-verified: the exact `--query` paths into command output (the reference pages do not document output shapes; the troubleshooting bullet above covers drift) and the note that Standard-tier Logic Apps appear in the functionapp list. The maritime CLI surface is desk-checked against the current CLI. If a provider-side command errors on a flag, trust `az <group> --help` over this document and continue.
