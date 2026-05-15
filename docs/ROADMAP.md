# Roadmap

DailyForge ships in small, focused versions. Each version is intentionally narrow — the goal is to keep the tool useful and auditable rather than feature-rich.

Concrete work items live in [GitHub Issues](https://github.com/qapitall/dailyforge/issues). This page captures the high-level direction only.

## v0.1 — Skill MVP

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

## v0.2 — Multi-provider support

DailyForge originally assumed GitHub + the `gh` CLI. Many Unity teams run their code on a
self-hosted **GitLab** or **Gitea** instance. v0.2 makes the provider a config choice
without changing the rest of the pipeline. Status: shipped.

**Included:**
- `provider` field in `config.json` — `github`, `gitlab`, or `gitea`
- GitLab REST API v4 backend (PAT auth, URL-encoded project paths, paginated diff endpoint)
- Gitea REST API v1 backend (token auth, GitHub-shaped responses)
- Single skill, one provider-adapter section — same classifier, prompts, and Discord delivery
- Preflight `doctor` step that verifies tools, auth, and repo reachability in one pass
- Configurable report output language (`report_language` in `config.json`, `--lang` per-run override)
- Per-project embed splitting for long reports
- Reliability fixes: timezone-aware UTC, file-based JSON handling, paginated commit-diff fetch

## v0.3 — Auto-invoke Skill

**Planned:**
- Experimental `invocation: auto` mode so Claude Code suggests the latest report at session start
- Lightweight session telemetry (opt-in)
- Weekly rollup reports
- Opt-in personal commit-quality coaching

## v0.4 — Extended Integrations

**Planned:**
- Slack webhook support
- Bitbucket support

## Open design questions

These are tracked as issues; pull requests welcome.

- **Skill language mix:** the skill currently mixes Bash (`curl`/`gh`, `jq`). Should some logic move to inline Node/Python for readability, at the cost of an extra dep?
- **API rate limits:** fine for 10 repos × once a day, but a 50-repo team would need pagination tuning across all three providers.
- **Mixed-provider orgs:** v0.2 picks one provider per config. A team split across GitHub and GitLab would need per-repo provider routing — deferred unless there is demand.
