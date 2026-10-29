---
name: readiness-check
description: Check a plan in the engine before it goes to review — every item complete, every dependency recorded, every requirement covered, every workspace target declared — and give the verdict the plan summary opens with.
---

# The readiness check

Run it against the engine as it stands, never against your draft: read each item with `get_item`
(`include=["checklists","relations"]`).

## For every phase

1. Its description follows the plan template: `# Plan:`, `## Feature summary`, `## Architectural
   decisions`, `### Delivers`, `### Requires`, `### What to build`, `### Implementation
   checklist`, `### Acceptance criteria`, `### Depends on`, `### User stories addressed`,
   `### Touches`.
2. It has 3 to 8 implementation checklist entries and at least one acceptance criterion, each an
   observable fact.
3. `parallel_group`, `reversibility` and `blast_radius` are set.
4. Each `### Depends on` line names a real key, and a `precedes` relation from that key exists; no
   chain of relations comes back to the phase.
5. The ledger holds with real keys: each `Requires` row is in the baseline files, a declared
   workspace row, or a blocker's `Delivers`.
6. Each `workspace:` row in `Requires` shows in `list_workspace` on the phase, at that target, with
   credential `present` or `not_needed`. One that does not is **missing**.
7. An `afk: false` line has its reason, and the phase was created with `afk=false`.

## For the plan as a whole

- **Coverage:** every P0 requirement and work-implying decision of the Spec maps to a checklist
  entry, or is listed as a gap.
- **Tasks and bugs:** a body that says what and why and when it is done, both checklists, a bug's
  `brief` and `repro`.
- **Initiatives:** the open decisions match `## Open questions`, at most 12.
- **Reach:** nothing outside the target and the items under it was changed, and nothing was
  readied, started or claimed.
- **Environment:** the target's repository has `.zcore/verify.sh` (else its phases cannot pass a
  gate) and, when the Spec adds packages or services, a dev container that provides them or a
  phase that adds them. Say which.
- **Send-back:** every point of the send-back comment is answered, by a change or a reason.

## Verdict

- `ready to accept`: nothing is missing.
- `ready once <what>`: only things a person does stand between the plan and its start (credentials
  to enter, a dependency to record, an `afk: false` phase to assign).
- `not ready — <why>`: the plan has a hole you could not mend (a requirement nothing covers, a
  ledger row nothing delivers, a refused declaration it relies on).
