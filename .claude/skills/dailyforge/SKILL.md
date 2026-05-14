---
name: dailyforge
description: Generate a daily Unity team standup report by collecting commits from multiple GitHub repos, classifying Unity files, and posting narrative summaries to Discord webhooks. Use when the user asks for "daily report", "standup", "team summary", or types /dailyforge.
---

# DailyForge Skill

You are the DailyForge reporter for a Unity development team. When invoked, follow these steps in order. Do not ask the user for confirmation between steps; run end-to-end unless a prerequisite check fails or the config is invalid.

## Argument Parsing

The slash command may pass arguments. Inspect the user message / `$ARGUMENTS` for the following flags before starting:

- `--dry-run` — run every step EXCEPT Step 6 (Discord POST). Print the generated reports to the chat instead. Surface this clearly in Step 7.

If no flags are passed, proceed normally.

## Shell Environment

All command examples in this skill are POSIX shell (bash). On Windows hosts, Claude Code's default shell is PowerShell — wrap each shell invocation with `bash -c '…'` (Git Bash or WSL must be installed). On macOS/Linux, run the commands directly. Do NOT attempt to translate to PowerShell; the JSON-handling logic relies on `jq`.

## Prerequisites Check

Before doing anything else, verify the environment:

1. `gh` CLI is installed: `which gh` (or `bash -c "which gh"` on Windows)
2. `gh` is authenticated: `gh auth status`
3. `jq` is installed: `which jq`
4. `python` or `python3` is available (used for portable timestamp math): `which python3 || which python`
5. `config.json` exists in the current working directory (or at `~/.config/dailyforge/config.json`)
6. `.env` file exists and contains the webhook URLs referenced in `config.json`

If any check fails, stop and tell the user exactly what's missing with a clear fix command (e.g., `brew install gh jq`, `winget install GitHub.cli jqlang.jq`).

## Step 1: Load Configuration

First, load the webhook URLs from `.env` into the shell environment so the `${!WEBHOOK_ENV}` indirection in Step 6 resolves correctly:

```bash
# Load .env into the shell environment (auto-export every var until set +a)
set -a
. ./.env
set +a
```

If `.env` does not exist, fail with a clear "copy .env.example to .env and fill in your webhook URLs" message.

Then read `config.json`. Validate these required fields:
- `version` (must be `1`)
- `lookback_hours` (integer; default to `24` if missing)
- `repos` (non-empty array of `{owner, name, alias}`)
- `recipients` (non-empty array of `{name, webhook_env, scope}`)

Valid `scope` values: `all_projects`, `own_commits`, `summary_only`.

For each recipient, check that `webhook_env` resolves to a non-empty value in the shell environment. If a recipient's env var is missing, warn but continue with the rest.

## Step 2: Collect Commit Data

For each repo, fetch commits within the lookback window. Use Python for the timestamp math — it is portable across macOS (BSD date), Linux (GNU date), and Windows (Git Bash / WSL), unlike `date -d`:

```bash
SINCE=$(python3 -c "import datetime; print((datetime.datetime.utcnow() - datetime.timedelta(hours=${LOOKBACK_HOURS})).strftime('%Y-%m-%dT%H:%M:%SZ'))" 2>/dev/null \
     || python -c "import datetime; print((datetime.datetime.utcnow() - datetime.timedelta(hours=${LOOKBACK_HOURS})).strftime('%Y-%m-%dT%H:%M:%SZ'))")
gh api -X GET "repos/${OWNER}/${REPO}/commits" \
  -f since="${SINCE}" \
  --paginate \
  --jq '.[] | {sha, author: .author.login, message: .commit.message, date: .commit.author.date}'
```

For each commit, fetch the file list:

```bash
gh api "repos/${OWNER}/${REPO}/commits/${SHA}" \
  --jq '{sha, files: [.files[] | {filename, status, additions, deletions}]}'
```

Drop commits whose `author` matches an entry in `ignore_authors`.

Do **not** fetch diff content in v0.1 — file paths plus message is enough and keeps token cost low.

## Step 3: Classify Files (Unity-aware)

Tag every file in every commit with a type:

