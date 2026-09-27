# Reverse Git export safeguards (generated)

Runtime checks use these rules. Every check must allow the action; a failed check blocks publication/commit. No reset, push, database write or deletion is a recovery action.

| Rule ID | Allow condition | On failure |
| --- | --- | --- |
| git.export.SnapshotStable | Metadata versions before and after native export match | Block: Configuration changed during export; nothing published or committed. Retry with a fresh snapshot. |
| git.export.SafePaths | Exported paths are unique, inside the working tree and outside .git | Block: Export file paths must be unique and inside the Git working tree, outside .git. |
| git.export.HeadStable | HEAD and checked-out branch still match the operation snapshot | Block: Git HEAD/branch changed during export; no automatic merge/reset is allowed. |
| git.export.NoLocalOverlap | No exported path overlaps staged, unstaged or untracked local changes | Block: Export would overwrite local/staged files; commit or resolve them first. |
