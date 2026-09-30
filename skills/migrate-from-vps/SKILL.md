---
name: migrate-from-vps
description: Migrate agent-shaped workloads off any VPS or bare server reachable over SSH, including cron jobs, tmux or screen resident bots, pm2 and supervisor apps, systemd services and timers, and Docker containers, whether the box is a Hetzner or Linode or OVH server, a Raspberry Pi, or an old EC2 instance treated as a pet. Inventories the box over SSH with a fixed list of read-only commands, recreates each workload as a Maritime agent, verifies it, then hands the human a decommission checklist. Use when someone wants to move bots, cron jobs, or workers off a server they SSH into onto maritime.sh.
---

# Migrate from a VPS to Maritime

Move the automation layer of a VPS or bare server onto Maritime: crontab entries, systemd services and timers, pm2 and supervisor processes, bots living in tmux or screen sessions, and Docker containers. This is the classic "$5 droplet running my bot under tmux" migration, and it applies to any box you can SSH into: Hetzner, Linode, OVH, DigitalOcean, a Raspberry Pi, an old EC2 instance nobody dares touch. Maritime hosts each workload as its own agent with a schedule, encrypted env vars, and logs, for a flat monthly price per agent.

This skill runs in two places: locally in a terminal agent such as Claude Code (preferred, keys stay on the user's machine) or inside a Maritime agent. Either way the same rules apply.

## Safety rules, non-negotiable

1. **A regular user, and a behavioral contract instead of enforcement.** Honest version: there is no such thing as an enforced read-only SSH login on a stock box. So the contract is behavioral and hard: connect as a normal non-root user the human designates, run only the commands listed in Step 1 (all read-only), never write a file on the box, never install anything, never use sudo. Anything that needs elevated read access is quoted for the human to run themselves and paste back. Never accept root SSH keys or a root login, even if offered.
2. **SSH keys never travel through chat.** Chat transcripts are logs. The human adds a public key for a keypair they control to the box's `~/.ssh/authorized_keys` themselves. Running locally, you use the user's own SSH agent or config. Running inside a Maritime agent, the user generates a dedicated keypair on their own machine and installs the private key via `maritime env set` or the dashboard env vars pane (encrypted at rest), never by pasting into the conversation. Remind them to remove the key from `authorized_keys` when the migration is done.
3. **Never modify or delete anything on the box.** No `systemctl disable`, no `pm2 stop`, no editing crontabs, no killing processes, ever, not even if asked mid-flow. Cutover is a human-run checklist at the end of this document.
4. **Confirm before every paid action.** Creating a Maritime agent charges the first month at creation. Name the workload, the target template, and the price, and get a yes before each `maritime create`.

## Prerequisites

- `ssh` and `rsync` locally. Verify access with `ssh <user>@<host> hostname` before anything else.
- `maritime` CLI: `npm i -g maritime-cli`, then `export MARITIME_TOKEN=mk_...` (the user mints a key with `maritime keys create --name migration --json`). Always pass `--json`: success is JSON on stdout, failure is JSON on stderr, branch on exit code.
- `jq`.

## Step 1: Inventory

The commands below are the complete list you may run on the box. Every one is read-only. Set `BOX=<user>@<host>` once and sweep all five habitats a workload can live in:

```bash
# Cron: the most common home of a VPS job
ssh "$BOX" 'crontab -l'
ssh "$BOX" 'cat /etc/crontab'
ssh "$BOX" 'ls /etc/cron.d /etc/cron.daily /etc/cron.hourly'
# Other users' crontabs need root. Quote this for the HUMAN to run, never run it yourself:
#   sudo ls /var/spool/cron/crontabs      # Debian/Ubuntu layout; RHEL-family boxes use /var/spool/cron

# systemd: long-running services and timers
ssh "$BOX" 'systemctl list-units --type=service --state=running'
ssh "$BOX" 'systemctl cat <unit>'          # for user-authored units, typically in /etc/systemd/system
ssh "$BOX" 'systemctl list-timers'
ssh "$BOX" 'systemd-analyze calendar "<OnCalendar spec>"'  # read-only: normalizes a timer spec, prints its next firing

# Process managers, whichever exists on the box
ssh "$BOX" 'pm2 ls'
ssh "$BOX" 'pm2 describe <name>'
ssh "$BOX" 'pm2 env <id>'                  # env of one pm2 process; treat output as credentials
ssh "$BOX" 'forever list'
ssh "$BOX" 'supervisorctl status'
ssh "$BOX" 'cat /etc/supervisor/conf.d/*'  # Debian layout; if absent, cat /etc/supervisord.conf and follow its [include] section

# Sessions someone left running
ssh "$BOX" 'tmux ls'
ssh "$BOX" 'screen -ls'
ssh "$BOX" 'ps aux'                        # to see what actually runs inside those sessions

# Docker
ssh "$BOX" 'docker ps'
ssh "$BOX" 'docker inspect <container>'    # image, command, env; see the warning below

# General lay of the land
ssh "$BOX" 'ps aux --sort=-%cpu | head -30'
ssh "$BOX" 'date'                          # the box timezone; cron lines run in it
ssh "$BOX" 'ls ~/apps ~/bot ~/scripts'     # guesses; let the ps and pm2 output guide the paths
```

**`docker inspect` output contains env secrets in plain text.** Treat it like a credential: never echo it into chat, never quote it in your summary. Write the values you need straight into a local `.env` file (Step 3) and nothing else.

Version note: pm2, forever, and supervisor drift across versions. If a subcommand or flag above is missing, run `pm2 --help` or `supervisorctl help` on the box and adapt; the intent (list processes, show one process with its env) is stable. Also, `ssh "$BOX" 'pm2 ls'` can fail with "command not found" when pm2 was installed via nvm, because non-interactive shells skip nvm's setup; the fix is the full path, usually `~/.nvm/versions/node/<ver>/bin/pm2`.

Agent-shaped workloads are the ones that wake up, do a job, and go quiet, or sit waiting for messages: cron scripts, report generators, scrapers, Slack/Telegram/Discord bots, queue drainers, sync jobs. Present the inventory to the user as a table (workload, where it lives, schedule or trigger, what it appears to do) and let them pick what moves.

## Step 2: Map

| VPS concept | Maritime concept |
|---|---|
| crontab line / /etc/cron.d entry | Agent + trigger (same cron expression; shift hours if the box is not UTC) |
| systemd timer + oneshot service | Agent + trigger (convert OnCalendar to cron) |
| systemd long-running service | Agent (always-on or auto-sleep) |
| pm2 / forever / supervisor process | Agent |
| Bot in a tmux or screen session | Agent (`--always-on`, or auto-sleep if it only reacts to messages, which is cheaper) |
| Docker container | Agent (often the same Dockerfile, minus the gotcha below) |
| .env files, `Environment=` lines, pm2 env | Agent env vars, AES-encrypted at rest |
| journalctl, pm2 logs, nohup.out | Agent logs (`maritime logs`) |

Crontab lines are already real five-field cron, so trigger expressions copy over unchanged when the box runs UTC; otherwise note the offset from `date`, plan to shift the hour fields when you recreate the trigger in Step 3, and watch for a `CRON_TZ=` line, which overrides the box timezone for that one table. One trap: lines in `/etc/crontab` and `/etc/cron.d` carry a sixth field, the username, between the schedule and the command; take the five time fields only. systemd `OnCalendar` converts mechanically: `OnCalendar=*-*-* 05:00:00` becomes `0 5 * * *`. For anything fancier, `systemd-analyze calendar` from Step 1 prints the normalized form and the next firing; a bare weekday spec implies midnight, so `OnCalendar=Mon..Fri` means Mon through Fri at 00:00:00, cron `0 0 * * 1-5`. Write out the equivalent cron and show the user both forms.

## Step 3: Recreate on Maritime

**Pick the target shape.** Two honest cases:

- **The workload is an LLM-driven bot or assistant.** Recreate it on a Maritime framework template. Enumerate live, never hardcode ids:
  ```bash
  maritime templates --json
  maritime create <name> --template <id-from-that-list> --json
  ```
- **The workload is arbitrary code (a script, a pm2 app, a container).** This is a port, not a copy, and you should say so. Budget an hour, not a click. Pull the code down read-only, then get it into a git repo the user controls:
  ```bash
  rsync -az "$BOX":/path/to/app ./app-copy
  ```
  Add a Dockerfile if there is none. The minimal patterns for a Python and a Node worker:
  ```dockerfile
  FROM python:3.13-slim
  RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/*
  WORKDIR /app
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  COPY . .
  CMD ["python", "main.py"]
  ```
  ```dockerfile
  FROM node:24-slim
  RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/*
  WORKDIR /app
  COPY package*.json ./
  RUN npm ci --omit=dev
  COPY . .
  CMD ["node", "index.js"]
  ```
  Dockerfile gotcha that will bite: never use a shell-string CMD like `CMD ["sh", "-c", "python main.py $PORT"]`. Maritime's micro-VM init flattens CMD to one string and the VM kernel-panics on boot. Launch a real program directly, as both patterns above do. And keep `ca-certificates` in slim images or every HTTPS call fails. Then:
  ```bash
  maritime create <name> --repo https://github.com/<user>/<repo> --json
  ```

**Env vars and secrets.** Gather from the three places a VPS hides them: `pm2 env <id>` output, `Environment=` and `EnvironmentFile=` lines in the unit files you already read with `systemctl cat`, and `.env` files sitting next to the code. Cat them straight into a local file, never into chat:

```bash
ssh "$BOX" 'cat /path/to/app/.env' > ./vps.env
# append KEY=value lines extracted from pm2 env / unit files / docker inspect by hand
maritime env import <agent> ./vps.env --reload --json
rm ./vps.env
```

Env changes apply on next boot unless you pass `--reload`.

**Schedules.** Recreate each cron line as a Maritime trigger with the same expression: `maritime triggers create <agent> --type cron --cron "0 5 * * *" --json`, or in the dashboard: agent page, Triggers pane, cron type. On backends older than 2026-08-07 the CLI form created triggers that never fired (the config key mismatch is fixed platform-side now), so always confirm with `maritime triggers list <agent> --json` and watch the first firing. Maritime triggers do not inherit the box's timezone: assume UTC, shift the hour fields when the box runs something else, and confirm the first real firing time in Step 4. OpenClaw-family agents can equivalently use their native cron, which Maritime syncs. For bring-your-own-code agents remember that timers inside a sleeping VM do not fire; either serve `GET /schedules`, use the SDK scheduler observer, or mark the agent `--always-on` at create time. A tmux-resident bot that only reacts to inbound messages can usually auto-sleep instead, which is cheaper.

## Step 4: Verify before touching the box

Do not proceed until the Maritime side has done the job at least once:

```bash
maritime status <agent> --json | jq -r '.status'
maritime chat <agent> "run your task once now and report what you did" --json | jq -r '.response'
maritime logs <agent> --level error --json
```

For scheduled workloads, wait for one real scheduled firing and check its output against a known-good run from the box (the human pulls `journalctl -u <unit>` or `pm2 logs` for comparison). Both sides can safely run in parallel; that is the point of leaving the box untouched.

## Step 5: Cutover and decommission (human runs every command here)

Print this checklist for the user. These are write operations on the box, so they run them over their own SSH session. You never run them.

1. Pause the old job, reversibly. Cron: comment out the line with `crontab -e` (uncomment to restore). systemd: `systemctl disable --now <unit>` (restore with `systemctl enable --now <unit>`). pm2: `pm2 stop <name> && pm2 save` (restore with `pm2 start <name> && pm2 save`). supervisor: `supervisorctl stop <name>`. tmux bot: detach and stop the process inside the session.
2. Watch Maritime for a few days: `maritime logs <agent> --json`. If anything is wrong, the one-line restore above brings the box back instantly.
3. Only when satisfied, remove: delete the unit files and `systemctl daemon-reload`, `pm2 delete <name> && pm2 save`, delete the crontab lines for real, remove the code directory.
4. Remove the migration public key from `~/.ssh/authorized_keys` on the box.
5. Cancel the VPS itself only if it hosts nothing else. Before canceling, check for other tenants of the box: databases, nginx sites, mail, backup jobs, a friend's side project. The inventory in Step 1 shows workloads, not everything; have the human do a final sweep before the box dies.

## What stays on the box

Be upfront about this list rather than letting the user discover it:

- **Databases on localhost (Postgres, MySQL, Redis).** Moving data is out of scope for this skill. Two honest options: switch the agent to a managed database it reaches over TLS, or keep the VPS alive as a database-only box the agent connects to over the public internet with a firewall rule. Either way, every `localhost` or `127.0.0.1` connection string you find in env vars is a decision point: it breaks the moment the worker moves. Flag each one explicitly and let the user choose before cutover.
- **Nginx sites, static sites, reverse proxies.** They stay. Maritime hosts agents, not the box's websites.
- **Mail servers, backup targets, VPN endpoints, anything else on the box.** Stays. This is exactly why step 5 warns about other tenants before canceling.

## Reference documentation

The authoritative pages for every source-side tool this skill touches. All links checked 2026-08-07.

- [crontab(5): tables for driving cron](https://man7.org/linux/man-pages/man5/crontab.5.html) - field layout, the CRON_TZ variable, system vs per-user crontabs
- [systemd.time(7): calendar event syntax](https://man7.org/linux/man-pages/man7/systemd.time.7.html) - the OnCalendar grammar you are converting to cron, with normalization examples
- [systemd.timer(5): timer units](https://man7.org/linux/man-pages/man5/systemd.timer.5.html) - how OnCalendar timers activate their matching service
- [systemctl(1)](https://man7.org/linux/man-pages/man1/systemctl.1.html) - list-units, cat, list-timers, and the enable/disable --now pair used at cutover
- [journalctl(1)](https://man7.org/linux/man-pages/man1/journalctl.1.html) - pulling a known-good run with -u for the Step 4 comparison
- [pm2 process management](https://pm2.keymetrics.io/docs/usage/process-management/) - list, describe, stop, delete, save
- [supervisord: running supervisorctl](https://supervisord.org/running.html) - supervisorctl actions and config file locations
- [tmux(1)](https://man7.org/linux/man-pages/man1/tmux.1.html) - list-sessions (ls) and attach
- [docker inspect](https://docs.docker.com/reference/cli/docker/inspect/) - the JSON you mine for image, command, and env
- [rsync(1)](https://man7.org/linux/man-pages/man1/rsync.1.html) - archive-mode pulls of the code directory

## Troubleshooting

- `maritime create` exits with HTTP 402: billing gate, the account needs a plan or payment method. The error detail names the fix; show it verbatim.
- `agent_unavailable` on chat: the agent is not running, `maritime start <agent> --json` first.
- Env var changes not visible: they land on next boot; use `--reload` or `maritime restart <agent>`.
- A 404 on exec/file operations usually means the agent is asleep, not gone. Start it, wait a few seconds, retry. Do not recreate.
- SSH prompts for a password: the public key is not installed for that user. Stop and ask the human to add it; never type or handle a password.
- `crontab -l` says "no crontab for <user>": that user has none; the job may live under another user (the human runs the sudo listing from Step 1) or in `/etc/cron.d`.

Command syntax in this skill was verified on 2026-08-07: box-side commands against the man pages and vendor docs linked above (cron, systemd, pm2, supervisor, tmux, Docker, rsync), maritime commands against the CLI source, and the Docker base image tags against Docker Hub. It has not been live-run against a real VPS, so if a box-side command errors on a flag, trust `<tool> --help` on the box over this document and continue.
