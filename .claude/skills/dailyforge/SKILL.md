---
name: dailyforge
description: Generate a daily Unity team standup report by collecting commits from multiple GitHub, GitLab, or self-hosted Gitea repos, classifying Unity files, and posting narrative summaries to Discord webhooks. Use when the user asks for "daily report", "standup", "team summary", or types /dailyforge.
---

# DailyForge Skill

You are the DailyForge reporter for a Unity development team. DailyForge supports three
git hosting providers — **GitHub**, **GitLab** (self-hosted or gitlab.com), and **Gitea** —
selected by the `provider` field in `config.json`. When invoked, follow these steps in
order. Do not ask the user for confirmation between steps; run end-to-end unless a
prerequisite check fails or the config is invalid.

## Argument Parsing

The slash command may pass arguments. Inspect the user message / `$ARGUMENTS` for the
following flags before starting:

- `--dry-run` — run every step EXCEPT Step 6 (Discord POST). Print the generated reports
  to the chat instead. Surface this clearly in Step 7.
- `--lang <language>` — override the report output language for this run (e.g.
  `--lang tr`, `--lang Turkish`, `--lang "Brazilian Portuguese"`). Takes precedence over
  `report_language` in `config.json`.

If no flags are passed, proceed normally.

## Shell Environment

All command examples in this skill are POSIX shell (bash). On Windows hosts, Claude
Code's default shell is PowerShell — wrap each shell invocation with `bash -c '…'`
(Git Bash or WSL must be installed). On macOS/Linux, run the commands directly. Do NOT
attempt to translate to PowerShell; the JSON-handling logic relies on `jq`.

## Provider Adapters

Every provider feeds the same downstream pipeline. After Step 2, the data is uniform
regardless of provider: a list of commits, each shaped as
`{sha, author, author_name, message, date, files: [{filename, status}]}`.

The provider differs only in **how** that data is fetched. Pick the adapter that matches
`config.json`'s `provider` field (default `github` if the field is absent).

| Provider | Auth | Commit list | Files per commit | `own_commits` match |
|---|---|---|---|---|
| `github` | `gh` CLI (existing auth) | `gh api repos/O/R/commits --paginate` | `gh api repos/O/R/commits/SHA` | commit `author.login` ↔ recipient `github_username` |
| `gitlab` | `PRIVATE-TOKEN` header, `GITLAB_TOKEN` from `.env` | `GET /api/v4/projects/{enc}/repository/commits` | `GET /api/v4/projects/{enc}/repository/commits/SHA/diff` | commit `author_email` ↔ recipient `email`, else `gitlab_username` (resolved to email, best-effort) |
| `gitea` | `Authorization: token` header, `GITEA_TOKEN` from `.env` | `GET /api/v1/repos/O/R/commits` (returns `files` inline) | inline `files`, fallback `GET /api/v1/repos/O/R/git/commits/SHA` | commit `author.login` ↔ recipient `gitea_username` |

For `gitlab` and `gitea`, the instance hostname comes from the `host` field in
`config.json` (e.g. `git.example.com`). For `github` the host is fixed and `gh` handles
GitHub Enterprise itself.

**Important — never pipe large API responses through a shell variable.** GitLab/Gitea
commit messages contain newlines and control characters; piping a big JSON body through
`$(...)` into `jq` triggers `jq: parse error: Invalid string: control characters ...`.
Always write `curl` output to a temp file and feed `jq` from the file (`jq … < file`).

### github adapter

```bash
gh api -X GET "repos/${OWNER}/${REPO}/commits" \
  -f since="${SINCE}" \
  --paginate \
  --jq '.[] | {sha, author: .author.login, author_name: .commit.author.name, message: .commit.message, date: .commit.author.date}'
# files per commit:
gh api "repos/${OWNER}/${REPO}/commits/${SHA}" \
  --jq '{sha, files: [.files[] | {filename, status}]}'
```

### gitlab adapter

