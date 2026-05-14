# Changelog

All notable changes to DailyForge are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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

[Unreleased]: https://github.com/qapitall/dailyforge/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/qapitall/dailyforge/releases/tag/v0.1.0
