# Example Output

These are fabricated samples that show what each `recipients[].scope` produces. Use them to set expectations before your first real run, and as a yardstick when tuning the prompt templates in `prompts/`.

All examples follow the rules in `prompts/all-projects.md`, `prompts/own-commits.md`, and `prompts/summary-only.md`: no commit counts, no line counts, no leaderboards, narrative only.

---

## Scope: `all_projects` (lead view)

Sent privately to the team lead. Goes through every monitored repo, surfaces anomalies at the bottom.

```markdown
**Yesterday across the team** (last 24h)

### Alpha
Ahmet wrapped up the dash mechanic on the player controller and wired it into
the boss arena scene. The hitstop ScriptableObject was tuned alongside —
gameplay and data landed together. Ayşe touched the input remapping UI prefab.

### Beta
Mert started the inventory system; the slot prefab and a placeholder layout
are in, no behaviour yet. The shader for the rarity glow effect was prototyped
in a sandbox scene.

### ClientX
Quiet day.

---

**Heads-up**
- A few commits mixed script + scene + prefab changes — those are hard to review.
- Mert hasn't pushed in three days on ClientX, where he was on the boss AI task. Might be worth a check-in.
```

---

## Scope: `own_commits` (individual DM)

Sent privately to each developer, filtered to their own commits. Friendly tone, optional single hygiene nudge.

```markdown
**Your day, Ahmet** (last 24h)

You closed out the dash mechanic on the player controller — the input handler,
the cooldown logic, and the iframe window all landed together. You wired it
into the boss arena scene and tuned the hitstop ScriptableObject in the same
window, so gameplay and data went live as a pair. Clean cut.

---

One small thing: a couple of your commit messages this window were under ten
characters ("wip", "fix"). Future-you will appreciate a slightly fuller note —
even one sentence helps when you're reading back through the log.
```

If commit hygiene is fine, the trailing section is omitted entirely.

---

## Scope: `summary_only` (team channel)

Sent to a public team channel. No names, no per-project breakdown, just a heartbeat.

```markdown
**Team — last 24h**

Steady day across the projects. Gameplay systems and UI work landed in the
larger codebases; one project was quiet. No blockers surfaced.
```

For a quiet day:

```markdown
**Team — last 24h**

Quiet across the board. No commits in the window.
```

---

## When to expect a "Quiet day"

- Weekends, holidays, off-cycle weeks
- Right after a release when everyone is on rest/decompression
- When `lookback_hours` is too short relative to your team's commit cadence — try bumping it to `48` or `72` in `config.json` and re-running

DailyForge never invents activity. If the provider's API returns no commits in the window, the report says so.
