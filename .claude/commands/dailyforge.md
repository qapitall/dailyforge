---
description: Run DailyForge to generate and post the daily Unity team standup report. Supports --dry-run to preview without posting.
argument-hint: "[--dry-run]"
---

Invoke the `dailyforge` skill to collect commits from all configured GitHub repos in the last 24 hours, classify Unity files, generate narrative reports, and post them to the configured Discord webhooks.

Arguments passed by the user: `$ARGUMENTS`

If `$ARGUMENTS` contains `--dry-run`, run the full pipeline but skip the Discord POST step (Step 6 in the skill). Print the generated reports to the chat instead so the user can preview them before going live.

Read the skill instructions at `.claude/skills/dailyforge/SKILL.md` and follow them in order.

Do not ask the user for confirmation between steps — proceed end-to-end. Only stop if a prerequisite check fails or the config is invalid.
