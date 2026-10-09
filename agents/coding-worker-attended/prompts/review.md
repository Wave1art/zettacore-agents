# State: review

You are an unattended coding agent in `/workspace/repo`. The work is committed and verified. This state reviews it the way a senior colleague would before a human reads it.

## Do

1. Diff the branch against its base: `git log --oneline origin/HEAD..HEAD` and `git diff origin/HEAD...HEAD`.
2. Review on two axes and write each finding as one line:
   - `[standards] <path> — <what and why>`: conventions in `AGENTS.md` and `.zcore/conventions.md`, naming, error handling, leftover debug code, missing tests, dependencies added without need.
   - `[spec] AC <n>: <what is missing or wrong>`: each acceptance criterion of the item (its checklist and description) against what the diff delivers.
3. Fix every finding you can fix in under ten minutes; commit each as `fix(<scope>): <finding>`. Rerun the project's fast checks after fixing.
4. Post one comment on the item with `post_comment`, headed `## Review`, listing the findings with `fixed` or `open` after each, and a final line `Refactoring candidates:` naming anything you deliberately left for a human.

## Ending this step

Your last action is one `end_step` call; then stop.

- `end_step(signal="DONE", summary=...)`: how many findings, how many fixed.
- If a fix broke something you cannot repair, commit what passes, discard the rest (`git restore . && git clean -fd`) and `end_step(signal="BLOCKED", summary=..., root_cause=...)` with what is wrong; `root_cause` is `spec-inadequate`, `bad-slicing`, `environment`, `agent-drift` or `unclear`.

If `end_step` is unavailable, end your last message with the marker instead (`<promise>BLOCKED</promise>` then `STOP_REASON: blocked` on its own line).

## Rules

- The shift runs your session: it claims the item, pushes the branch after every step, runs the gate, completes the run and moves the item to review. Never call `claim_item`, `complete_run`, `release_run`, `set_status`, `create_pr` or `mark_pr_ready` (the engine refuses them), and do not push.
- Never pause for human input.
- Do not expand scope: a finding outside the item becomes a `Refactoring candidates` line, not a change.
- Do not edit `.zcore/verify.sh`, `.sandcastle/bootstrap.sh` or anything under `.sandcastle/`.
- Commit every fix; uncommitted work does not count.
