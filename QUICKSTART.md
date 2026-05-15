# Quick Start

Get DailyForge running in 5 minutes.

## Prerequisites

- [ ] Claude Code installed (Pro or Max plan) — check with `claude --version`
- [ ] `jq` installed — check with `jq --version`
- [ ] `python3` (or `python`) available — check with `python3 --version`
- [ ] `curl` installed (already on most systems)
- [ ] Read access to the repos you want to monitor
- [ ] **GitHub only:** `gh` CLI installed and authenticated — check with `gh auth status`
- [ ] **GitLab / Gitea:** a read-only API token (see Step 3)

### Install the CLI tools

| OS | Command |
|---|---|
| macOS | `brew install jq python` (add `gh` for GitHub) |
| Ubuntu / Debian | `sudo apt install jq python3 curl` (add `gh` for GitHub) |
| Windows | `winget install jqlang.jq Python.Python.3` (add `GitHub.cli` for GitHub) |

**Windows users:** DailyForge's shell logic is POSIX (`bash`). Run Claude Code from
**Git Bash** or **WSL**, not PowerShell — `jq`, `curl`, and the skill's shell snippets
all assume a bash environment. If you use GitHub, authenticate `gh` once with
`gh auth login` from the same shell.

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
- `provider`: `github`, `gitlab`, or `gitea`
- `host`: the instance hostname for `gitlab` / `gitea` (e.g. `git.example.com`) — leave empty for `github`
- `report_language`: the language reports are written in (e.g. `English`, `Turkish`, `tr`) — defaults to `English`
- `repos`: the repos you want to monitor (`owner`/`name`/`alias` — for GitLab, `owner` is the group/namespace)
- `recipients`: who gets what kind of report (scope: `all_projects`, `own_commits`, or `summary_only`)

For `own_commits` recipients, add the provider's identity key: `github_username`,
`gitea_username`, or — for GitLab — `email` (GitLab matches commit authors by email).

## Step 3: Set up authentication

### GitHub

Authenticate the `gh` CLI once: `gh auth login`. No token goes in `.env`.

### GitLab

Create a Personal Access Token at
`https://<your-gitlab-host>/-/user_settings/personal_access_tokens` with the
`read_api` and `read_repository` scopes.

### Gitea

Create an access token at `https://<your-gitea-host>/user/settings/applications`
with read-only repository access.

## Step 4: Set up Discord webhooks and `.env`

In Discord: Server Settings → Integrations → Webhooks → New Webhook.
Select the target channel, copy the URL.

```bash
cp .env.example .env
$EDITOR .env
```

Fill in the webhook URLs, and — for GitLab/Gitea — the `GITLAB_TOKEN` or `GITEA_TOKEN`.
**Never commit `.env`** — it's already in `.gitignore`.

> **macOS:** Finder hides dotfiles like `.env` by default. Toggle visibility with
> `Cmd + Shift + .`, or just open it directly with `code .env` / `open -e .env`.

## Step 5: Tell your team

Before running DailyForge against any repo, inform every developer whose commits will be
processed. Send a written notice covering what the tool reads, where reports go, which
third parties are involved, and how to raise concerns. See the "Ethical Framework"
section of [`README.md`](README.md) for the required minimum and [`SECURITY.md`](SECURITY.md)
for operator obligations under KVKK / GDPR. **Do not skip this step** — it's the ethical
and legal foundation of the tool.

## Step 6: Run it

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

To preview in a different language without touching `config.json`, pass `--lang`:

```
/dailyforge --dry-run --lang tr
```

Claude will run prerequisite checks and a preflight `doctor` pass, then:
1. Fetch last 24h commits from each repo
2. Classify Unity files
3. Generate a tailored report for each recipient
4. Post to the corresponding Discord webhooks
5. Print a summary

Check the Discord channels — verify the messages arrived and the format looks right.

## Minimal smoke test

The example config ships three recipients with three different scopes. To get your
first green run fast, trim it down:

> One repo, one recipient with `scope: "all_projects"`, one webhook. Set `provider` and
> `host`, fill in a single `DISCORD_WEBHOOK_*` in `.env`, and run `/dailyforge --dry-run`.

Once that works end-to-end, add the rest of your repos and recipients.

## Daily usage

Open Claude Code, type `/dailyforge`. The report takes about 30-60 seconds.

## Troubleshooting

**`gh authentication failed`** (GitHub)
```bash
gh auth login
```

**GitLab/Gitea returns `401` or `403`**
The token is invalid or missing scopes. Regenerate it — GitLab needs `read_api` +
`read_repository`, Gitea needs read-only repository access. Confirm `GITLAB_TOKEN` /
`GITEA_TOKEN` is set in `.env` and `host` in `config.json` is correct.

**GitLab/Gitea returns `404` on a repo**
The repo/project path is wrong, or the token can't see it. For GitLab, `owner` must be
the full group/namespace. DailyForge skips an unreachable repo and continues with the rest.

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
v0.2 splits long reports into per-project embeds. If you still see truncation, check that the report isn't a single project over 4096 chars.

## Next steps

- Re-read the "Ethical Framework" section in `README.md` and confirm the team-disclosure step is done
- Run for a week and tune the prompt templates in `prompts/` to your team's style
- File issues or feature requests on GitHub