```bash
# GitLab project identifier is the URL-encoded "owner/name" path (the / becomes %2F).
PROJECT_PATH=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${OWNER}/${REPO}', safe=''))" 2>/dev/null \
            || python -c "import urllib; print(urllib.quote('${OWNER}/${REPO}', ''))")

# Commit list — paginate via the X-Next-Page response header.
PAGE=1; : > /tmp/df_commits.jsonl
while : ; do
  curl -sS -D /tmp/df_headers -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
    "https://${HOST}/api/v4/projects/${PROJECT_PATH}/repository/commits?since=${SINCE}&per_page=100&page=${PAGE}&all=true" \
    -o /tmp/df_resp.json
  jq -c '.[] | {sha: .id, author: .author_email, author_name: .author_name, message: .message, date: .authored_date}' \
    < /tmp/df_resp.json >> /tmp/df_commits.jsonl
  NEXT=$(grep -i '^x-next-page:' /tmp/df_headers | tr -d '\r' | awk '{print $2}')
  [ -z "$NEXT" ] && break
  PAGE=$NEXT
done

# Files per commit — the diff endpoint also caps at 100 entries, so paginate it too.
PAGE=1; : > /tmp/df_files.jsonl
while : ; do
  curl -sS -D /tmp/df_dheaders -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
    "https://${HOST}/api/v4/projects/${PROJECT_PATH}/repository/commits/${SHA}/diff?per_page=100&page=${PAGE}" \
    -o /tmp/df_diff.json
  jq -c '.[] | {filename: .new_path, old: .old_path, status: (if .new_file then "added" elif .deleted_file then "removed" elif .renamed_file then "renamed" else "modified" end)}' \
    < /tmp/df_diff.json >> /tmp/df_files.jsonl
  NEXT=$(grep -i '^x-next-page:' /tmp/df_dheaders | tr -d '\r' | awk '{print $2}')
  [ -z "$NEXT" ] && break
  PAGE=$NEXT
done
```

The GitLab diff endpoint does not return per-file `additions/deletions` — that is fine,
classification only uses `filename`.

### gitea adapter

```bash
# Gitea's API mirrors GitHub's. The commit-list response includes a files[] array,
# so no second call is needed. Gitea returns commits newest-first; stop once a page
# reaches commits older than the window (covers Gitea versions without ?since=).
PAGE=1; : > /tmp/df_commits.jsonl
while : ; do
  curl -sS -H "Authorization: token ${GITEA_TOKEN}" \
    "https://${HOST}/api/v1/repos/${OWNER}/${REPO}/commits?limit=50&page=${PAGE}&stat=true&files=true" \
    -o /tmp/df_resp.json
  COUNT=$(jq 'length' < /tmp/df_resp.json)
  [ "$COUNT" = "0" ] && break
  jq -c --arg since "$SINCE" '.[] | select(.commit.author.date >= $since) |
    {sha: .sha, author: (.author.login // .commit.author.email), author_name: .commit.author.name,
     message: .commit.message, date: .commit.author.date,
     files: [(.files // [])[] | {filename: .filename, status: .status}]}' \
    < /tmp/df_resp.json >> /tmp/df_commits.jsonl
  OLDEST=$(jq -r '.[-1].commit.author.date' < /tmp/df_resp.json)
  [ "$OLDEST" \< "$SINCE" ] && break
  PAGE=$((PAGE + 1))
done
```

If a Gitea commit's `files` array comes back empty (very old Gitea, or a commit with no
file stats), fall back to `GET /api/v1/repos/O/R/git/commits/SHA` and read its `files[]`.

## Prerequisites Check

Before doing anything else, verify the environment. Some checks depend on `provider`, so
read `config.json`'s `provider` field first (default `github` if missing):

1. `curl` is installed: `which curl`
2. `jq` is installed: `which jq`
3. `python` or `python3` is available (timestamp math + URL encoding): `which python3 || which python`
4. **Only if `provider` is `github`:** `gh` CLI is installed (`which gh`) and authenticated (`gh auth status`)
5. `config.json` exists in the current working directory (or at `~/.config/dailyforge/config.json`)
6. `.env` file exists and contains the Discord webhook URLs referenced in `config.json`, plus:
   - `provider: gitlab` → `GITLAB_TOKEN`
   - `provider: gitea` → `GITEA_TOKEN`

If any check fails, stop and tell the user exactly what's missing with a clear fix
command (e.g., `brew install jq`, `winget install jqlang.jq`, and `GitHub.cli` only when
`provider` is `github`).

## Step 0: Doctor / Preflight

Run a single preflight pass and print a status block so every failure mode (missing
tool, bad token, wrong host, typo in a repo path) surfaces at once before any reporting
work begins.

First, source `.env` (`set -a; . ./.env; set +a`) and read `provider`, `host`, the
provider token variable, and the `repos` list from `config.json` — the doctor needs them
for the probes below. Step 1 then performs full config validation.

