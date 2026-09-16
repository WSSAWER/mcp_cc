# Synchronization decisions (generated)

Git fetch and commit comparison never reserve a database connection. An unchanged commit completes without inventory, ibcmd or Designer calls. Only a changed commit proceeds to inventory validation and reservation; a busy database defers that update. Catch-up after import retains the same lease until apply.

| Rule ID | Condition | Result |
| --- | --- | --- |
| inventory.explicit | Explicit refresh or unknown component | Refresh once on an explicit request or missing component error. |
| inventory.connection | First use/startup, new connection, or changed connection settings | Refresh once; do not poll inventory on sync ticks. |
| inventory.error | Previous discovery failed | Retry discovery on the next actual request, preserving the previous inventory on failure. |
| inventory.cache | None of the above | Use in-memory inventory. |

## Git preflight

| Rule ID | Required condition | Failure |
| --- | --- | --- |
| git.stable_head | HEAD still equals the fetched commit | Block: Git HEAD changed during preflight; no DB connection reserved. |
| git.clean_source | No uncommitted source changes | Block: Git source is dirty; no DB connection reserved. |
| git.linear_history | History has not diverged from last success | Block: Git history diverged; no DB connection reserved. Review source/history before resuming. |
| git.needs_database | Initial/full load, or at least one loadable changed file | Otherwise acknowledge commit without DB calls/lease. |

## Failed Git import recovery

| Rule ID | Condition | Result |
| --- | --- | --- |
| recovery.cancelled | Operation cancelled/interrupted | Pause: database state requires review. |
| recovery.applied | Apply succeeded but a later step failed | Pause: editable reset cannot undo an applied database change. |
| recovery.no_reset | No imported state or reset did not exit with code 0 | Pause: rollback is not confirmed. |
| recovery.watch | Import/apply failed, reset exited 0, apply never succeeded, not cancelled | Enabled Git job keeps polling; retain failure and last-successful version. |
| recovery.same_commit | Fetched hash equals the rejected hash | Wait for a new commit; no inventory, database lease or native command. |
| recovery.new_commit | Fetched hash differs; source/connection unchanged | Validate source, then reserve DB and import; clear failure only after successful apply/check. |
| recovery.poll_error | Git-only check fails while a safe checkpoint exists | Retain checkpoint and retry polling at the configured interval. |
| recovery.restart | Persisted safe checkpoint; no interrupted native load | Restore Git polling. Old errors without rollback evidence remain paused. |
| recovery.disabled | Job was stopped | No polling or automatic resumption. |
