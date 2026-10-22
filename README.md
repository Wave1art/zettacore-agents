# Zettacore starter agents

The agent definitions that ship with [Zettacore](https://github.com/Wave1art/zettacore-combined):
one directory per agent under `agents/`, each holding `agent.yaml`, its state prompts and, for the
overseer, its authority table. A Zettacore install reads this repository as its **default agent
source** (ADR 0003): the engine syncs the definitions from the newest release tag it can run
(`latest`, ADR 0011), stores each one with its files, and the worker installs them into the
sandbox at provision. Nothing here is built
into a Zettacore image; changing an agent is a commit here and a sync there.

This repository has its own version (`v1.0.0` onwards, semver) and releases without Zettacore.
Each definition may declare `requires_engine` (a version range such as `">=2.0.0-rc.10"`): an
engine outside the range does not register it, and a source following `latest` stays on the
newest tag whose definitions all run on that engine. From `v1.3.0` the OpenCode definitions
require Zettacore `2.0.0-rc.10` or later, the first engine with `end_step`, and from `v1.4.0`
(the `-frontier` keys) and `v1.6.0` (the rest) `2.0.0-rc.12`, the first that checks out the
item's own branch. `v1.7.0` drops `skills` and `model.fallback_tier` from every definition
(Zettacore ADR 0032) and makes all of them, `reviewer` and `overseer` included, require
`2.0.0-rc.27`; an older engine following `latest` stays on `v1.6.0`. `v1.8.0` changes prompts
only, with the same engine range: every agent narrates one sentence of intent before its tool
calls, so a run's transcript reads as more than tool calls, and the checklist prompts say `.zc/`
is the host's and never committed. `v2.0.0` makes `reviewer` 3.0.0 an OpenCode agent in the reviewed change's
own environment (Zettacore ADR 0051) and requires `2.0.0-rc.48`; an older engine following
`latest` stays on `v1.8.0`. `v2.1.0` (`reviewer` 3.1.0) ends each axis with `end_step` carrying
the verdict as its `result`, as every agent finishes, instead of an answer file, and requires
`2.0.0-rc.49`; an older engine following `latest` stays on `v2.0.0`.

## Layout

```
agents/<key>/agent.yaml          # the definition; key equals the directory name
agents/<key>/prompts/<state>.md  # one prompt per state, plus remediation.md where remediation is on
agents/<key>/authorities.yaml    # overseer only: what it may decide without a person
schema/agent.schema.json         # JSON Schema for agent.yaml
LICENSE                          # MIT, the same licence as Zettacore
```

`schema/agent.schema.json` is generated from the Zettacore product repository by
`zc agents schema --write` (the worker's `zettacore_workers.agents.schema` model) and copied here;
when that model changes, the file must be copied again. Validate a checkout with the worker CLI
from a Zettacore clone: `uv run zc agents validate /path/to/zettacore-agents/agents`.

## The agents

Every definition names its model as a tier: the LiteLLM model group `zc/<tier>` the install
serves (`zc/standard` for every definition except `reviewer`, which names `zc/frontier`; every coding definition, the two `-frontier` keys included, runs on `zc/standard`; ADR 0008). A definition names one tier: fallbacks between models are LiteLLM's routing (ADR 0004), not part of the definition. The starter set needs only `zc/standard` and `zc/frontier`; none names `zc/cheap`. `profile: coder` may push to the item's
branch; `profile: gate` never pushes. A project enables a definition on a Temporal task queue
(Project settings > Agents, one definition per queue, ADR 0002); the work type's allocation then
routes items to that queue or to the definition by name.

| Key | Purpose | Runtime / profile | Serves | Tier (`zc/<tier>`) | Sandbox and runtime notes |
|---|---|---|---|---|---|
| `coding-worker` | Implements one phase through the five-state loop `resolve, implement, verify, review, handoff`: decides what to build from the item and its spec, builds it against `.zc/checklist.md`, runs the pinned verification contract with `zc-verify` as the gate does, reviews the diff as a senior colleague would, then closes the item for the host push and gate. One remediation pass after a red gate. | `opencode`, `coder`, unattended | `phase` items (allocation `queue:shift-coding`) on the queue it is enabled on, `shift-coding` by default | `standard` | 1 CPU, 2 GiB, `dockerd: true` (compose-based test suites); exit checks: `checklist_exists` after implement, `zc-verify` after verify (`bash .zcore/verify.sh` where the sandbox has no `zc-verify`), checklist complete after handoff; idle 20 m, gate 10 m, bootstrap 2 m |
| `coding-worker-frontier` | `coding-worker` kept under its older key for existing bindings: the same five-state loop, prompts and checks, now on the standard tier. | `opencode`, `coder`, unattended | `phase` items (allocation `queue:shift-coding`) on the queue it is enabled on | `standard` | As `coding-worker` |
| `coding-worker-attended` | The same five-state loop, but `interaction: attended`: every OpenCode permission prompt becomes a `question` in the inbox and the shift waits for the answer instead of allowing it once. | `opencode`, `coder`, attended | `phase` items, on its own queue (one definition per queue) | `standard` | As `coding-worker`; the open question holds the idle clock |
| `implement-only` | Single-state variant of `coding-worker`: plan, build, verify and close in one `implement` pass per iteration, gated on the pinned contract. The cheapest way to run a well-specified phase. | `opencode`, `coder`, unattended | `phase` items on the queue it is enabled on | `standard` | 1 CPU, 2 GiB, `dockerd: true`; one 60 m state, two attempts; idle 20 m, gate 10 m |
| `implement-only-frontier` | `implement-only` kept under its older key for existing bindings: the same single `implement` pass, now on the standard tier. | `opencode`, `coder`, unattended | `phase` items on the queue it is enabled on | `standard` | As `implement-only` |
| `bug-fixer` | Diagnoses one defect from the bug's `brief` and `repro` fields, reproduces it with a failing regression test, fixes the cause (not the symptom) and closes the item, gated on the pinned contract. | `opencode`, `coder`, unattended | `bug` items (allocation `agent:bug-fixer`) | `standard` | 1 CPU, 2 GiB, `dockerd: true`; one 60 m state, two attempts; idle 20 m, gate 10 m |
| `reviewer` | Reviews a change as an agent in the change's own environment (Zettacore ADR 0051), spec then standards in one sandbox, by the criteria of the attended `/zc:review` skill: `spec` (each acceptance criterion met, partly met, missing or cannot verify, with file:line evidence or a probe it ran; the Spec's requirements and Standing invariants; `reversibility` and `blast_radius` confirmed or challenged) and `standards` (every changed file against the project's conventions, lessons, style guides and `CONTEXT.md` vocabulary, each finding citing its rule; Minimal Impact, Simplicity First, No Laziness; the implementer's refactoring candidates re-checked). Reads a brief of every changed file, the Spec and the green gate, opens and runs what it needs, and ends with `end_step` carrying its verdict for the workflow to record. | `opencode`, `reviewer` | `review` items (allocation `agent:reviewer`) on queue `shift-review`; one `ReviewWorkflow` per round, checked out at the green gate's commit, with no push | `frontier` | The reviewed item's environment and dev stack (its own `sandbox` only without one); 45 m per axis, one attempt |
| `overseer` | Supervises a shift that stopped at an exit state: `diagnose` names the cause and the action in one sentence; `repair` writes a brief of at most five lines for the corrective pass. Relaunches or repairs within `authorities.yaml` (one relaunch, repair allowed; `contract_integrity` and `decision_timeout` always escalate), else escalates to the inbox. | `llm-loop`, `gate` | Started by `ShiftWorkflow` on queue `shift-oversight` when the project enables `overseer`; no item allocation | `standard` | 1 CPU, 1 GiB, no dockerd; 10 m per state, one attempt; a decision escalated to a person times out after 24 h |

The OpenCode definitions share `prompts/remediation.md` (one corrective pass after a red gate)
and the permission map `bash: {"*": allow, "rm -rf *": deny}`. Every prompt stays around 400
words.

### How an OpenCode agent talks to its shift

Every OpenCode prompt starts with `get_step` (where the shift is: this step and the next, how
earlier steps ended, the handoff pointers) and ends the step with one `end_step` call:

- `DONE`: this step's work is finished; the shift runs its exit checks and the next step.
- `NO_MORE_TASKS`: the item's work is finished (the last step); the shift pushes, gates and
  closes the item. The summary is the closing comment.
- `BLOCKED`, `SETUP_ERROR`, `CONTEXT_ROT`: the agent cannot go on; the summary says what a person
  must do and `root_cause` why. The engine posts the summary on the item.

The worker reads the call from OpenCode's event stream and acts on it; the `<promise>` markers
stay as the fallback when the tool is unavailable. The shift pushes the branch after every step,
so the prompts ask the agent to commit as it goes and never to push. `reviewer` ends with `end_step`
too, its verdict in `result`, on a token whose only tool that is; `overseer` runs bounded
LiteLLM calls without them and keeps its markers. All of them run the default Zettacore sandbox image; none sets `sandbox.image`.

Per-agent evals are not part of this release; the definitions are validated against the schema
and exercised by Zettacore's own journeys.

## How an install uses this repository

A fresh Zettacore install with no agent sources links this repository on the engine's first jobs
tick: it creates the repository row for `AGENT_SOURCE_DEFAULT_URL`
(`https://github.com/Wave1art/zettacore-agents`) in the root organisation, adds an agent source
at `AGENT_SOURCE_DEFAULT_REF` (`latest` from Zettacore `2.0.0-rc.10`) with path `agents`, and
syncs it. The repository is
public, so the sync reads it anonymously; no GitHub token or credential is needed. Admin > Agents
shows the source with one row per definition (`registered`, `unchanged` or `error`); Sync runs it
again on demand, and the jobs process syncs every ten minutes.

A source's ref is a branch (`main`), a tag (`v1.3.0`), `latest` (the newest release this engine
can run) or a tag pattern (`v1.*`, the newest release it matches). Change it with Edit in Admin >
Agents > Sources (or `PATCH /api/v1/agent-sources/{id} {"ref": "latest"}`); the source is read in
full on the next sync. Each synced change registers a new definition version; projects on the
`latest` policy pick it up, pinned projects keep theirs. An install created before `2.0.0-rc.10`
keeps the tag it was linked at until its ref is edited.