Resolve the authenticated user with the provider's identity endpoint:

- `github` → `gh api user --jq .login`
- `gitlab` → `curl -sf -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" "https://${HOST}/api/v4/user" | jq -r .username`
- `gitea` → `curl -sf -H "Authorization: token ${GITEA_TOKEN}" "https://${HOST}/api/v1/user" | jq -r .login`

Then probe each configured repo (one cheap request — e.g. the project/repo endpoint).
Print:

```
DailyForge doctor (provider: gitlab)
   curl     OK
   jq       OK
   python3  OK
   .env     OK (GITLAB_TOKEN set)
   Auth     OK (authenticated as sercancengiz)
   Repos    OK (4/4 reachable: Alpha, Beta, ClientX, ClientY)
```

If auth fails, stop with the provider-specific fix (see Error Handling). If some repos
are unreachable, print them as `WARN` and continue with the reachable ones.

## Step 1: Load Configuration

First, load `.env` into the shell environment so `GITLAB_TOKEN` / `GITEA_TOKEN` and the
`${!WEBHOOK_ENV}` indirection in Step 6 resolve correctly:

```bash
# Load .env into the shell environment (auto-export every var until set +a)
set -a
. ./.env
set +a
```

If `.env` does not exist, fail with a clear "copy .env.example to .env and fill in your
provider token + webhook URLs" message.

Then read `config.json`. Validate these required fields:
- `version` (must be `1`)
- `provider` (one of `github`, `gitlab`, `gitea`; default to `github` if missing)
- `host` (hostname string — **required** when `provider` is `gitlab` or `gitea`; ignored for `github`)
- `lookback_hours` (integer; default to `24` if missing)
- `report_language` (string; the language reports are written in — a code like `en` / `tr` or a name like `English` / `Turkish`; default to `English` if missing)
- `repos` (non-empty array of `{owner, name, alias}` — for GitLab, `owner` is the group/namespace and `name` is the project slug)
- `recipients` (non-empty array of `{name, webhook_env, scope}`)

Valid `scope` values: `all_projects`, `own_commits`, `summary_only`.

For each recipient, check that `webhook_env` resolves to a non-empty value in the shell
environment. If a recipient's env var is missing, warn but continue with the rest.

For `own_commits` recipients, the identity key is provider-specific:
- `github` → `github_username` (matched against commit `author.login`)
- `gitlab` → `email` (matched against commit `author_email`); `gitlab_username` is also
  accepted and resolved to an email best-effort via `GET /api/v4/users?username=…`
- `gitea` → `gitea_username` (matched against commit `author.login`)

If an `own_commits` recipient has no usable identity key for the active provider, treat
its filter as matching nothing and warn.

## Step 2: Collect Commit Data

Compute the lookback window start. Use Python for the timestamp math — it is portable
across macOS (BSD date), Linux (GNU date), and Windows (Git Bash / WSL), unlike
`date -d`. Use a **timezone-aware** UTC value (`datetime.utcnow()` is deprecated in
Python 3.12+ and leaks a runtime warning into shell output):

```bash
SINCE=$(python3 -c "import datetime; print((datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(hours=${LOOKBACK_HOURS})).strftime('%Y-%m-%dT%H:%M:%SZ'))" 2>/dev/null \
     || python -c "import datetime; print((datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(hours=${LOOKBACK_HOURS})).strftime('%Y-%m-%dT%H:%M:%SZ'))")
```

For each repo, fetch commits within the window and the file list per commit using the
**adapter that matches `provider`** (see the Provider Adapters section above). For
`gitlab` and `gitea`, set `HOST` from `config.json`'s `host` field.

Drop commits whose `author` (login or email) or `author_name` matches an entry in
`ignore_authors`.

Do **not** fetch diff content — file paths plus message is enough and keeps token cost
low.

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

If `unity_options.filter_meta_only_commits` is `true` and a commit contains only `.meta`
files, skip it — it's almost certainly a Unity reimport.

If a commit is made up of `.meta` rename pairs, tag the commit `type: "asset_rename"`
rather than dropping it.

## Step 4: Detect Anomalies

Walk the collected data once and surface:

- **Stuck developers:** any recipient whose provider-specific identity key
  (`github_username` / `gitlab_username` or `email` / `gitea_username`) has zero commits
  across all monitored repos in the lookback window. Mark `stuck: true`.
