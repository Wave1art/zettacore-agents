---
name: work-types
description: How Zettacore's work types compose, what each needs before work starts, and what the planner may create under its target. Load it before creating any item; list_work_types has the project's own.
---

# Work types and how they compose

Call `list_work_types` once per run. Each entry carries the type's `fields` and `field_schema`
(what `fields` accepts), its `aspects`, its default `allocation`, its `lifecycle` (display states,
the metastatus of each, the transitions) and its `composition` (which children it takes, which
parents it accepts). The table below is the default catalogue; where the project's list differs
(a custom type, a changed rule), the list wins.

| Type | Container | Children you may create | What it needs before work starts | Routed by default to |
|---|---|---|---|---|
| `initiative` | yes | `decision`, `feature` | A body with `## Goal`, `## Notes`, `## Open questions`, `## Out of scope`; a decision for each first-ring question (`scope-initiative`) | the owners |
| `decision` | no | none | `## Question` and `## Resolution mode`, and the fields `question` and `resolution_mode` (`grill`, `research`, `prototype`, `task`) | by mode: `grill` the owners, `research` and `prototype` `queue:research`, `task` `queue:planning` |
| `feature` | yes | `phase` (ordered), `task`, `bug` | A Spec as its description; phases with checklists and `precedes` relations; its workspace (`spec-to-tickets`) | `queue:planning` |
| `phase` | no | `task`, `bug` | A phase plan (the plan template), an implementation and an acceptance-criteria checklist, `parallel_group`, `reversibility`, `blast_radius` | `queue:coding` |
| `task` | no | none | What and why, when it is done, its checklists (`plan-task-or-bug`) | anyone |
| `bug` | no | none | `brief`, `repro`, `severity`, its checklists (`plan-task-or-bug`) | `queue:bugs` |

`question`, `review` and `contract_change` children are the engine's and the shift's, never yours.
Your own planning item (`planning`) is never a parent.

## Your reach

You write only your planning item, the target, and items under the target (its children and
theirs). Every new item's `parent_id` is the target or an item under it; anything else is read
only (`ITEM_OUT_OF_SCOPE`). So:

- A phase takes a task or a bug found while planning it (a follow-up, a defect the plan uncovers),
  not a split of itself: slices are sibling phases under the feature.
- A task or a bug takes no children you may create. One too large for one run is an open question
  that proposes the split (a feature with phases, say); you cannot create its siblings.
- A relation joins two items within your reach. A dependency on an item elsewhere (another
  feature, another team's bug) is an open question naming both keys, for a person to record.

## Creating

- `create_feature`, `create_phase`, `create_task`, `create_bug`, and `create_item` for the others
  (`item_type="decision"`), each with `parent_id` and, where the type takes them, `checklists`
  (`{"checklist_type": "implementation" | "acceptance_criteria", "label": ...}`) and `fields`
  matching its `field_schema`.
- What you create starts in its type's first state: backlog for most (a feature `triage`, a phase
  or task `backlog`, a bug `triage`). A decision has no backlog state and starts `ready`, so it
  reaches its resolver at once: create one only for a real first-ring question.
- Leave `allocation` out: the type's rule routes the item. An allocation you set may only be a work
  queue (`queue:<name>`, else `ALLOCATION_QUEUE_ONLY`).
- Work a person must deliver (it cannot verify itself in a sandbox) is created with `afk=false`
  and no allocation; the engine gives it to the project's members, or its owners. Who should take
  it is an open question.
- An item takes its parent's repository. Pass `repository_id="owner/repo"` only for work in
  another of the project's repositories (`list_repositories`).
- Never set `fields.autonomous` (a start mode is a person's: `START_MODE_PERSON`), never call
  `set_status`, `set_statuses`, `start_work` or `claim_item`. Accept plan readies what you
  created; the target keeps its own status.
- Containers (initiative, feature) are never claimed; their status follows their children.