## Forking and linking your own source

Fork this repository (or start an empty one with the same layout), edit or add definitions under
`agents/<key>/`, then link it:

1. Add the fork as a repository (Project settings > Repositories, or
   `POST /api/v1/repositories {"url": "https://github.com/you/your-agents"}`). A private fork needs
   a `vcs_token` credential on a project linked to it; a public one needs nothing.
2. Admin > Agents > sources: add the repository with its `ref` and `path` (`agents`); or
   `POST /api/v1/agent-sources {"repository_id": ..., "ref": "main", "path": "agents"}`.
3. Sync (`POST /api/v1/agent-sources/{id}/sync`) and enable the definitions you want on a queue in
   Project settings > Agents.

A definition in your source is validated against the schema on every sync; the console shows the
reason for any row in error (invalid YAML, a missing prompt, a key owned by another source).

Authoring reference: `docs/agents/authoring.md` in the Zettacore repository (states, checks,
custom sandbox images, versions and version policy).

## When two sources ship the same key

Sources are ordered (Zettacore ADR 0033, engine 2.0.0-rc.27 and later). A key belongs to the
highest *enabled* source in that order that ships a valid definition of it; a source lower down
that ships the same key shows it as **shadowed**, naming the repository that provides it.

So to replace a starter agent with your fork's version under the same key (`coding-worker`):

1. Link your fork as an agent source (Admin > Agents).
2. Move it above this repository in the source order. Its next sync takes the key; projects on
   the `latest` version policy run your fork's newest version, and a project pinned to a version
   keeps it.
3. To go back, move it down or disable it: the key returns to this repository.

Disabling a source hands its keys to the next enabled source that ships them; deleting one leaves
its keys to whichever source ships them next. Registered versions are never removed, because runs
reference them. Renaming your definition (`acme-coding-worker`) avoids the question altogether.
