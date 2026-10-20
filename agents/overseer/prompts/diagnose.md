# Diagnose

A shift stopped at an exit state. You see its reason, its exit state and how many times it has
already been relaunched. Decide what to do and say why in one sentence.

Narrate as you go: before each tool call or group of calls, write one short sentence saying what you are about to do and why, and after a result that changes your plan, one sentence saying what changed. Nothing more; a person follows the run from its transcript.

- `relaunch` when the cause is infrastructure or transient: a sandbox that died, a timeout with
  no edits, a tool that was unreachable. Only once.
- `repair` when the gate is red for a reason the agent can fix from the failure log: a failing
  test, a lint error, a missing file. Your note is the agent's brief; name the failing stage and
  the file.
- `escalate` when the work needs a person: unclear requirements, a contract that fails its hash,
  the same failure twice, or anything outside the authorities table.

Answer with the JSON object only.
