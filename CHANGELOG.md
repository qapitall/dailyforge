# Changelog

All notable changes to DailyForge are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.2.0] — 2026-05-15

### Added
- Multi-provider support — `provider` field in `config.json` selects `github`, `gitlab`, or `gitea`
- GitLab REST API v4 backend: PAT auth (`GITLAB_TOKEN`), URL-encoded project paths, `X-Next-Page` pagination
- Gitea REST API v1 backend: token auth (`GITEA_TOKEN`), inline commit-file lists
- `host` field in `config.json` for self-hosted GitLab / Gitea instances
- Provider Adapters section in `SKILL.md` — one fetch path per provider, shared classifier and delivery
- Step 0 preflight `doctor` — verifies tools, auth, and repo reachability in a single pass before reporting
- Provider-specific `own_commits` identity keys: `github_username`, `gitlab_username` / `email`, `gitea_username`
- Configurable report output language — `report_language` field in `config.json` plus a `--lang` runtime override on `/dailyforge`; report text was English-only before

### Changed
- `.env` now carries the provider API token (GitLab/Gitea) alongside the Discord webhook URLs
- Prerequisite checks are provider-aware — `gh` is required only when `provider` is `github`
- README, QUICKSTART, ARCHITECTURE, ROADMAP, and SECURITY updated for the three-provider model
- Roadmap restructured: the planned standalone CLI was dropped; v0.2 is now multi-provider support

### Fixed
- Timestamp math uses timezone-aware UTC (`datetime.now(timezone.utc)`) — avoids the Python 3.12+ `utcnow()` deprecation warning leaking into shell output
- Large GitLab/Gitea API responses are read from a temp file instead of a shell variable — avoids `jq` control-character parse errors on multi-line commit messages
- Commit-diff fetch is paginated — file lists for commits touching 100+ files are no longer silently truncated

## [0.1.0] — 2026-05-14

Initial public release. Skill-mode only — CLI mode is planned for v0.2.

### Added
- Claude Code skill at `.claude/skills/dailyforge/SKILL.md`
- `/dailyforge` slash command with `--dry-run` preview flag
- Multi-repo commit collection via `gh api`
- Unity-aware file classification: C# scripts, scenes, prefabs, ScriptableObjects, `.meta` files (with rename detection), shaders, materials, animations, textures, models, audio
- `.meta`-only commit filtering for Unity reimports
- Anomaly surfacing: stuck developers, commit-hygiene patterns, mixed-domain commits — surfaced as awareness, never as metrics
- Three recipient scopes: `all_projects` (lead view), `own_commits` (private developer DM), `summary_only` (team-channel heartbeat)
- Discord webhook delivery as rich embeds
- POSIX shell + Python 3 for portable timestamp math (macOS / Linux / Windows via Git Bash or WSL)
- `.env` loading via `set -a; . ./.env; set +a` so webhook indirection resolves correctly

### Documentation
- README with problem framing, architecture, and configuration
- QUICKSTART with OS-specific install matrix (macOS / Ubuntu / Windows) and troubleshooting
- SECURITY.md with threat model, vulnerability reporting, and operator obligations under KVKK / GDPR
- Ethical framework with anti-patterns and required team-disclosure content (in README)
- docs/ARCHITECTURE.md with component diagrams and design rationale
- docs/ROADMAP.md outlining v0.2+ direction
- docs/example-output.md with fixtures for all three scopes
- GitHub issue and pull-request templates
- GitHub Actions workflow for Markdown lint + link checking

### Known limitations
- Reports over 4096 chars are truncated by Discord embed limits (per-project splitting planned for v0.2)
- English output only — i18n on the v0.4 roadmap
- No standalone CLI yet — requires an interactive Claude Code session
- No persistent state — each run is independent

[Unreleased]: https://github.com/qapitall/dailyforge/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/qapitall/dailyforge/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/qapitall/dailyforge/releases/tag/v0.1.0
