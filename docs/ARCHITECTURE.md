# Architecture

DailyForge intentionally has very few moving parts. The v0.1 design fits inside a single Claude Code session — no servers, no databases, no background processes, no API keys to manage. The v0.2 CLI mode reuses the same building blocks, just behind a different wrapper.

## v0.1 — Skill mode (current)

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

1. **`gh api`** (bash) — fetches commits and file lists from each monitored repo
2. **Unity classifier** (inline in SKILL.md) — tags each file by type (`.cs`, `.unity`, `.prefab`, `.asset`, `.meta`, etc.) and filters `.meta`-only commits
3. **Claude** — turns the structured commit data into a narrative report per recipient scope
4. **`curl`** — posts each report to the configured Discord webhook as a rich embed

State is per-run. Nothing persists between invocations except `config.json` and `.env`.

## Why a Claude Code Skill?

Instead of the traditional CLI-first approach, the primary distribution is Claude Code's native skill system. The trade-offs:

- **No API keys.** The user's Claude Pro/Max session is already authenticated. No `ANTHROPIC_API_KEY` to manage.
- **No cron.** The user opens Claude Code in the morning and types `/dailyforge`. The user is the trigger.
- **No persistent process.** Nothing running in the background.
- **Free GitHub auth via `gh` CLI.** If the user has run `gh auth login`, Claude Code already has access. No PAT to generate.
- **Single setup step.** Clone the repo, drop `.claude/skills/dailyforge/` in place, run.

The downside is that v0.1 requires a human in the loop. The v0.2 CLI mode exists for cases where automation is needed.

## v0.2 — CLI mode (planned)

```
┌─────────────────────────────────────────────────────────────┐
│           dailyforge CLI (Node.js + TypeScript)             │
└─────────────────────────────────────────────────────────────┘
           │                    │                    │
           ▼                    ▼                    ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ Octokit or   │   │ Unity-aware  │   │ Discord      │
   │ gh CLI       │   │ classifier   │   │ webhooks     │
   └──────────────┘   └──────────────┘   └──────────────┘
                              │                    ▲
                              ▼                    │
                      ┌──────────────────┐         │
                      │ Anthropic API    │─────────┘
                      └──────────────────┘
```

Same `config.json` schema, same prompt templates, same Unity classifier — only the wrapper differs. The CLI is what makes GitHub Action and cron-based deployment possible. Requires an Anthropic API key (instead of a Claude Code session).

See [`ROADMAP.md`](ROADMAP.md) for the full v0.2+ direction.

## Data flow & boundaries

| Boundary | Reads / Writes | Note |
|---|---|---|
| `gh api` (GitHub) | reads commit metadata only | author, message, file paths, timestamps — never diff content in v0.1 |
| Claude (Anthropic) | receives structured commit JSON, returns narrative | never receives webhook URLs |
| `curl` (Discord) | writes rich embed | one POST per recipient |
| Local filesystem | reads `config.json` and `.env` | never writes anywhere except stdout |

See [`../SECURITY.md`](../SECURITY.md) for the threat model.
