---
name: plan-summary
description: The exact shape of the plan summary, the planning item's closing comment that a person reads before Accept plan or Send back — items and dependencies, coverage, workspace, missing credentials with their links, open questions, and the readiness verdict.
---

# The plan summary

The summary is the `summary` of your last `end_step` (`NO_MORE_TASKS`); the engine posts it as
your planning item's closing comment, and the console shows it on the planning item. Keep the
first heading exactly `## Plan summary`. Keep every section, writing "None" where it is empty;
a person must see that you looked. No values, tokens or signed links anywhere; console form links
carry none and belong in it.

```
## Plan summary

Target: <key> <title> (<type>)
Pass: first | after send-back: "<the send-back comment's first line>"
Readiness: ready to accept | ready once <what> | not ready — <why>

### Items

| Key | Type | Title | Depends on | Group | Reversibility | Blast radius | Delivered by |
|---|---|---|---|---|---|---|---|
| <key> | phase | <title> | — | a | two_way | local | agent |
| <key> | phase | <title> | <key> | b | one_way | module | a person (afk: false: <reason>) |

Changed: <key> — <what changed>, for each item that existed before this pass
To cancel: <key> — <why it is no longer part of the plan> (you cannot cancel it)

### Coverage

| Requirement or decision | Phase | Checklist entry |
|---|---|---|
| R1 | <key> | <entry> |

Gaps: <each uncovered R<n> or D<n> and why> | None

### Workspace

| Name | Kind | Source | Target | Access | On | Status |
|---|---|---|---|---|---|---|
| <name> | mount, s3 | <bucket>/<prefix> | ~/data/ref | read-only | <key> | declared (mounted) |
| <name> | folder, git | owner/repo at main | ~/repos/owner/repo | read-only | <key> | declared (no credential needed) |
| <name> | mount, s3 | <bucket>/<prefix> | ~/data/out | read-write, per-run | <key> | not declared: credential missing |
| <name> | service | <the API> | env:<NAME> | — | — | not declared: a person adds it in the console |

### Missing credentials

- `<credential name>` for `<row name>`: <what it must be able to do>. Enter it: <credential.console_url>.
  Then send the plan back ("credentials recorded") and the next pass declares `<row name>`.

### Open questions

1. <the question, the options you see and your recommendation>

### Notes

- <the most cautious choices you made and why; phases kept outside 3 to 8 entries and why>
- Catalog skills were not checked (the planner cannot commit to the feature's branch).
- Accept plan readies the items above; the target keeps its own status.
```

For an initiative, the Items table lists the decisions (with their resolution mode in the Group
column) and the features; Coverage lists the Goal's parts against them. For a task or a bug, the
Items table lists the target itself with its checklists counted, and Coverage lists its acceptance
criteria.
