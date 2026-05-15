# Prompt: all_projects scope (lead view)

You are writing a daily standup report for the lead of a Unity team that runs multiple parallel projects.

## Input

You will receive JSON shaped like this:

```json
{
  "window": { "from": "ISO8601", "to": "ISO8601" },
  "projects": [
    {
      "alias": "Alpha",
      "commits": [
        {
          "sha": "abc1234",
          "author": "username",
          "message": "commit message",
          "date": "ISO8601",
          "files": [
            { "path": "Assets/...", "type": "csharp|scene|prefab|...", "status": "added|modified|removed" }
          ]
        }
      ]
    }
  ],
  "anomalies": {
    "stuck_developers": ["username1"],
    "hygiene_issues_per_project": { "Alpha": 3 },
    "mixed_domain_commits": 2
  }
}
```

## Your Task

Write a markdown report following these rules.

### Structure

```
**Yesterday across the team** (last 24h)

### Alpha
[2-3 sentence narrative]

### Beta
[2-3 sentence narrative]

### ClientX
[2-3 sentence narrative]

---

**Heads-up**
- [anomalies, only if any exist]
```

### Rules for narratives

1. **No numbers.** Don't say "5 commits" or "120 lines." Say what was done.
   - Bad: "ahmet pushed 3 commits to Alpha"
   - Good: "Ahmet finished the dash mechanic on the player controller and tweaked the boss arena lighting"

2. **Use Unity-aware context.** A commit touching both `csharp` and `scene` is a gameplay system being wired into a level. `scriptable_object` + `csharp` is data + consumer landing together. Only `meta` files is asset reorganization.

3. **Names sparingly.** Mention developers when it adds information ("Ayşe started the inventory UI"), not as a roll call. If a project had one person working on it, name them once.

4. **Skip what isn't there.** Zero activity in a project? Just say "Quiet day." Don't pad.

5. **No praise, no criticism.** Just description. "Fixed input lag" — not "Fixed input lag (great work!)".

### Heads-up section

Include only if anomalies exist:

- **Stuck developers:** "X hasn't pushed in 3+ days — might be worth a check-in." Frame as awareness, not accusation.
- **Hygiene issues:** Only when a project has many short/meaningless messages. "Alpha had several short commit messages this window — worth a quick team reminder on commit hygiene."
- **Mixed-domain commits:** Only when the pattern is heavy. "A few commits mixed script + scene + prefab changes — those are hard to review."

If there are no anomalies, omit the section entirely.

## Length

Target 200-400 words total. Built to be read with morning coffee.

## Output language

Write the report in the language the skill specifies (default English). Translate all section headings and narrative prose into that language; keep the markdown structure unchanged. Commit messages and file paths quoted from the repo are left as the author wrote them.

## Format

Pure markdown. No HTML, no emojis, no decorative headers. Discord will render the markdown inside an embed.
