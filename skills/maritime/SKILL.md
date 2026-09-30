---
name: maritime
description: Deploy and operate AI agents on Maritime (maritime.sh), a cloud that gives every agent its own persistent Firecracker micro-VM that sleeps when idle and wakes in about a second. Use when asked to host, deploy, run, or manage an agent on Maritime, to give an agent a persistent computer, to provision one agent per end user from a backend, or to move a bot, worker, or scheduled job onto Maritime. Covers signup, headless login, the JSON output contract, exit codes, deploy recipes, and the email verification wall.
metadata:
  author: maritime-sh
  version: "1.0"
---

# Maritime

Maritime hosts AI agents. Each agent runs in its own micro-VM with a persistent
disk at `/data`. It sleeps when idle (free while asleep), wakes in about a
second on the next message or trigger, and hosting is a flat monthly plan with
no metering. The free plan includes 3 machines, no card.

This skill is for driving Maritime as a program or an AI agent, not as a
human at a keyboard. Every command speaks JSON and every failure is
structured.

## Rules

1. Pass `--json` on every command. Success: one JSON value on stdout, stderr
   empty. Failure: `{"ok": false, "error": {"code", "message", "status"}}` on
   stderr, stdout empty. Read stdout, branch on the exit code, never scrape
   human text.
2. Exit codes: `0` ok, `1` error, `2` auth, `3` not found, `4` usage.
3. Never guess flags. `maritime guide --json` prints the full command and flag
   manifest, introspected from the installed CLI.
4. Never paste a secret into chat. Keys go into env vars or `maritime env set`.
5. Confirm with the user before a paid create. On the free plan a create is
   free (3 machines). A refused create exits `1` with `error.status` 402 and
   `error.code` `seat_limit`; show `error.message` unchanged, it names the fix.

## Install

```bash
npm install -g maritime-cli      # Node 20.12 or later; the command is `maritime`
maritime --version               # 1.9.0 or later has `login --device` and `signup` in --help
```

If a flag below is unknown, the CLI is old: `npm install -g maritime-cli@latest`.

## Authenticate

Try these in order and stop at the first that works.

1. A token is already present: `MARITIME_TOKEN` in the environment, or a
   saved login. Check with `maritime whoami --json` (exit `2` means no token).
2. The user has an account and you have no browser: `maritime login --device
   --json`. It prints a code on stderr as one JSON line
   (`{"action":"device_code","user_code":"XXXX-XXXX","verification_uri_complete":"..."}`).
   Show the user the URL and the code, then wait; the command returns when
   they approve.
3. The user has an account and a browser on this machine: `maritime login`.
4. No account: `maritime signup -e <email> -p <password> -n <name> --json`.
   Ask the user for the email and password; never invent them. Signup saves a
   token and returns `emailVerified: false`.

The email verification wall: until the user clicks the link Maritime emailed
them, `create`, `deploy`, `start`, `restart`, `keys create` and computer
creation are refused with status `403` and `error.code`
`email_not_verified`. When you see it, tell the user to open the email and
click the link, offer `maritime login --resend` to send it again, then retry
the same command. Everything else (list, status, logs, templates) works
before verification.

For long-running automation mint a key once the account is verified:
`maritime keys create --name <purpose> --json` returns `raw_key` (`mk_...`,
shown once, never expires). Export it as `MARITIME_TOKEN`.

`MARITIME_API_URL` overrides the API base (default `https://api.maritime.sh`).

## Deploy

Pick the recipe that matches what the user has.

### A ready-made agent from a template

```bash
maritime templates --json                                  # ids: openclaw, hermes, zeroclaw, claude_code, codex, dsh
maritime create my-agent --template openclaw --json
maritime status my-agent --json                            # poll until .status is "active"
maritime chat my-agent "introduce yourself" --json | jq -r '.response'
```

Most templates need no model key (Maritime's metered LLM proxy). `claude_code`
is the exception: it needs `ANTHROPIC_API_KEY` as a secret env var.

### The user's own code (any framework, Dockerfile at the repo root)

```bash
maritime create my-bot --repo https://github.com/you/agent --json
maritime status my-bot --json                              # builds take 1 to 3 minutes
maritime logs my-bot --json
```

The container contract: listen on `0.0.0.0:$PORT` (injected, never hardcode
8080), answer `GET /health` with any 2xx, answer `POST /chat` with JSON
`{"message": "..."}` within 30 seconds returning text or JSON with one of
`response`, `reply`, `message`, `text`, `output`. Persist under `/data` only.
Never use a shell-string `CMD`; launch a real program. Full contract:
https://maritime.sh/docs/frameworks/custom.md

Private GitHub repos need the Maritime GitHub App installed from the
dashboard. Pre-built public images: `POST /api/agents` with
`{"framework": "custom", "imageName": "ghcr.io/acme/img:tag"}`.

### A public web app on its own HTTPS URL

```bash
maritime create my-app --repo https://github.com/you/app --public --port 3000 --json
```

It sleeps when idle and wakes on the next visit.

### Secrets and env vars

```bash
maritime env set my-agent OPENAI_API_KEY=sk-... --reload --json    # secret by default
maritime env import my-agent ./prod.env --reload --json
```

Without `--reload` the change applies on the next boot.

### Schedules

Timers inside a sleeping VM do not fire. Use platform triggers:

```bash
maritime triggers create my-agent --type cron --cron "0 9 * * *" --json
```

### One agent per end user, from a backend

Use the SDK instead of the CLI: `npm install maritime-sdk` or
`pip install maritime`, key in `MARITIME_API_KEY`. `provision({externalId})`
is idempotent. Docs: https://maritime.sh/docs/build.md

### A persistent desktop for an existing agent (Computers, paid plan)

One MCP server, one Linux desktop per end user id:

```bash
claude mcp add --transport http maritime-computers https://mcp.maritime.sh/mcp \
  --header "Authorization: Bearer mk_..."
```

Docs: https://maritime.sh/docs/computers.md

## Verify before you finish

1. `maritime status <agent> --json` reports `active` (or `sleeping` after
   idle; both are healthy).
2. `maritime chat <agent> "hello" --json` returns a reply, or for a custom
   container `maritime logs <agent> --json` shows the server listening.
3. Tell the user the agent name, the dashboard URL
   `https://maritime.sh/agents`, and what you changed in their repo.

## Inspect and repair

```bash
maritime list --json
maritime info my-agent --json                  # env count, triggers, last deploy
maritime history my-agent --json               # build and deploy attempts with buildLog
maritime exec my-agent ls /data --json | jq -r '.stdout'
maritime restart my-agent --json               # also picks up new env vars
maritime delete my-agent --yes --json          # --yes is required non-interactively
```

`error.code` `agent_unavailable` on chat means the agent is not running:
`maritime start <agent>`, wait a few seconds, retry. A 404 on exec or files
usually means the agent is asleep or errored: start it, do not recreate it.

## Reference

- Contract file: https://maritime.sh/docs/cli/llms.txt
- Setup procedure for agents: https://maritime.sh/agents.md
- Everything: https://maritime.sh/llms.txt (index) and https://maritime.sh/llms-full.txt
- REST API: https://api.maritime.sh/openapi.json (bearer `mk_` key)
- Any docs page as markdown: append `.md` to its URL, for example
  https://maritime.sh/docs/quickstart.md
