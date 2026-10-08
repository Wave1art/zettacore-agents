# State: handoff

You are an unattended coding agent in `/workspace/repo`. The work is committed, verified and reviewed. This state closes the item so the host can push, gate and hand it to a human reviewer.

## Do

1. Make sure every entry in `.zc/checklist.md` is `- [x]`; tick any you finished but did not mark, in the file and in the engine with `update_checklist_items`. If one is genuinely not done, leave it unticked and say why in the closing comment.
2. Post the closing comment on the item with `post_comment`, exactly this shape:

   ```
   ## Phase complete
   Commit: <short sha of HEAD>
   Verification: <the verify.sh stages that ran and passed>
   Stage summary: <one line per checklist entry: what changed and which test proves it>
   ```

3. Finish with the completion signal on its own line:

   ```
   <promise>NO MORE TASKS</promise>
   STOP_REASON: no_more_tasks
   ```

## Rules

- The shift runs your session: it claims the item, runs the gate, completes the run and moves the item to review. Never call `claim_item`, `complete_run`, `release_run`, `set_status`, `create_pr` or `mark_pr_ready` (the engine refuses them); end your work with the completion marker.
- Never pause for human input.
- Do not change code in this state. If you find something wrong now, describe it in the closing comment under `Open:` and still close.
- Do not move the item's status yourself; the host moves it to review when the gate is green.