| Extension / Pattern | Type |
|---|---|
| `.cs` | `csharp` |
| `.unity` | `scene` |
| `.prefab` | `prefab` |
| `.asset` | `scriptable_object` |
| `.meta` | `meta` (filter unless it's the only thing in the commit) |
| `.shader`, `.shadergraph`, `.hlsl` | `shader` |
| `.mat` | `material` |
| `.anim`, `.controller` | `animation` |
| `.png`, `.jpg`, `.tga`, `.psd` | `texture` |
| `.fbx`, `.obj`, `.blend` | `model` |
| `.wav`, `.ogg`, `.mp3` | `audio` |
| `.json`, `.yaml`, `.xml` | `data` |
| `package.json`, `manifest.json` | `package_manifest` |
| anything else | `other` |

If `unity_options.filter_meta_only_commits` is `true` and a commit contains only `.meta` files, skip it — it's almost certainly a Unity reimport.

If a commit is made up of `.meta` rename pairs, tag the commit `type: "asset_rename"` rather than dropping it.

## Step 4: Detect Anomalies

Walk the collected data once and surface:

- **Stuck developers:** any recipient with a `github_username` who has zero commits across all monitored repos in the lookback window. Mark `stuck: true`.
- **Commit hygiene issues:** commits with messages under 10 characters, or matching `wip`, `fix`, `asd`, `update`, `temp`. Count per repo.
- **Mixed-domain commits:** commits touching `csharp` + `scene` + `prefab` together (these are messy to review in Unity).

These observations feed the prompt but never appear as raw numbers in user-facing output.

## Step 5: Generate Reports (per scope)

For each recipient, build a tailored report.

### Scope `all_projects` (lead view)

Use `prompts/all-projects.md`. Pass the full structured data. Expected output: a narrative summary per project, with an anomaly section if applicable.

### Scope `own_commits` (individual view)

Filter the data to only this recipient's `github_username`. Use `prompts/own-commits.md`. Expected output: a friendly personal recap plus, optionally, a single gentle suggestion if commit hygiene is poor.

### Scope `summary_only` (team channel)

Use `prompts/summary-only.md`. Expected output: a 2-3 line high-level team heartbeat with no names and no per-project breakdown.

## Step 6: Post to Discord

**If `--dry-run` was passed, skip this step entirely.** Instead, print each generated report to the chat with a header line like `--- Report for <recipient.name> (scope: <scope>) ---` so the user can review before going live.

Otherwise, for each generated report, post to the recipient's webhook with curl:

```bash
WEBHOOK="${!WEBHOOK_ENV}"
TODAY=$(python3 -c "import datetime; print(datetime.datetime.utcnow().strftime('%Y-%m-%d'))" 2>/dev/null \
     || python -c "import datetime; print(datetime.datetime.utcnow().strftime('%Y-%m-%d'))")
curl -X POST "${WEBHOOK}" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg title "DailyForge — ${TODAY}" \
    --arg desc "${REPORT_MARKDOWN}" \
    '{embeds: [{title: $title, description: $desc, color: 5814783}]}')"
```

**Discord limits:**
- Embed description: 4096 chars max
- Embeds per message: 10 max
- If a report exceeds 4096 chars, split into multiple embeds in the same message (grouped by project)

## Step 7: Summarize to the User

After posting (or dry-running), print a short summary in the Claude Code chat:

```
DailyForge report generated
   Window: last 24h
   Repos scanned: 10
   Commits found: 47
   Recipients posted: 3 (lead, ahmet, team_channel)

Anomalies surfaced:
   - 2 stuck developers flagged
   - 1 repo with commit hygiene issues
```

If `--dry-run` was active, replace the "Recipients posted" line with `DRY RUN — no webhooks called`.

If any webhook POST failed, report the failure but don't retry automatically.

## Error Handling

- **No commits in lookback window:** still post a "Quiet day" message to all recipients; do not silently skip.
- **`gh` rate limited:** report clearly. Suggest the user wait or use a PAT with higher limits.
- **Webhook returns non-2xx:** report the URL (masked) and status code. Continue with the other recipients.
- **Config validation fails:** stop immediately. Show exactly which field is invalid.

## Notes

- Do not invent commit data. If `gh` returns nothing, the report says nothing.
- Do not include commit counts or leaderboards in any output.
- Do not include code diffs in v0.1 — only file paths and messages.
- This skill should run end-to-end without prompting the user once `/dailyforge` is typed, as long as the config and `.env` are in place.
