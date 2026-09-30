---
name: migrate-from-aws
description: Migrate agent-shaped workloads from AWS to Maritime, including scheduled Lambdas, EventBridge crons, ECS scheduled tasks, and EC2-hosted bots. Inventories the AWS account with read-only credentials, recreates each workload as a Maritime agent, verifies it, then hands the human a decommission checklist. Use when someone wants to move automations, bots, or cron jobs off AWS onto maritime.sh.
---

# Migrate from AWS to Maritime

Move the automation layer of an AWS account onto Maritime: scheduled Lambda functions, EventBridge cron rules, ECS scheduled tasks, and bots or workers squatting on EC2 instances. Maritime hosts each one as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent.

This skill runs in two places: locally in a terminal agent such as Claude Code (preferred, credentials stay on the user's machine) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **Read-only AWS credentials only.** Ask the user to mint a dedicated IAM user or role with the AWS-managed `ReadOnlyAccess` policy. Do not substitute the job-function `ViewOnlyAccess` policy: it is list-only, with no Secrets Manager, no EventBridge Scheduler, and no `lambda:Get*` actions, so half the inventory below fails under it. If the user offers root or admin keys, refuse them and show the read-only setup instead:
   ```bash
   aws iam create-user --user-name maritime-migration-readonly
   aws iam attach-user-policy --user-name maritime-migration-readonly \
     --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
   aws iam create-access-key --user-name maritime-migration-readonly
   ```
   The human runs those three with their own credentials; you only ever receive the resulting read-only key pair. Remind them to delete this user when the migration is done.
2. **Credentials never travel through chat.** Chat transcripts are logs. Running locally, the user configures a profile themselves (`aws configure --profile maritime-migration`). Running inside a Maritime agent, the user sets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION` from their own machine with `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation.
3. **Never modify or delete anything on AWS.** No `delete-function`, no `disable-rule`, no `terminate-instances`, ever, not even if asked mid-flow. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target template, and the price, and get a yes before each `maritime create`.

## Prerequisites

- `aws` CLI v2, authenticated with the read-only credentials above. Verify with `aws sts get-caller-identity`.
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.
- `jq`.

## Step 1: Inventory

Sweep every region the user cares about (`--region` on each call, or loop over `aws ec2 describe-regions`). Collect four kinds of workload:

```bash
# Scheduled EventBridge rules and what they trigger
aws events list-rules --query 'Rules[?ScheduleExpression!=null].[Name,ScheduleExpression,State]' --output table
aws events list-targets-by-rule --rule <rule-name>

# The newer EventBridge Scheduler, if in use. list-schedules omits the expression;
# get-schedule returns ScheduleExpression and ScheduleExpressionTimezone
aws scheduler list-schedules --query 'Schedules[].[Name,GroupName,State]' --output table
aws scheduler get-schedule --name <schedule-name>

# All Lambda functions (cross-reference with rule targets to find the scheduled ones)
aws lambda list-functions --query 'Functions[].[FunctionName,Runtime,Handler,MemorySize]' --output table

# ECS scheduled tasks show up as EventBridge targets with an ecs parameters block; also:
aws ecs list-clusters && aws ecs list-task-definitions

# Long-running bots on EC2
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value|[0],InstanceType]' --output table

# Secrets and parameters the workloads read (names only for now)
aws secretsmanager list-secrets --query 'SecretList[].Name' --output table
aws ssm describe-parameters --query 'Parameters[].Name' --output table
```

Agent-shaped workloads are the ones that wake up, do a job, and go quiet: cron Lambdas, report generators, scrapers, Slack/Telegram bots, queue drainers, sync jobs. Present the inventory to the user as a table (workload, trigger, runtime, what it appears to do) and let them pick what moves.

## Step 2: Map

| AWS concept | Maritime concept |
|---|---|
| Scheduled Lambda / EventBridge cron rule | Agent + trigger (cron schedule) |
| ECS scheduled task | Agent + trigger |
| EC2-hosted bot or worker | Agent (bring-your-own code, always-on or auto-sleep) |
| Lambda env vars, Secrets Manager, SSM params | Agent env vars, AES-encrypted at rest |
| CloudWatch Logs | Agent logs (`maritime logs`) |
| IAM role attached to the workload | Scoped keys in env vars for whatever the agent still calls |

Schedule expressions translate mechanically, with two traps. EventBridge `rate(5 minutes)` becomes `*/5 * * * *`. EventBridge cron has six fields with `?` placeholders; standard cron has five: `cron(0 12 * * ? *)` becomes `0 12 * * *`. Trap one: numeric day-of-week differs. EventBridge counts 1-7 starting at Sunday, standard cron counts 0-6 starting at Sunday, so EventBridge `2-6` (Monday to Friday) becomes `1-5`; convert day names to numbers too, since name ranges are not portable across cron implementations. Trap two: the `L`, `W`, and `#` wildcards have no five-field equivalent; re-derive those schedules by hand. Classic EventBridge rule crons always run in UTC, so keep the trigger timezone UTC unless the user says otherwise. EventBridge Scheduler schedules carry their own `ScheduleExpressionTimezone`; mirror it on the Maritime trigger.

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest cases:

- **The workload is an LLM-driven bot or assistant.** Recreate it on a Maritime framework template. Enumerate live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
- **The workload is arbitrary code (a Lambda handler, a container, a script).** This is a port, not a copy, and you should say so. Budget an hour, not a click. Fetch the code read-only:
  ```bash
  aws lambda get-function --function-name <fn> --query 'Code.Location' --output text
  # returns a presigned URL valid for 10 minutes; curl it to get the deployment zip.
  # Container-image functions have no zip: Code shows an ImageUri instead, so
  # source comes from the user's repo or registry.
  ```
  For ECS, `aws ecs describe-task-definition` gives the image ref; the source should come from the user's own repo. Wrap the handler in a small always-running entrypoint or HTTP server, add a Dockerfile, push to a GitHub repo the user controls, then:
  ```bash
  maritime create <name> --repo https://github.com/<user>/<repo> --json
  ```
  Dockerfile gotcha that will bite: never use a shell-string CMD like `CMD ["sh", "-c", "python main.py $PORT"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly. Install `ca-certificates` in slim images.

**Env vars and secrets.** Pull only the specific secrets the chosen workload reads (this is the one place read scope on secret values is needed):

```bash
aws lambda get-function-configuration --function-name <fn> --query 'Environment.Variables'
aws secretsmanager get-secret-value --secret-id <name> --query 'SecretString' --output text
```

`ReadOnlyAccess` deliberately stops at secret metadata: it does not grant `secretsmanager:GetSecretValue`, so expect `AccessDenied` on that second command. Two clean fixes, both run by the human: attach the AWS-managed `AWSSecretsManagerClientReadOnlyAccess` policy to the migration user for the duration (it adds `GetSecretValue` plus the scoped `kms:Decrypt`), or have the human run the `get-secret-value` calls themselves and put the values straight into the local `.env` file below.

Write them to a local `.env` file, import, then destroy the file:

```bash
maritime env import <agent> ./aws-migration.env --reload --json
rm ./aws-migration.env
```

Env changes apply on next boot unless you pass `--reload`.

**Schedules.** Recreate each cron as a Maritime trigger with the converted expression: `maritime triggers create <agent> --type cron --cron "0 12 * * *" --json`, or in the dashboard: agent page, Triggers pane, cron type. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. For bring-your-own-code agents remember that timers inside a sleeping VM do not fire; either serve `GET /schedules`, use the SDK scheduler observer, or mark the agent `--always-on` at create time.

## Step 4: Verify before touching AWS

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For scheduled workloads, wait for one real scheduled firing and check its output against a known-good run from CloudWatch Logs. Both sides can safely run in parallel; that is the point of leaving AWS untouched.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on AWS, so they run them with their own admin credentials. You never run them.

1. Pause the old schedule (reversible): `aws events disable-rule --name <rule>`
2. Watch Maritime for a few days: `maritime logs <agent> --json`. If anything is wrong, `aws events enable-rule --name <rule>` restores AWS in one command.
3. Only when satisfied, delete. A rule with targets refuses to die: first `aws events remove-targets --rule <rule> --ids <target-id>` (ids come from `aws events list-targets-by-rule --rule <rule>`), then `aws events delete-rule --name <rule>` (`--force` is only for rules managed by another AWS service). Then `aws lambda delete-function --function-name <fn>`, terminate the EC2 instance, and remove unused Secrets Manager entries.
4. Delete the migration IAM user made in the safety section: `aws iam delete-access-key --user-name maritime-migration-readonly --access-key-id <id>`, `aws iam detach-user-policy --user-name maritime-migration-readonly --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess` (repeat for `AWSSecretsManagerClientReadOnlyAccess` if it was attached), then `aws iam delete-user --user-name maritime-migration-readonly`.
5. If the migration ran inside a Maritime agent, remove the AWS env vars from it: `maritime env remove <agent> AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

## What stays on AWS

Be upfront about this list rather than letting the user discover it:

- **S3, DynamoDB, RDS, ElastiCache**: data services do not move. The Maritime agent keeps calling them over the public endpoint with scoped credentials in its env vars. VPC-only resources (an RDS with no public access, anything behind a private subnet) are unreachable from Maritime; flag each one you find and let the user decide (public endpoint with strict security group, a proxy, or leave that workload on AWS).
- **Route 53 / DNS**: stays. Nothing on Maritime needs it unless the user points a domain at a public web agent, which is a separate task.
- **SQS/SNS-driven workloads**: Maritime triggers are cron and webhook shaped. An SQS consumer can move only if the producer can hit a webhook instead, or the agent polls the queue on a schedule. Say which one applies.

## Official AWS documentation

The exact pages behind the claims in this skill, all current as of 2026-08-07:

- [AWS CLI reference: events list-rules](https://docs.aws.amazon.com/cli/latest/reference/events/list-rules.html) (rule fields: Name, ScheduleExpression, State)
- [AWS CLI reference: events delete-rule](https://docs.aws.amazon.com/cli/latest/reference/events/delete-rule.html) (targets must be removed first; what `--force` really does)
- [AWS CLI reference: lambda get-function](https://docs.aws.amazon.com/cli/latest/reference/lambda/get-function.html)
- [Lambda GetFunction API: the presigned Code.Location URL, valid 10 minutes](https://docs.aws.amazon.com/lambda/latest/api/API_GetFunction.html)
- [AWS CLI reference: secretsmanager get-secret-value](https://docs.aws.amazon.com/cli/latest/reference/secretsmanager/get-secret-value.html) (required permissions listed at the top)
- [EventBridge rule schedule expressions: rate and six-field cron, UTC](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-scheduled-rule-pattern.html)
- [EventBridge Scheduler schedule types and timezones](https://docs.aws.amazon.com/scheduler/latest/UserGuide/schedule-types.html)
- [ReadOnlyAccess managed policy JSON](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/ReadOnlyAccess.html)
- [AWS managed policies for Secrets Manager, including AWSSecretsManagerClientReadOnlyAccess](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-policies.html)
- [Manage access keys for IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)

## Troubleshooting

- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.

Every aws command, flag, and --query expression in this skill was verified on 2026-08-07 against AWS CLI v2.27 offline help and the official documentation linked above, and every maritime command against the CLI source. A live end-to-end run against a real AWS account remains desk-checked only. If a provider-side command errors on a flag, trust `aws <service> <verb> help` over this document and continue.
