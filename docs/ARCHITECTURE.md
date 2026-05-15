# Architecture

DailyForge intentionally has very few moving parts. The whole design fits inside a single Claude Code session — no servers, no databases, no background processes. v0.2 adds GitLab and Gitea support behind a provider adapter, but the orchestration and delivery path are unchanged.

## v0.1 — Skill mode

```
┌─────────────────────────────────────────────────────────────┐
│                Claude Code Session (Pro/Max)                │
│                                                             │
│   User types: /dailyforge                                   │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │       .claude/skills/dailyforge/SKILL.md            │   │
│   │       (Claude reads and follows these steps)        │   │
│   └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│         ┌────────────────┼────────────────┐                 │
│         ▼                ▼                ▼                 │
│   ┌──────────┐    ┌──────────────┐  ┌────────────┐          │
│   │ gh api   │    │ Unity        │  │ curl ->    │          │
│   │ (bash)   │    │ classifier   │  │ Discord    │          │
│   │          │    │ (inline)     │  │ webhook    │          │
│   └──────────┘    └──────────────┘  └────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

The skill orchestrates:

1. **`gh api`** (bash) — fetches commits and file lists from each monitored repo (v0.2 generalizes this into a provider adapter — see below)
2. **Unity classifier** (inline in SKILL.md) — tags each file by type (`.cs`, `.unity`, `.prefab`, `.asset`, `.meta`, etc.) and filters `.meta`-only commits
3. **Claude** — turns the structured commit data into a narrative report per recipient scope
4. **`curl`** — posts each report to the configured Discord webhook as a rich embed

State is per-run. Nothing persists between invocations except `config.json` and `.env`.

## Why a Claude Code Skill?

Instead of the traditional CLI-first approach, the primary distribution is Claude Code's native skill system. The trade-offs:

- **No API keys for Claude.** The user's Claude Pro/Max session is already authenticated. No `ANTHROPIC_API_KEY` to manage.
- **No cron.** The user opens Claude Code in the morning and types `/dailyforge`. The user is the trigger.
- **No persistent process.** Nothing running in the background.
- **Free GitHub auth via `gh` CLI.** If the user has run `gh auth login`, Claude Code already has access. No PAT to generate. (GitLab and Gitea use a read-only token instead — see below.)
- **Single setup step.** Clone the repo, drop `.claude/skills/dailyforge/` in place, run.

The downside is that DailyForge requires a human in the loop — it runs when the user types `/dailyforge`.

## v0.2 — Multi-provider

v0.2 keeps the exact same skill mode and adds a provider adapter so the backend is a
config choice. The `provider` field in `config.json` selects one of three code paths;
everything after the fetch — the Unity classifier, the prompt templates, the Discord
delivery — is shared.

```
┌─────────────────────────────────────────────────────────────┐
│                .claude/skills/dailyforge/SKILL.md            │
│                                                              │
│   provider = github | gitlab | gitea                        │
│         │            │             │                        │
│         ▼            ▼             ▼                         │
│   ┌──────────┐ ┌────────────┐ ┌────────────┐                 │
│   │ gh api   │ │ GitLab     │ │ Gitea      │   provider      │
│   │ (GitHub) │ │ REST v4    │ │ REST v1    │   adapter       │
│   └──────────┘ └────────────┘ └────────────┘                 │
│         └────────────┼─────────────┘                         │
│                      ▼                                       │
│   ┌──────────────┐  ┌────────┐  ┌──────────────┐             │
│   │ Unity        │  │ Claude │  │ curl ->      │             │
│   │ classifier   │─▶│        │─▶│ Discord      │             │
│   └──────────────┘  └────────┘  └──────────────┘             │
└──────────────────────────────────────────────────────────────┘
```

- **GitHub** keeps the `gh` CLI path — existing auth, no token in `.env`.
- **GitLab** uses the REST API v4 over `curl` with a `read_api` + `read_repository`
  Personal Access Token (`GITLAB_TOKEN` in `.env`). Project identifiers are the
  URL-encoded `owner/name` path. The commit-diff endpoint is paginated.
- **Gitea** uses the REST API v1 over `curl` with a read-only token (`GITEA_TOKEN` in
  `.env`). Gitea's responses are GitHub-shaped and include affected files inline.

Same `config.json` schema (plus `provider` / `host`), same prompt templates, same Unity
classifier — only the fetch adapter differs.

See [`ROADMAP.md`](ROADMAP.md) for the full v0.2+ direction.

## Data flow & boundaries

| Boundary | Reads / Writes | Note |
|---|---|---|
| Provider API (GitHub / GitLab / Gitea) | reads commit metadata only | author, message, file paths, timestamps — never diff content |
| Claude (Anthropic) | receives structured commit JSON, returns narrative | never receives webhook URLs or provider tokens |
| `curl` (Discord) | writes rich embed | one POST per recipient |
| Local filesystem | reads `config.json` and `.env` | `.env` holds the provider token + webhook URLs; never writes anywhere except stdout and `/tmp` scratch files |

See [`../SECURITY.md`](../SECURITY.md) for the threat model.
