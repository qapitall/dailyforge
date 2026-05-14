# Quick Start

Get DailyForge running in 5 minutes.

## Prerequisites

- [ ] Claude Code installed (Pro or Max plan) — check with `claude --version`
- [ ] `gh` CLI installed and authenticated — check with `gh auth status`
- [ ] `jq` installed — check with `jq --version`
- [ ] `python3` (or `python`) available — check with `python3 --version`
- [ ] `curl` installed (already on most systems)
- [ ] Read access to the GitHub repos you want to monitor

### Install the CLI tools

| OS | Command |
|---|---|
| macOS | `brew install gh jq python` |
| Ubuntu / Debian | `sudo apt install gh jq python3 curl` |
| Windows | `winget install GitHub.cli jqlang.jq Python.Python.3` |

**Windows users:** DailyForge's shell logic is POSIX (`bash`). Run Claude Code from **Git Bash** or **WSL**, not PowerShell — `gh`, `jq`, and the skill's shell snippets all assume a bash environment. Authenticate `gh` once with `gh auth login` from the same shell.

## Step 1: Clone

```bash
git clone https://github.com/qapitall/dailyforge.git
cd dailyforge
```

## Step 2: Configure

```bash
cp config.example.json config.json
$EDITOR config.json
```

Edit:
- `repos`: the repos you want to monitor (owner/name/alias)
- `recipients`: who gets what kind of report (scope: `all_projects`, `own_commits`, or `summary_only`)

## Step 3: Set up Discord webhooks

In Discord: Server Settings → Integrations → Webhooks → New Webhook.
Select the target channel, copy the URL.

```bash
cp .env.example .env
$EDITOR .env
```

Fill in the webhook URLs. **Never commit `.env`** — it's already in `.gitignore`.

## Step 4: Tell your team

Before running DailyForge against any repo, inform every developer whose commits will be processed. Send a written notice covering what the tool reads, where reports go, which third parties are involved, and how to raise concerns. See the "Ethical Framework" section of [`README.md`](README.md) for the required minimum and [`SECURITY.md`](SECURITY.md) for operator obligations under KVKK / GDPR. **Do not skip this step** — it's the ethical and legal foundation of the tool.

## Step 5: Run it

```bash
cd /path/to/dailyforge
claude
```

> **Windows:** open **Git Bash** or **WSL** for this step, not PowerShell.

In Claude Code, do a **dry-run first** to preview the reports without spamming Discord:

```
/dailyforge --dry-run
```

Claude prints each report to the chat. If the output looks right, run the real thing:

```
/dailyforge
```

Claude will run prerequisite checks, then:
1. Fetch last 24h commits from each repo
2. Classify Unity files
3. Generate a tailored report for each recipient
4. Post to the corresponding Discord webhooks
5. Print a summary

Check the Discord channels — verify the messages arrived and the format looks right.

## Daily usage

Open Claude Code, type `/dailyforge`. The report takes about 30-60 seconds.

## Troubleshooting

**`gh authentication failed`**
```bash
gh auth login
```

**`config.json not found`**
Make sure you're in the `dailyforge` directory when running Claude Code. The skill uses `pwd` to find the config.

**`Webhook returned 401`**
Webhook URL is wrong or revoked. Regenerate it in Discord.

**Webhook URL comes through empty / `curl` posts to nothing**
The skill sources `./.env` from the working directory. Make sure your `.env` is at the repo root and that each `webhook_env` in `config.json` matches a variable name in `.env` exactly (`DISCORD_WEBHOOK_LEAD`, not `DISCORD_WEBHOOK_lead`).

**`date: invalid option` or `python: command not found`**
The skill uses `python3`/`python` for portable timestamp math because BSD/macOS `date` and Windows shells don't accept `date -d`. Install Python 3 from the OS table above.

**Windows: command not found / strange quoting errors**
You're in PowerShell. Open Git Bash or a WSL terminal and re-run Claude Code from there.

**`No commits found`**
If there really are no commits in the lookback window, that's the correct output. Try bumping `lookback_hours` to test the pipeline.

**Report too long, Discord cut it off**
Known v0.1 limitation. v0.2 will split into per-project embeds.

## Next steps

- Re-read the "Ethical Framework" section in `README.md` and confirm the team-disclosure step is done
- Run for a week and tune the prompt templates in `prompts/` to your team's style
- File issues or feature requests on GitHub
