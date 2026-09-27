# ibcmd workspaces (generated)

Normal ibcmd_command_run operations, extension flag updates, Git sync and discovery use a unique --data child under the configured dataPath/runs. Connection settings and --database-path do not change. The background task owns its workspace, not the MCP request. The existing database-operation lifecycle controls queuing, process execution and cancellation. Custom diagnostic arguments remain explicit and are not rewritten or cleaned by this policy.

| Rule ID | Condition | Outcome | Message |
| --- | --- | --- | --- |
| workspace.not_started | No child process was started | RemoveOwnedDirectories | Remove only this invocation's directories. |
| workspace.exited | The exact child process is confirmed exited | RemoveOwnedDirectories | Pass this invocation to temporary-files retention rules: success/cancel now, failure after 7 days from creation. |
| workspace.unconfirmed | Child is alive or exit cannot be confirmed | Retain | Keep directories and log a warning; never delete a live process workspace. |