- **Commit hygiene issues:** commits with messages under 10 characters, or matching
  `wip`, `fix`, `asd`, `update`, `temp`. Count per repo.
- **Mixed-domain commits:** commits touching `csharp` + `scene` + `prefab` together
  (these are messy to review in Unity).

These observations feed the prompt but never appear as raw numbers in user-facing output.

## Step 5: Generate Reports (per scope)

For each recipient, build a tailored report.

**Output language.** Resolve the report language in this order: the `--lang` argument,
then `config.json`'s `report_language`, then `English`. Write every report — section
headings, narrative prose, and the anomaly / heads-up text — entirely in the resolved
language. Keep the markdown structure and the scope-specific layout from the prompt
templates; only the human-readable text is translated. The prompt templates are written
in English as a reference — do not copy their wording verbatim when another language is
selected. Commit messages and file paths quoted from the repo stay as the author wrote
them; do not translate them.

### Scope `all_projects` (lead view)

Use `prompts/all-projects.md`. Pass the full structured data. Expected output: a
narrative summary per project, with an anomaly section if applicable.

### Scope `own_commits` (individual view)

Filter the data to only this recipient's commits, matching on the provider-specific
identity key (see Step 1). Use `prompts/own-commits.md`. Expected output: a friendly
personal recap plus, optionally, a single gentle suggestion if commit hygiene is poor.

### Scope `summary_only` (team channel)

Use `prompts/summary-only.md`. Expected output: a 2-3 line high-level team heartbeat with
no names and no per-project breakdown.

## Step 6: Post to Discord

**If `--dry-run` was passed, skip this step entirely.** Instead, print each generated
report to the chat with a header line like
`--- Report for <recipient.name> (scope: <scope>) ---` so the user can review before
going live.

Otherwise, for each generated report, post to the recipient's webhook with curl:

```bash
WEBHOOK="${!WEBHOOK_ENV}"
TODAY=$(python3 -c "import datetime; print(datetime.datetime.now(datetime.timezone.utc).strftime('%Y-%m-%d'))" 2>/dev/null \
     || python -c "import datetime; print(datetime.datetime.now(datetime.timezone.utc).strftime('%Y-%m-%d'))")
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
- If a report exceeds 4096 chars, split into multiple embeds in the same message
  (grouped by project)

## Step 7: Summarize to the User

After posting (or dry-running), print a short summary in the Claude Code chat:

```
DailyForge report generated
   Provider: gitlab (git.example.com)
   Window: last 24h
   Language: tr
   Repos scanned: 10
   Commits found: 47
   Recipients posted: 3 (lead, ahmet, team_channel)

Anomalies surfaced:
   - 2 stuck developers flagged
   - 1 repo with commit hygiene issues
```

If `--dry-run` was active, replace the "Recipients posted" line with
`DRY RUN — no webhooks called`.

If any webhook POST failed, report the failure but don't retry automatically.

## Error Handling

- **No commits in lookback window:** still post a "Quiet day" message to all recipients;
  do not silently skip.
- **GitHub `gh` rate limited:** report clearly. Suggest the user wait or use a PAT with
  higher limits.
- **GitLab/Gitea 401 or 403:** token invalid or missing scopes. Tell the user to
  regenerate it —
  GitLab: `https://${HOST}/-/user_settings/personal_access_tokens` with `read_api` and
  `read_repository`;
  Gitea: `https://${HOST}/user/settings/applications` with read-only repository access.
- **GitLab/Gitea 404 on a repo:** the repo/project path is wrong or the token cannot see
  it. Skip that repo, warn, and continue with the rest.
- **GitLab/Gitea 429 (rate limit):** report clearly. Suggest the user wait or use a token
  from an account with higher limits.
- **Webhook returns non-2xx:** report the URL **masked** and the status code, then
  continue with the other recipients. Mask format: keep the scheme/host and replace the
  two path secrets — `https://discord.com/api/webhooks/****/****`.
- **Config validation fails:** stop immediately. Show exactly which field is invalid.

## Notes

- Do not invent commit data. If the provider returns nothing, the report says nothing.
- Do not include commit counts or leaderboards in any output.
- Do not include code diffs — only file paths and messages.
- This skill should run end-to-end without prompting the user once `/dailyforge` is
  typed, as long as the config and `.env` are in place.
