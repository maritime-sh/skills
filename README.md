# Maritime skills

Skills that teach an AI coding agent to deploy and operate agents on
[Maritime](https://maritime.sh): a cloud that gives every agent its own
persistent micro-VM that sleeps when idle and wakes in about a second.

## Install

Claude Code, Cursor, Codex, Gemini CLI, and 70 other agents, with the
[skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add maritime-sh/skills --skill maritime
```

Claude Code, as a plugin:

```
/plugin marketplace add maritime-sh/skills
/plugin install maritime@maritime
```

Any installer that reads the Agent Skills discovery index:

```bash
npx skills add https://maritime.sh
```

By hand: copy `skills/maritime/SKILL.md` into `.claude/skills/maritime/` in
your project. `maritime init` does this for you.

## Skills

| Skill | What it does |
|---|---|
| `maritime` | Sign up or sign in without a browser, the JSON output contract, deploy from a template or a repo, verify, and the email verification wall. |
| `migrate-from-aws` | Move scheduled Lambdas, EventBridge crons, ECS tasks, and EC2 bots onto Maritime. |
| `migrate-from-gcp` | Cloud Functions, Cloud Run jobs, Cloud Scheduler. |
| `migrate-from-azure` | Functions timers, Container Apps jobs, Key Vault. |
| `migrate-from-cloudflare` | Workers crons, KV, R2, D1 data. |
| `migrate-from-vercel` | vercel.json crons, serverless functions, env. |
| `migrate-from-heroku` | Scheduler jobs, worker dynos, config vars. |
| `migrate-from-fly` | Machines, fly.toml processes, secrets. |
| `migrate-from-render` | Cron jobs, background workers, env groups. |
| `migrate-from-digitalocean` | App Platform workers and jobs, droplet bots. |
| `migrate-from-vps` | Any server over SSH: systemd, pm2, crontab, tmux. |

Every migration skill uses read-only source credentials, never deletes
anything on the source provider, and hands the human a cutover checklist.

## For agents

The setup procedure is at https://maritime.sh/agents.md. The CLI contract is
at https://maritime.sh/docs/cli/llms.txt. Every docs page is served as
markdown when `.md` is appended to its URL.

## Source

These files are mirrored from Maritime's main repository. Open an issue here
for corrections; pull requests are folded back into the source and republished.
