---
name: scope-initiative
description: Plan an initiative into its first ring of decisions (at most 12 open) and the features its goal implies, keeping the initiative's body sections as /zc:scope and /zc:decide read them. The planner's form of the plugin's /zc:scope, with no interview.
---

# Scoping an initiative

The output is decisions and feature stubs, not code. A person resolves the decisions
(`/zc:decide`, or the queue their mode routes to); each feature is planned on its own once its
Spec stands.

## 1. The body

The initiative's description holds these sections; keep their names, which `/zc:decide` and
`/zc:write-spec` read:

```
## Goal
## Notes            (### Later questions inside it: one held-back question per line)
## Open questions   (one line per open decision: - #<key> <question>)
## Out of scope
## Resolved         (added by /zc:decide; never yours to write)
```

When Goal or Out of scope is missing or vague, draft it from the description, the brief and the
codebase, mark it `(drafted by the planner)`, and put it on the open-questions list for a person
to confirm. Never change a Goal a person wrote; propose the change as an open question.

## 2. The first ring

List the questions that must be answered before anything else can be: those whose answer does not
depend on another open question's. "Which queue holds the jobs?" is first ring; "how is that
queue partitioned?" waits, under `### Later questions` in `## Notes`.

- Phrase each as a question with one answer ("Which store holds sessions?", not "Sessions").
- A question the codebase or the project's other items answer is not a decision: answer it in
  `## Notes`, citing where (a path, an item key).
- Give each a resolution mode, with a one-line reason: `grill` (a judgement a person reaches by
  being questioned), `research` (a fact someone must find), `prototype` (needs something built to
  see), `task` (needs a small piece of work first).

## 3. The cap

At most 12 decisions are open (status `ready` or `resolving`) under one initiative. Count them with
`list_items(parent_id=<initiative>, item_type=["decision"])`. Create at most `12 - open` new ones,
those that block the most others; the rest go under `### Later questions`, and the summary says
so.

## 4. Create

```
create_item(item_type="decision", parent_id=<initiative>, title=<the question>,
  description="## Question\n\n<the question, one line of context>\n\n## Resolution mode\n\n<mode>",
  fields={"question": <the question>, "resolution_mode": <mode>})
```

A decision starts `ready` and reaches its resolver at once, so create no speculative ones.

For each feature the goal plainly implies and whose shape no open decision decides:
`create_feature(parent_id=<initiative>, title=..., description=<a Spec draft>)`. The draft uses the
Spec's sections as far as they are known (Problem Statement, Goals, Non-Goals, Requirements,
Workspace, Out of Scope), each unknown marked `Open: depends on #<decision key>` or left for
`/zc:write-spec`. Do not slice these features into phases in the same pass.

## 5. Record

Rewrite `## Open questions` as one `- #<key> <question>` line per decision now open under the
initiative, those already open included: read the description, change only that section and
`### Later questions`, and write the whole description back with `update_item`. The initiative's
own status is a person's to move (to `deciding`); say so in the summary.
