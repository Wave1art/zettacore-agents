# State: handoff

You are an unattended coding agent in `/workspace/repo`. The work is committed, verified and reviewed. This state closes the item so the host can push, gate and hand it to a human reviewer.

## Do

1. Make sure every entry in `.zc/checklist.md` is `- [x]`; tick any you finished but did not mark, in the file and in the engine with `update_checklist_items`. If one is genuinely not done, leave it unticked and say why in the closing summary.
2. Check `git status` is clean: commit anything that belongs to the item, discard the rest.

## Ending this step

Your last action is one `end_step` call; then stop.

`end_step(signal="NO_MORE_TASKS", summary=...)`, with the closing summary exactly this shape (the engine posts it on the item):

```
## Phase complete
Commit: <short sha of HEAD>
Verification: <the verify.sh stages that ran and passed>
Stage summary: <one line per checklist entry: what changed and which test proves it>
Open: <anything wrong you found now, or "none">
```

If `end_step` is unavailable, post that summary with `post_comment` and end your last message with `<promise>NO MORE TASKS</promise>` then `STOP_REASON: no_more_tasks` on its own line.

## Rules

- The shift runs your session: it claims the item, pushes the branch, runs the gate, completes the run and moves the item to review. Never call `claim_item`, `complete_run`, `release_run`, `set_status`, `create_pr` or `mark_pr_ready` (the engine refuses them), and do not push.
- Never pause for human input.
- Do not change code in this state. If you find something wrong now, put it under `Open:` and still close.
- Do not move the item's status yourself; the host moves it to review when the gate is green.
