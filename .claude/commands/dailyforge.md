---
description: Run DailyForge to generate and post the daily Unity team standup report. Supports --dry-run to preview without posting.
argument-hint: "[--dry-run] [--lang <language>]"
---

Invoke the `dailyforge` skill to collect commits from all configured GitHub, GitLab, or Gitea repos in the last 24 hours, classify Unity files, generate narrative reports, and post them to the configured Discord webhooks. The git provider is selected by the `provider` field in `config.json`.

Arguments passed by the user: `$ARGUMENTS`

If `$ARGUMENTS` contains `--dry-run`, run the full pipeline but skip the Discord POST step (Step 6 in the skill). Print the generated reports to the chat instead so the user can preview them before going live.

If `$ARGUMENTS` contains `--lang <language>`, write the reports in that language instead of the `report_language` configured in `config.json`.

Read the skill instructions at `.claude/skills/dailyforge/SKILL.md` and follow them in order.

Do not ask the user for confirmation between steps — proceed end-to-end. Only stop if a prerequisite check fails or the config is invalid.
