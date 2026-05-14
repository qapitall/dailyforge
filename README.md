# DailyForge

> A Claude Code–powered, multi-project daily standup reporter for Unity teams.

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Status](https://img.shields.io/badge/status-v0.1-green.svg)
![Built with Claude Code](https://img.shields.io/badge/built%20with-Claude%20Code-orange.svg)
![Platform](https://img.shields.io/badge/platform-macOS%20%7C%20Linux%20%7C%20Windows-lightgrey.svg)

**Status:** v0.1 — Skill mode shipped. CLI mode (v0.2) is on the [roadmap](docs/ROADMAP.md).

See [`QUICKSTART.md`](QUICKSTART.md) to get running in 5 minutes, or [`docs/example-output.md`](docs/example-output.md) for sample reports.

<!--
Screenshot placeholder.
Add a real Discord-side screenshot at docs/images/discord-preview.png showing
one of the rich embeds in a channel, then uncomment the line below.

![DailyForge daily report in Discord](docs/images/discord-preview.png)
-->

---

## Problem

If you lead a team where multiple developers work on multiple parallel Unity projects, checking each repository every morning to understand "who did what, how is progress, is commit quality healthy, is anyone stuck" is not practical.

**Gaps in existing solutions:**
- Enterprise tools like Jellyfish and Swarmia are expensive and overkill for indie/mid-sized teams.
- Claude Code skills like Daily Standup Generator are designed for single-repo, single-user scenarios.
- None of them produce Unity-aware context (.meta, prefab, scene, ScriptableObject).
- None of them reduce the "10 developers across 10 projects" scenario into a single view.

## Solution

A tool that pulls commit activity from the last 24 hours across multiple GitHub repositories, enriches it with Unity-specific context, runs it through Claude to produce a narrative summary, and posts daily reports to Discord webhooks.

It currently ships as a Claude Code Skill (`/dailyforge`) — zero setup, no API keys, no cron. A standalone CLI for automation (cron / GitHub Action) is on the [v0.2 roadmap](docs/ROADMAP.md).

**Target user:** a team lead or tech lead managing multiple parallel Unity projects.

**How it works:**
1. Lead defines monitored GitHub repos and recipients in `config.json`
2. Opens Claude Code, types `/dailyforge`
3. Claude fetches commits via `gh` CLI, classifies Unity files, and generates a tailored report per recipient scope
4. Each report is posted to the configured Discord webhook

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the design rationale and component diagram.

---

## Ethical Framework

The core design choice: **the recipient list is fully under the lead's control.** The lead can route reports to themselves only, to themselves plus selected team members, or to a team-wide channel.

### Required practices

1. **Inform your team in writing before deploying.** Every developer whose commits will be processed must be told, in advance, in a form they can retain. A proper notice covers at minimum:
   - Who is operating the tool (data controller, contact)
   - What data is read — commit author, message, file paths, timestamps via `gh api`
   - What data is **not** read — local machine, IDE, chat, unpushed work
   - Where reports go — which Discord channels, who has access
   - Which third parties receive the data — GitHub, Discord, Anthropic (all US-based)
   - The lawful basis for processing and how long Discord retains the reports
   - How to raise a concern or request access / deletion of one's data

2. **Reads only data already shared via git.** No local machine access, no IDE access, no screen recordings, no keystrokes. Everything is visible via a plain `gh api` call.

3. **Not a performance review tool.** No commit counts, no line counts, no leaderboards. Reports are narrative. Lines of code is a bad proxy and is excluded by design.

4. **Configuration is auditable.** `config.json` is committed. Webhook URLs live in `.env` (never committed).

5. **Open source.** Any team member can open the repo and verify exactly what is sent.

### Anti-patterns to avoid

- Using the report to make performance judgments ("X had fewer commits this week")
- Treating it as a micromanagement signal ("Why no commits at 2pm?")
- Installing the tool without telling the team
- Storing reports in HR files or referencing them in performance reviews
- Refusing to let developers see their own reports if they ask

Features that increase the surveillance surface (hourly activity maps, per-person rankings, "productivity scores") may be declined regardless of code quality. See [`SECURITY.md`](SECURITY.md) for operator obligations under KVKK / GDPR.

### Recipient model

Config defines a `recipients` array:

```json
"recipients": [
  { "name": "lead", "webhook_env": "DISCORD_WEBHOOK_LEAD", "scope": "all_projects" },
  { "name": "ahmet", "webhook_env": "DISCORD_WEBHOOK_AHMET", "scope": "own_commits", "github_username": "ahmetdev" },
  { "name": "team_channel", "webhook_env": "DISCORD_WEBHOOK_TEAM", "scope": "summary_only" }
]
```

Three scopes:
- `all_projects` — full report across every monitored repo (for the lead)
- `own_commits` — only the developer's own commits, sent to them privately
- `summary_only` — high-level team heartbeat, no names, for a public channel

---

## What's in v0.1

- Claude Code skill at `.claude/skills/dailyforge/SKILL.md`
- Multi-repo commit collection via `gh api`
- Unity-aware file classification (C#, scenes, prefabs, ScriptableObjects, shaders, materials, animations, textures, models, audio)
- `.meta`-only commit filtering and rename detection
- Narrative reports — no commit counts, no line counts, no leaderboards
- Discord webhook delivery as rich embeds
- Three recipient scopes: `all_projects`, `own_commits`, `summary_only`
- `/dailyforge --dry-run` preview mode

See [`docs/ROADMAP.md`](docs/ROADMAP.md) for what's coming in v0.2+.

---

## Configuration

Copy and edit the example files at the repo root:

- `config.example.json` → `config.json` — repos to monitor and recipients
- `.env.example` → `.env` — Discord webhook URLs (never commit)

Required tools: `gh` (authenticated), `claude` (Pro or Max), `jq`, `python3`, `curl`. Full prerequisite checklist and OS-specific install commands in [`QUICKSTART.md`](QUICKSTART.md).

---

## Sample Output

```
**Yesterday across the team** (last 24h)

### Alpha
Ahmet wrapped up the dash mechanic on the player controller and wired it into
the boss arena scene. The hitstop ScriptableObject was tuned alongside —
gameplay and data landed together.

### Beta
Ayşe started the inventory UI; layout and slot prefabs are in, no behaviour
yet. Quiet otherwise.

### ClientX
Quiet day.

---

**Heads-up**
- A few commits mixed script + scene + prefab changes — those are hard to review.
```

Full set (lead view, individual DM, team-channel heartbeat) in [`docs/example-output.md`](docs/example-output.md).

## Repository Layout

```
dailyforge/
├── README.md
├── QUICKSTART.md
├── SECURITY.md                  # Vulnerability reporting + threat model
├── CHANGELOG.md
├── LICENSE                      # MIT
├── .gitignore
├── config.example.json
├── .env.example
├── docs/
│   ├── ARCHITECTURE.md          # Design rationale + component diagrams
│   ├── ROADMAP.md               # v0.2+ direction
│   └── example-output.md        # Sample reports for each scope
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   ├── workflows/
│   │   └── lint.yml             # CI: markdown lint + link checker
│   └── PULL_REQUEST_TEMPLATE.md
├── .claude/
│   ├── skills/
│   │   └── dailyforge/
│   │       └── SKILL.md         # Main skill definition
│   └── commands/
│       └── dailyforge.md        # /dailyforge slash command
└── prompts/
    ├── all-projects.md          # Lead view prompt
    ├── own-commits.md           # Individual DM prompt
    └── summary-only.md          # Team channel prompt
```

---

## Roadmap

See [`docs/ROADMAP.md`](docs/ROADMAP.md) — v0.2 CLI mode, v0.3 auto-invoke, v0.4 GitLab/Slack/Bitbucket integrations.

---

## Contributing

Issues and pull requests are welcome. DailyForge has explicit anti-surveillance design constraints (see the "Ethical Framework" section above) that shape which features are accepted — features that increase the surveillance surface may be declined regardless of code quality.

For bugs and security issues, see [`SECURITY.md`](SECURITY.md).

---

## License

MIT — see [`LICENSE`](LICENSE).
