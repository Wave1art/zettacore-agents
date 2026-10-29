---
name: plan-task-or-bug
description: Plan a task, a bug or any other non-container item into what its run needs before it starts — a sharpened description, implementation and acceptance checklists, the bug's brief and repro, the workspace and the items it waits for. Changes the target itself; creates no children.
---

# Planning a task or a bug

A task or a bug is planned in place: its own body, fields and checklists, its workspace and its
relations. It takes no children you may create, so a split is a proposal, never an act.

## 1. Read before writing

Read the target, every comment on it and what it relates to. Find the code it touches in your
working directory (and the other repositories under `~/repos/` when it reaches them), the tests
that cover that code, and how `.zcore/testing.md` says they run. Search the project's items for
duplicates and earlier work (`search_items`); a likely duplicate is an open question naming both
keys, never a relation you guess.

## 2. The body

Keep what the person wrote. Add, under headings of your own:

- **Task:** `## What and why` (one paragraph), `## Done when` (observable outcomes), and `## Notes`
  (the files and modules it touches as found, the decisions you made and why).
- **Bug:** keep its `## Summary`, `## Reproduction Steps`, `## Observed Behaviour`, `## Expected
  Behaviour` and `## Affected Area`. Fill the `brief` field in this shape, from what the body and
  the code say:

  ```
  ## Expected
  ## Actual
  ## Reproduction
  ## Scope
  ## Severity
  ```

  and `repro` (the shortest command or steps that show it). Set `severity` only when it is empty,
  by the type's options (`critical`, `high`, `medium`, `low`). Name the suspected cause as
  `Suspected cause: <file or module> — <why>`, a hypothesis you checked against the code, never a
  conclusion; never write `root-cause:`, which the fixer records.

Write the body back whole with `update_item(item_id, description=..., fields=...)`.

## 3. Checklists

An implementation checklist of 3 to 8 entries (for a bug the first is "a failing regression test
that reproduces it") and an acceptance-criteria checklist whose entries assert facts directly.
Add them with `create_checklist_item(item_id, checklist_type, label)`; change an existing entry
with `update_checklist_items` rather than adding a second.

## 4. Size, hands and dependencies

- Work too large for one run (more than 8 implementation entries that do not merge): an open
  question proposing the split, for example a feature with phases. You cannot create siblings.
- Work that cannot verify itself in a sandbox (a contract file, access only a person has): an open
  question saying so; a person decides who delivers it.
- Work that waits for another item: a `precedes` relation when both lie within your reach; else an
  open question naming the key.

## 5. Workspace

Follow the `workspace` skill, declaring on the target itself (`item_id=<target>`).
