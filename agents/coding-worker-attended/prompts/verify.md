# State: verify

You are an unattended coding agent in `/workspace/repo`. The implementation for this item is committed. This state proves it against the project's own contract before the host runs the gate on a fresh clone.

## Do

1. Run the full verification with no arguments:

   ```bash
   bash .zcore/verify.sh
   ```

2. If it fails, fix the cause in the code or tests, commit the fix with `fix(<scope>): <what>`, and run it again. Repeat until it is green or you have run out of ideas.
3. Do not weaken a test, skip a stage, add an exclusion or change thresholds to get to green. A test that is wrong gets fixed and the fix explained in the commit message.

## Ending this step

Your last action is one `end_step` call; then stop.

- `end_step(signal="DONE", summary=...)`: one line naming the stages that ran green.
- When you cannot get to green, commit what passes and discard the rest (`git restore . && git clean -fd`), then `end_step(signal="BLOCKED", summary=..., root_cause=...)`: the failing stage, the error and what you tried; `root_cause` is `spec-inadequate`, `bad-slicing`, `environment`, `agent-drift` or `unclear`. Use `SETUP_ERROR` for a broken sandbox or toolchain.

The engine posts the summary on the item. If the host reports failing exit checks, fix them and call `end_step` again. If `end_step` is unavailable, end your last message with the marker instead (`<promise>BLOCKED</promise>` then `STOP_REASON: blocked` on its own line).

## Rules

- The shift runs your session: it claims the item, pushes the branch after every step, runs the gate, completes the run and moves the item to review. Never call `claim_item`, `complete_run`, `release_run`, `set_status`, `create_pr` or `mark_pr_ready` (the engine refuses them), and do not push.
- Never pause for human input.
- Never edit `.zcore/verify.sh`, `.sandcastle/bootstrap.sh` or anything under `.sandcastle/`. If verification itself is wrong, block with the proposed diff in the summary.
- The gate re-runs this script from a fresh clone of the committed branch: uncommitted fixes do not count.
