---
name: spec-to-tickets
description: Turn a feature's Spec into tracer-bullet vertical phases under the feature, each with its plan, checklists, risk fields and precedes relations, checked by the ledger rule and a coverage matrix. The planner's form of the plugin's /zc:spec-to-tickets, with no person to ask.
---

# Spec to tickets

The target is a feature; its description is the Spec (`# Name`, `Feature short name:`, Problem
Statement, Goals, Non-Goals, User Stories, Requirements with `#### R<n> — ` headings, Seams,
Implementation Decisions with `D<n>`, external and toolchain dependencies and Standing
invariants, Workspace, Testing Decisions, Out of Scope, Further Notes). Where a section is
missing or silent, plan from what there is and put each gap on the open-questions list; never
invent a requirement. A decision log `.zcore/decisions/<feature short name>.md` in the
repository, when there is one, is the authoritative record of what was decided.

## 1. Baseline and durable decisions

The environment baseline is what `.zcore/tech-stack.md` and `.zcore/testing.md` record, with
`.zcore/verify.sh`. The feature's workspace is the rows of the Spec's **Workspace** section (the
`workspace` skill declares them). Before slicing, name the decisions unlikely to change across the
phases (route structures, schema shape, key data models, the authentication approach, third-party
boundaries); they go in every phase's `## Architectural decisions`.

## 2. Vertical slice rules

- **Governing invariant: every phase is implementable and verifiable using only the baseline, the
  feature's workspace and the union of `Delivers` of its transitive blockers.** Every rule below
  applies it.
- Each phase is a thin vertical slice through every layer (schema, API, UI, tests), demoable or
  verifiable on its own. Prefer many thin slices to few thick ones.
- Every phase has a `Delivers` / `Requires` ledger of concrete, greppable artifacts.
- No file or function names likely to change; do name durable decisions (routes, schema shapes,
  model names).
- **Walking skeleton:** for a project's first feature, or one that adds toolchain or services, the
  first phase executes the delivery pipeline end to end: one trivial test through the real
  runner, a no-op migration applied and rolled back (with a database), the app booted, one
  endpoint hit. A connectivity ping is not enough.
- **Baseline diff:** a `Requires` row sourced `baseline` is something the baseline files record.
  Anything missing is added by an `afk=false` phase or removed from the plan.
- **Workspace rule:** a phase that reads or writes a Workspace row names it in `Requires` as
  `workspace: <name> at <target>` and says in "What to build" what it uses it for. A phase never
  assumes a repository, bucket or credential the Workspace section does not list.
- **Human hands:** a phase that cannot verify its own acceptance criteria from inside a sandbox
  (it changes a contract file, needs access only a person has) carries `> afk: false — <reason>`
  and is created with `afk=false`.
- **Contract checkpoint:** a new stack surface (a new language, a first end-to-end tier, a new
  service or package) gets an `afk=false` phase extending `.zcore/verify.sh`, before any phase
  that depends on it.
- **Suite checkpoint:** a new test stack surface (a compose stack, a first integration or
  end-to-end tier, a new runner) gets an `afk=false` phase (the contract checkpoint's, when both
  apply) whose acceptance criteria say: the tests run under their own `verify.sh` stage; two
  copies of it pass side by side with different `ZC_RUN_ID`s; `.zcore/testing.md` `## Test Tiers`
  and `## Isolation` record the stage, its tier, its budget and its isolation.

## 3. Checks on the draft, before any item exists

1. **Coverage matrix:** one row per P0 requirement (`R<n>`) and per work-implying decision
   (`D<n>`), each mapped to the phase and checklist item that delivers it. Close every gap with a
   checklist item; a gap you leave goes on the open-questions list. A Spec without R/D numbers
   gets no matrix; say so.
2. **Granularity:** every phase's implementation checklist has 3 to 8 items. Merge or split the
   others; keep one only with a reason, recorded for the summary.
3. **Ledger:** for each phase, `Requires` ⊆ baseline ∪ workspace ∪ the `Delivers` of every
   transitive blocker (walk the whole `Depends on` chain). A row no earlier phase delivers, or a
   provider that is not among its blockers, means re-ordering or re-slicing now.
4. **Standing invariants:** the first phase whose work touches an invariant's resource gets the
   checklist item "Add a regression test explicitly asserting `<invariant text>`, not solely via a
   generic completeness tool"; a documentation-only invariant is restated verbatim in that phase's
   `## Architectural decisions` instead.

## 4. Fields

- `parallel_group`: a short label; phases share one only when no dependency joins them and, by
  the Spec's Seams, they touch different modules. Every phase has one.
- `reversibility`: `two_way` when reverting the commits undoes it, `one_way` otherwise (a
  destructive migration, a published contract).
- `blast_radius`: `local` (one module and its tests), `module` (others call into it), `system`
  (a boundary, a contract, shared infrastructure). When unsure, take the more cautious value and
  say so in the summary.
- Leave `branch_from` out: each phase is cut from the feature's branch.

## 5. Create

For each phase, blockers first:

```
create_phase(parent_id=<feature>, title=..., description=<the plan below>,
  parallel_group=..., reversibility=..., blast_radius=..., afk=false (only an afk: false phase),
  checklists=[{"checklist_type": "implementation", "label": ...}, ...,
              {"checklist_type": "acceptance_criteria", "label": ...}, ...])
```

Then for each blocker `create_relation(relation_type="precedes", source_id=<blocker>,
target_id=<phase>)`, and rewrite each `### Depends on` section with the real keys
(`update_item(item_id, description)`, the whole description). The `- [ ]` lines stay in the
description too. A dependency on another feature is an open question (it lies outside your
reach). Catalog skills are not checked: you cannot commit to the feature's branch; say so.

## Plan template

```
# Plan: <phase title>

> <Feature Name>

> afk: false — <one-line reason> (only for a phase that cannot self-verify in a sandbox)

## Feature summary

What the feature is for, and that this is phase <n> of <N>; what the later phases cover.

## Architectural decisions

- **Routes**: ...
- **Schema**: ...
- **Key models**: ...

---

### Delivers

- table: ... | endpoint: ... | mcp-tool: ... | module: ... | package: ... | env: ... | service: ...

### Requires

- <artifact> — phase <n> | baseline (`<the check the agent runs>`) | feature workspace
- workspace: <name> at `<target>` (<kind>, <access>) — feature workspace

### What to build

The end-to-end behaviour of this slice, not layer by layer.

### Implementation checklist

- [ ] ...

### Acceptance criteria

Each asserts a fact directly (query the catalog, call the endpoint, read the artifact), never only
a tool's report of it.

- [ ] ...

### Depends on

- #<blocker key> | None — can start immediately

### User stories addressed

- User Story <n>

### Touches

- module: ... | boundary: ...
```
