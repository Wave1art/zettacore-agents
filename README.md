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
require Zettacore `2.0.0-rc.10` or later, the first engine with `end_step`.

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
serves (`zc/standard` with `zc/cheap` as the fallback here, and `zc/frontier` for `coding-worker-frontier` and `implement-only-frontier`; ADR 0008). `profile: coder` may push to the item's
branch; `profile: gate` never pushes. A project enables a definition on a Temporal task queue
(Project settings > Agents, one definition per queue, ADR 0002); the work type's allocation then
routes items to that queue or to the definition by name.

| Key | Purpose | Runtime / profile | Serves | Tiers (`zc/<tier>`, then fallback) | Sandbox and runtime notes |
|---|---|---|---|---|---|
| `coding-worker` | Implements one phase through the five-state loop `resolve, implement, verify, review, handoff`: decides what to build from the item and its spec, builds it against `.zc/checklist.md`, runs the project's own `.zcore/verify.sh`, reviews the diff as a senior colleague would, then closes the item for the host push and gate. One remediation pass after a red gate. | `opencode`, `coder`, unattended | `phase` items (allocation `queue:shift-coding`) on the queue it is enabled on, `shift-coding` by default | `standard`, `cheap` | 1 CPU, 2 GiB, `dockerd: true` (compose-based test suites); exit checks: `checklist_exists` after implement, `bash .zcore/verify.sh` after verify, checklist complete after handoff; idle 20 m, gate 10 m, bootstrap 2 m |
| `coding-worker-frontier` | `coding-worker` on the frontier tier: the same five-state loop, prompts and checks, for the phases that need the strongest model. Enable it instead of `coding-worker` on a queue, or on its own queue, since one definition serves a queue per project. | `opencode`, `coder`, unattended | `phase` items (allocation `queue:shift-coding`) on the queue it is enabled on | `frontier`, `standard` | As `coding-worker` |
| `coding-worker-attended` | The same five-state loop, but `interaction: attended`: every OpenCode permission prompt becomes a `question` in the inbox and the shift waits for the answer instead of allowing it once. | `opencode`, `coder`, attended | `phase` items, on its own queue (one definition per queue) | `standard`, `cheap` | As `coding-worker`; the open question holds the idle clock |
| `implement-only` | Single-state variant of `coding-worker`: plan, build, verify and close in one `implement` pass per iteration, gated on the pinned contract. The cheapest way to run a well-specified phase. | `opencode`, `coder`, unattended | `phase` items on the queue it is enabled on | `standard`, `cheap` | 1 CPU, 2 GiB, `dockerd: true`; one 60 m state, two attempts; idle 20 m, gate 10 m |
| `implement-only-frontier` | `implement-only` on the frontier tier: the same single `implement` pass, for well-specified phases that still need the strongest model. Enable it instead of `implement-only` on a queue, or on its own queue. | `opencode`, `coder`, unattended | `phase` items on the queue it is enabled on | `frontier`, `standard` | As `implement-only` |
| `bug-fixer` | Diagnoses one defect from the bug's `brief` and `repro` fields, reproduces it with a failing regression test, fixes the cause (not the symptom) and closes the item, gated on the pinned contract. | `opencode`, `coder`, unattended | `bug` items (allocation `agent:bug-fixer`) | `standard`, `cheap` | 1 CPU, 2 GiB, `dockerd: true`; one 60 m state, two attempts; idle 20 m, gate 10 m |
| `reviewer` | Reviews a change on one axis per run: `standards` (correctness, tests, naming, structure, error handling, security) or `spec` (each acceptance criterion met or not). Records findings as `review` checklist entries, a `## Review verdict:` comment and the `review_verdict` evidence. | `llm-loop`, `gate` | `review` items (allocation `agent:reviewer`) on queue `shift-review`; a `ReviewWorkflow` clones for the diff and never pushes | `standard`, `cheap` | 1 CPU, 1 GiB, no dockerd; bounded LiteLLM calls, 15 m per axis, one attempt |
| `overseer` | Supervises a shift that stopped at an exit state: `diagnose` names the cause and the action in one sentence; `repair` writes a brief of at most five lines for the corrective pass. Relaunches or repairs within `authorities.yaml` (one relaunch, repair allowed; `contract_integrity` and `decision_timeout` always escalate), else escalates to the inbox. | `llm-loop`, `gate` | Started by `ShiftWorkflow` on queue `shift-oversight` when the project enables `overseer`; no item allocation | `standard`, `cheap` | 1 CPU, 1 GiB, no dockerd; 10 m per state, one attempt; a decision escalated to a person times out after 24 h |

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
so the prompts ask the agent to commit as it goes and never to push. `reviewer` and `overseer`
run bounded LiteLLM calls without the engine's MCP tools and keep their markers. All of them run the default Zettacore sandbox image; none sets `sandbox.image`.

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

## Keys are unique across sources

The first source to register a key owns it. The engine's sync refuses the same key from a second
source and reports it on that source's row, in the words of
`engine/src/zettacore_engine/services/agent_sources.py`:

```python
owner = await session.get(AgentDefinition, layout.key)
if owner is not None and owner.source_id is not None and owner.source_id != source.id:
    return DefinitionSyncResult(
        key=layout.key,
        status="error",
        message=f"{layout.key!r} is already provided by agent source {owner.source_id}",
    )
```

So a fork that keeps the key `coding-worker` collides with this repository while both are linked.
Either rename your definition (`acme-coding-worker`) or disable this source first; per-source
priority is a follow-up.
