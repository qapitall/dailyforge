# Roadmap

DailyForge ships in small, focused versions. Each version is intentionally narrow — the goal is to keep the tool useful and auditable rather than feature-rich.

Concrete work items live in [GitHub Issues](https://github.com/qapitall/dailyforge/issues). This page captures the high-level direction only.

## v0.1 — Skill MVP (current)

The Claude Code skill that started the project. Status: shipped.

**Included:**
- `/dailyforge` slash command with `--dry-run`
- Multi-repo commit collection via `gh` CLI
- Unity-aware file classification (.cs / .unity / .prefab / .asset / .meta / shaders / textures / audio / models)
- Three recipient scopes: `all_projects`, `own_commits`, `summary_only`
- Discord rich-embed delivery
- POSIX shell + Python 3 for portable timestamp math (macOS / Linux / Windows via Git Bash or WSL)

**Known limitations:**
- Reports over 4096 chars get truncated by Discord embed limits (v0.2 will split per project)
- English output only
- No persistent state — each run is independent

## v0.2 — Standalone CLI

For teams that want DailyForge to run from a GitHub Action or a cron job, without needing an interactive Claude Code session.

**Planned:**
- TypeScript CLI: `dailyforge run`, `init`, `validate`, `me`
- Anthropic API key path (instead of Claude Code session)
- GitHub Action workflow template
- Per-project embed splitting for long reports
- Weekly rollup reports
- Opt-in personal commit-quality coaching

## v0.3 — Auto-invoke Skill

**Planned:**
- Experimental `invocation: auto` mode so Claude Code suggests the latest report at session start
- Lightweight session telemetry (opt-in)

## v0.4 — Extended Integrations

**Planned:**
- GitLab support
- Slack webhook support
- Bitbucket support
- i18n for report output (starting with Turkish)

## Open design questions

These are tracked as issues; pull requests welcome.

- **Skill language mix:** the skill currently mixes Bash (`gh`, `jq`, `curl`). Should some logic move to inline Node/Python for readability, at the cost of an extra dep?
- **`gh` rate limits:** fine for 10 repos × once a day, but a 50-repo team would need pagination tuning.
- **Report output language:** i18n architecture decisions go alongside v0.4 integrations.
