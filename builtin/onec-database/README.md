# 1C Database Manager MCP

Built-in MCP Control Center profile for named 1C database/configurator bindings.

Control Center starts one persistent built-in Streamable HTTP mediator with `McpControlCenter.exe --onec-db-http`. Client MCP sessions may connect and disconnect without owning or terminating its active 1C operation. This folder is a local install source marker for the Control Center profile.

Runtime project settings are stored outside this source folder:

```text
mcps\onec-database\.generated\projects.json
```

## Validated database structure

The owner loads the configuration into persistent C# database instances. JSON is
storage, not a command template. Each instance owns its bindings, validated adapter
access, connection lease, execution queue, operation/PID registry and sync-state
lock. HTTP sessions and CLI callers use the same instance. Aliases of the same
server/IB share it; conflicting SQL database or cluster/IB identifiers in aliases
block execution instead of silently selecting one. Different bases have independent
queues.

Set normal connection data with `upsert_project`:

- Designer/IB: `configuratorPaths`, `infobaseServer`, `infobaseName`,
  `infobaseUser`, `infobasePassword`; a file base uses `infobaseFilePath` instead
  of server/name. The server may include its port.
- ibcmd/SQL database: `ibcmdPath`, `dataPath`, `dbms`, `databaseServer`,
  `databaseName`, `databaseUser`, `databasePassword`. These describe the database
  behind the IB, not its cluster alias. They are never inferred from an IB name.
  File-mode ibcmd uses `infobaseFilePath` plus `dataPath`, with an empty `dbms`
  and no SQL connection. Its command explicitly includes `--database-path`;
  it never silently loads the unrelated default `data/db-data` database.
  The mediator requires an explicit data directory to avoid accidentally sharing
  the platform's default standalone-server directory between different bases.
- Repository: `repositoryAddress`, `repositoryUser`, `repositoryPassword`.
- Cluster administration: `racPath`, `rasHost`, `rasPort`, `clusterId`,
  `clusterUser`, `clusterPassword`, `infobaseId`.

The instance validates path resolution, file/directory kind, executable existence,
address shape, required fields and adapter relationships. Relative paths resolve
against the Control Center installation, not the process's current directory.
An invalid adapter cannot construct a normal executable command. Missing
executables are rechecked before launch. Use `get_project_actions` for readiness
and validation errors; `check_project_status` explicitly refreshes filesystem
validation as well as its network probes. `get_project_runtime` reports the live
instance ID, binding errors, validation errors, lease and operations. Static
validation is not proof of successful database authentication or a successful
native import; those outcomes are verified from the actual operation and its log.

Commands run against an immutable validated settings snapshot. Settings cannot be
changed while the same database has active requests or queued/running operations.
Changing an IB address or removing its binding additionally requires closing the
lease. Updating an idle binding preserves the instance. Direct edits to
`projects.json` are read on the next owner start; use `upsert_project` for live
updates. Unknown or incomplete settings can be inspected and corrected without
executing a database command.

Old `DesignerConnectionArguments`, `InfobaseConnectionString` and
`IbcmdConnectionArguments` are migration inputs only. They are parsed into fields,
then removed from the stored JSON. Explicit fields take precedence when both are
present. Recognized ibcmd aliases become canonical fields and commands emit
`--database-name`. Unsupported legacy options are retained for correction, but
are not silently forwarded to normal commands. Raw strings cannot override
validated connection fields.

For standard `run_ibcmd` / `build_ibcmd_arguments`, supply `operation` plus named
`extension`, `sourcePath` (import), `outputPath` (export), and `force` (apply)
where needed. The `arguments` array is accepted only for `operation=custom`.
Normal repository actions likewise use named fields; arbitrary appended
`arguments` require `operation=custom`. Custom is an explicitly unstructured
diagnostic escape hatch, not the normal database workflow.

Implementation layers are `OneCDatabaseModels` (persistence DTOs),
`OneCDatabaseRuntime` (database lifetime/ownership), `OneCDatabaseValidation`
(validated bindings and paths), `OneCDatabaseConnections` (connection entities
and migration), `OneCDatabaseRequests` (typed operation input),
`OneCDatabaseCommands` (argument builders), and `OneCDatabaseProcessCommand`
(direct OS argument vectors). Git sync and CLI scripts invoke these C# workflows;
scripts do not carry their own SQL/Designer connection command strings.

## Execution owner and diagnostics

Start `onec-database` in Control Center before using its CLI entry points. CLI
extension/sync/status commands connect to that running MCP through a local named
pipe; they never start an independent loader or keep a separate operation registry.
If the owner is absent, the command fails without starting 1C. The pipe permits
the owner's Windows account, local administrators and SYSTEM, not network users.
Existing generated `sync.ps1` / `status.ps1` scripts use this same route.

Project aliases with the same normalized IB connection share a lease and execution
queue, independently of the executable, user or extension. Server/ref, file path,
RAC identity and database connection fields are used in that order when available.
Keep aliases consistent: unrelated DNS aliases or different connection descriptions
cannot always be inferred to represent the same IB. Close the logical connection
before changing its IB address or removing the project. This is still a logical
lease, not a permanently open Designer process.

Non-zero child exit codes are failures. Synchronous MCP calls return `isError`;
background failures expose `exitCode`, `lastError` and `logPath`. Extension install
holds one queue lock across LoadCfg/UpdateDBCfg and never executes the second step
after a failed first step. Designer executions capture the platform's `/Out` log,
including UTF-8, UTF-16 and Windows-1251 output, because stdout alone is insufficient.
Use a Designer version compatible with the target server. Empty quoted command
arguments are preserved; `file:` repository URLs are converted to Windows paths.

Passwords and credential-bearing connection fields are encrypted with Windows
DPAPI for the account running the MCP. Existing plaintext settings are migrated
when the persistent owner starts. Moving to another computer or service account
requires re-entering those protected values; ciphertext cannot serve as a portable
password. Responses, operation arguments and new logs redact credentials. When
updating settings, omit unchanged credential/connection fields instead of submitting
`<redacted>` values. Historical logs are not rewritten automatically.

The MCP supports:

- a project-first workflow: call `list_projects`, select a binding, then call `get_project_actions` to see which Designer, repository, ibcmd, and RAC actions are actually configured;
- ready repository actions: `repository_get_objects`, `repository_update_objects`, `repository_lock_objects`, `repository_unlock_objects`, and `repository_commit_objects`; these accept a project name and action data while executable, infobase, repository address, and credentials come from the saved project;
- `ibcmd infobase config` operations: `generation_id`, `reset`, `apply`, `check`, `import`, `export`, `custom`;
- extension installation/update through `1cv8 DESIGNER`: `/LoadCfg`, `/UpdateDBCfg`, `-Extension`;
- extension flag updates through `ibcmd extension update`: `active`, `safe-mode`, `unsafe-action-protection`;
- 1C Designer configuration repository operations: `create`, `add_user`, `unbind_cfg`, `update_cfg`, `dump_cfg`, `report`, `lock`, `unlock`, `commit`, `set_label`, `custom`;
- recursive export from the information-base configuration: `export_infobase_object_recursive` discovers the metadata tree and exports the root plus its child forms, layouts, requisites, commands, and other child metadata as one managed operation;
- loading selected root objects into the information-base configuration: `infobase_load_objects` discovers the root XML and all child XML/BSL files in a hierarchical dump, creates the `listFile`, runs `/LoadConfigFromFiles`, and optionally `/UpdateDBCfg`;
- managed Git synchronization of complete configuration and extension XML dumps, with a durable last-successful commit and failure journal;
- RAS/RAC administration through the platform `rac.exe`: discovery of clusters and infobases, listing sessions for one configured infobase, and explicit termination of selected session UUIDs;
- optional explicitly bounded command execution with `timeoutSeconds`; Git synchronization has no time limit by default because a valid large import may run for hours;
- managed process operation tracking: `list_operations` shows operation id, live PID, status, elapsed time, last output time, CPU time, working/private memory, thread count and log path; `cancel_operation` terminates a running operation and its process tree;
- background execution for long operations: ready repository and load actions always return immediately with `operationId`; Advanced fallback commands support the explicit `background` option;
- safe argument-array command construction without shell.

For partial repository operations, pass `objects`, an array of root metadata names. Omit `recursive` in normal calls: `Configuration` / `Конфигурация` selects only the root; other named roots include descendants. Only an explicit `recursive=true/false` overrides this policy for every selected root. Configuration uses the platform's special XML element internally; callers use the same name list for all types. Lock/unlock/commit require a nonempty selection. `objectsPath` is an advanced XML alternative, parsed and validated into the same internal model, never forwarded unchanged. Invalid XML, unknown fields and operation-inapplicable parameters fail before launching Designer. `comment` is accepted only by commit/set_label, never lock. Examples of root names:

```text
Обработка.Потребности_ТОИР
Справочник.Номенклатура
Документ.ЗаказПокупателя
```

`dump_cfg` exports a complete repository configuration and does not accept an object selection.

## Normal object change workflow

The normal flow does not require constructing a Designer command line:

1. Select a saved project with `list_projects` and inspect available methods with `get_project_actions`.
2. If the object comes from the repository, call `repository_get_objects`, then `repository_lock_objects` with full root names. Use `repository_update_objects` for a normal repository update.
3. Export from the information-base with `export_infobase_object_recursive` when a hierarchical working dump is needed.
4. Edit the resulting XML/BSL files with a filesystem or Git tool. Editing source text is intentionally not hidden inside a database command.
5. Call `infobase_update_files` with `project`, `connectionType="designer"`, `files` and `allowExecution=true`. Pass paths to edited files or complete root XML files. Add new roots separately with `infobase_add_objects`.
6. Call `repository_commit_objects` to commit the selected roots, or `repository_unlock_objects` to release them without a commit.

`run_repository_command`, `build_repository_arguments`, `run_ibcmd`, and `build_ibcmd_arguments` are Advanced fallback tools for operations not represented by a ready action. They are not required for ordinary get/lock/load/commit work.

### Designer: update files or add new root objects

Both commands use the saved, validated database binding and its Designer executable. `files` are paths **on the MCP machine**, in hierarchical Designer XML layout, not client-machine paths or inline content. Upload/copy files first when using a remote MCP. A root XML selects that root and its companion directory recursively. A BSL/child file selects only that file for updates; its owning root XML must still be present alongside the hierarchical tree for identity validation. Adding a root always selects its complete supplied bundle.

```json
{"project":"MyProject","connectionType":"designer","files":["work/CommonModules/MyModule/Ext/Module.bsl"],"allowExecution":true}
```

`infobase_update_files` rejects the entire request if any root is missing in the current configuration or its UUID differs. `infobase_add_objects` accepts only new roots and rejects mixed new/existing roots and UUID collisions. **Pass only the new object files. Do not obtain or edit Configuration yourself:** caller-supplied `Configuration`, `Configuration.xml`, root `Ext` files and whole configuration-dump directories are rejected before any Designer/capture command.

For addition, the MCP exports the current Configuration and metadata index. The root-only `/DumpConfigToFiles -listFile` selection contains `Configuration`, so Designer also supplies its external properties. The MCP preserves the **complete exported bundle** (`Configuration.xml` plus every `Ext` file, recursively, including modules, interfaces, binary files and parent `.cf` files). There is no hardcoded file count or extension filter. It copies that bundle byte-for-byte into a private load package, edits only the manifest entries and adds the new object bundles. New entries go at the **end of their existing type group** in `ChildObjects`; an absent group is inserted in metadata type order. They are not appended indiscriminately at the end of Configuration. A second export rejects concurrent manifest or external-property changes before loading. Caller files are not changed; changed inputs during preparation abort the operation.

`infobase_add_objects` automatically uses the selected project's cached repository settings. When any repository settings are present, they must be valid; the workflow first captures only Configuration (nonrecursive), then performs the normal preparation/load sequence. Capture failure prevents loading. All Designer steps use the saved repository credentials. No repository configured means no capture and no extra prompt. Invalid or incomplete settings never silently select the no-repository path. No automatic commit or unlock occurs: after capture, the lock remains on success or failure until explicitly reviewed and committed/released.

Repository setting state is built with the in-memory database instance at configuration load and refreshed after any saved profile change (including credentials). `get_project_runtime.repository` exposes `not_configured`, `configured`, or `invalid_settings`, the check time and `source=saved_project_settings`. This is not proof of live repository authentication or discovery of an unconfigured IB binding (`liveAccessVerified=false`). Repository operations still fail on actual access errors; availability is not guessed from a nonempty path.

The generated load list includes the full Configuration bundle first and then the new objects; **`-partial` alone is not protection against losing Configuration's omitted external properties**. Existing-object updates do not load Configuration or its external files. The MCP verifies the root set and configuration UUID after loading. By default it then updates the database configuration with dynamic update disabled, warnings treated as errors, and permission to terminate blocking sessions. **Designer database update applies all pending changes of the selected editable configuration, including changes that predate this package.** Optional `terminateSessions=false` prohibits forced termination; `updateDatabase=false` changes only the editable configuration. `extension` selects an extension; `configuratorIndex` selects a saved Designer path. Platform support/repository locks remain enforced: the MCP does not bypass support restrictions or acquire someone else's locks.

The response is an operation, not a completion claim. By default there is no total-duration cutoff (`timeoutSeconds=0`); the owned loader PID/resource watchdog and explicit cancellation still apply. Set a positive timeout only when a total deadline is intended. Follow `list_operations`, `list_command_logs`/`search_log`, and `get_log_file`. The exact transmitted file list, byte counts and SHA-256 hashes are recorded as `LOAD FILE` entries in the downloadable command log. After successful load, verification and optional database update, the private `.generated/designer-files/<operationId>` tree is deleted (exports, index, load package and selection lists). Original caller files and permanent logs are never deleted. Failed operations retain their workspace for diagnosis; the log identifies its path. A successful load followed by a failed database update is **not a rollback**; inspect the native log before retrying. The older `infobase_load_objects` uses the same guarded update workflow and cannot add roots.

Verification: run `.test/smoke-all.ps1` with `smoke-onec-database-designer-files.ps1` for isolated process-level tests. The opt-in `smoke-onec-database-designer-live.ps1` uses `MCP_DESIGNER_TEST_MODE=probe` for read-only connection/export, or `write` for a real test database. Set `MCP_DESIGNER_TEST_EXE`, `SERVER`, `DATABASE`, `USER`, `PASSWORD`, and `ROOT` in the test process environment (the shared prefix applies to each name). The write test adds a uniquely named harmless common module, updates it, reads it back, and verifies mixed-root rejection without forcibly terminating sessions. It retains the test object and artifacts for inspection; never point it at a production database.

## Git synchronization

Synchronization settings belong to a named 1C project. One project can contain a main `configuration` target and any number of named `extension` targets. Each target stores:

- `gitRepositoryPath`: an existing local Git working tree;
- `sourceRelativePath`: the configuration folder inside that repository, containing `Configuration.xml`;
- `kind`: `configuration` or `extension`;
- `extensionName`: the 1C extension name when `kind=extension`;
- optional `scriptsDirectory`, for generated operator entry points.

Use `inspect_git_sync_source` first. It returns the repository root, current branch/HEAD and candidate folders containing `Configuration.xml`. Save the selected folder with `upsert_sync_target`. The one-shot `sync_git_target` never updates Git. Use the explicitly enabled `sync_auto` job for fetching and fast-forwarding the current branch from `origin`.

`sync_git_target` loads the exact committed HEAD represented by the selected folder. The first run performs a full XML import. Later runs calculate committed changes from the last successfully loaded commit and use `ibcmd config import files` with the absolute file list supplied through stdin, then run `config apply` and `config check`. A `Configuration.xml` change, deleted metadata or rewritten Git history switches the plan to a reviewed full load; rewritten history requires `forceFull=true`. A dirty selected source is rejected because it cannot be attributed to a commit hash.

Synchronization returns immediately by default and follows the actual loader PID without a total-duration cutoff. Resource samples are collected every **60 seconds** for the owned process tree. **180 seconds without any increase in CPU time or read/write/other I/O counters** is treated as a suspected hang: only that loader tree is forcibly stopped, the operation becomes `hung`, and automatic sync pauses with an error. Unavailable counters are reported as monitoring errors, not as inactivity. Output silence or zero rounded CPU percentage alone does not trigger the watchdog. This is a heuristic: a client may legitimately wait for a busy remote database, and killing it does not prove server-side work has stopped. Inspect the log/database before resuming.

### Automatic jobs and status

`sync_auto(project, target, allowExecution=true, intervalSeconds=15, credentialId=...)` creates or enables one persistent background job in the database instance. Repeated calls do not create duplicates. The job captures the current branch and origin, fetches that branch, and fast-forwards only a clean checkout. It never resets/stashes local work, switches branches or merges divergent histories. A fixed commit snapshot under `.generated/sync-snapshots` is used for import, so later checkout edits cannot change files in the active load. The snapshot is removed after the cycle.

One automatic cycle per database runs at a time; repositories shared by multiple jobs are serialized. Automatic imports use a separate cycle-owned `ibcmd` lease: an idle retained Designer lease does **not** block them and is never replaced or closed by the cycle. This includes Designer leases opened implicitly by repository commands. A retained `ibcmd` lease (the same loader route), other retained connection types, or an active database operation makes the job wait. Commands arriving during a load still use the shared database queue, so Designer and ibcmd commands do not execute concurrently. A cycle releases only its own connection on success or failure; it does not require closing a caller's idle Designer workflow. A new check is scheduled after the cycle, not while a load is still running. Git/SSH and import errors pause the job until another explicit `sync_auto` call. Jobs resume when the MCP owner restarts, except a restart during an import pauses for inspection. `stop_sync_auto` persists the disabled state and lets an already started load finish; `cancel_operation` is the separate explicit abort command.

Within an automatic cycle, **after import and before apply**, origin is fetched again (`checking_updates` in `sync-info`). If a newer commit exists, its fixed snapshot is imported under the **same operation and database lease**, then Git is checked again. Catch-up compares against the last imported snapshot, including reverted files; deletes/manifest changes require a full import. Apply/check run only once the latest fetch matches the imported commit. Intermediate attempts are `superseded`, never successful. Git/validation failure at this boundary blocks apply and pauses the job. Restart during this boundary also requires inspection. This is polling, not an atomic lock on the remote branch: a push after the final fetch belongs to the next cycle. One-shot `sync_git_target` keeps its no-fetch behaviour.

`sync-info(project, target)` reads the job and current/latest operation **without contacting Git**. It returns enabled state, phase (`fetching`, `importing`, `idle`, `waiting_database`, `waiting_repository`, `stopped`, `paused_error`), branch, loading commit, last successful commit/time, PID, precise process step, resource samples, error, operation ID and log ID/resource URI. `executionConnection` shows the database's current automatic-cycle connection separately from the caller's retained connection returned by `get_project_connection`; it is null between cycles. Use `get_log_file(logId)` to download the actual log. Job settings/status are persisted alongside target history in `auto.json`; they are independent of MCP client sessions.

### SSH preparation and private key storage

1. Call `prepare_sync_git(sshUrl, folder, allowExecution=true)` on the server running this MCP. It creates an Ed25519 deploy identity and returns `credentialId`, the **public** key and its fingerprint. Add that public key to the repository as a **read-only deploy key**. No repository ACL or ownership is silently changed.
2. Obtain and independently verify the Git server's host key. Provide a local verified `known_hosts` file to `connect_sync_git(credentialId, knownHostsPath, allowExecution=true)`. This copies the verified public host keys into the credential store, checks SSH access, and clones into a missing/empty folder. For an existing repository it only verifies the root and access; it does not replace its origin or files.
3. Configure the sync target and pass this `credentialId` to `sync_auto`. Its own fetch uses the prepared SSH URL, without changing the repository's stored origin. SSH uses the specified identity only, batch mode and strict host-key checking. Unknown/changed host keys are errors, not auto-accepted.

Private keys remain under `%ProgramData%/McpControlCenter/Secrets/Git/<credentialId>/id_ed25519`, accessible to the creating service account, SYSTEM and administrators. They are **not** stored in Git, the application folder, `.generated`, logs or the portable JSON. Do not paste private key contents into tools/chat. A new machine/service account requires a newly prepared identity. Keys are currently generated without a passphrase for unattended use; file protection and read-only repository permissions are essential, and revocation is performed at the Git host. Git host registration is an administrator step, not an automatic privilege escalation by this MCP. `ssh.exe` and `ssh-keygen.exe` must be available to the service account.

Without `credentialId`, Git uses the existing credential helper of the **MCP service account**, with interactive prompts disabled. A user's desktop TortoiseGit/Pageant login does not establish service access. A Git/SSH network command has a separate 120-second timeout; it is not the database import deadline.

The successful hash is advanced only after import, apply and check all succeed. `get_sync_status` returns the current and last-successful hashes, pending changes and every failed attempt after the last success, including its attempted hash, error and log path. `get_sync_changes` returns the pending commit list and changed files without changing Git or 1C. The journal is stored under:

```text
mcps\onec-database\.generated\sync\<project>\<target>\history.json
```

If `scriptsDirectory` is configured, every synchronization refreshes UTF-8-with-BOM `sync.ps1` and `status.ps1` entry points there; `install_sync_scripts` can refresh them explicitly without loading 1C. They invoke the same Control Center workflow and contain only the executable/root/project/target references—database and repository passwords are not copied into scripts.

To recursively export one object from the information-base configuration, call `export_infobase_object_recursive` with `project`, `objectName`, `outputPath`, and `allowExecution=true`. For example, `objectName=Обработка.Потребности_ТОИР` produces that root object and all its child metadata under `outputPath`. The operation:

- uses hierarchical XML format;
- first runs `/DumpConfigToFiles -configDumpInfoOnly` in a private `.generated` workspace;
- selects the root and its standalone child files from the direct `ConfigVersions/Metadata` entries and writes a UTF-8 BOM `listFile`; nested `Metadata` entries (attributes, tabular sections and their fields) are included in their parent XML, never requested as separate files;
- runs `/DumpConfigToFiles -listFile` into the requested folder;
- exposes both platform processes as one managed operation with a single operation id, lease, queue position, timeout, and final status;
- defaults to background execution and a one-hour timeout;
- participates in the same persistent Designer lease and per-project queue as repository operations;
- never clears the output folder and does not silently enable incremental `-update` mode.

Use `list_operations` to read its active step, current/last PID, state, elapsed time, and log path. Use `cancel_operation` for a controlled process-tree termination.

Repository commands use the validated IB/Designer binding and the repository address, user, and password. Passwords are never returned by `get_project`; only password-configured indicators are returned.

## Command logs and files

Every tool command is logged by default, including failures before process startup. All executable commands for the information base, database, Designer, repository, RAC and Git synchronization record operation ID, PID, steps, redacted arguments, stdout/stderr, exit status and error details. Designer's native `/Out` log is merged when its process finishes. Background execution continues appending to the original command log after the initial MCP reply.

Use the following tools without any server filesystem access:

- `list_command_logs`: find executions by exact `project`, `command`, or `operationId`; newest first. `limit` is 1–100; pass `nextCursor` as `before` for the next page.
- `search_log`: supply `logId` and literal `text` (case-insensitive). Search covers the complete on-disk log, including old entries omitted from file snapshots. Results contain line numbers; `startLine`/`nextLine` and `maxMatches` support continuation. Long matching lines keep their last 2,000 characters; a page contains at most 60,000 characters of snippets.
- `get_log_file`: supply `logId` to receive the last 60,000 Unicode characters as a UTF-8 `.log` file (embedded MCP resource, base64 bytes). Earlier content is omitted, not the end. No remote drive mapping is required.

For example: run `repository_update_objects`, save its `operationId` and `logId`, check `list_operations`, then call `search_log(logId, text="ERROR")` or `get_log_file(logId)`. If the initial reply was lost, find the execution through `list_command_logs(project, command="repository_update_objects")`. A log existing or reaching `endOfSnapshot` does not mean the operation completed: use its status in `list_operations`.

Each tool reply also provides a `resource_link` and `structuredContent.logId`. Standard MCP clients can use `resources/list` and `resources/read`; Unified routes these requests using its returned resource URI. Both resource reads and `get_log_file` keep the last 60,000 characters, without splitting Unicode surrogate pairs. Resource `_meta.truncatedStart` reports omitted older content; `returnedChars` and `sourceBytes` describe the snapshot. The complete file on disk is never shortened by a read. Command stdout/stderr returned inline also keeps its end rather than its beginning. The client determines how to save/display a file resource. When an operation is running, read again after completion for its final messages.

Files are retained across sessions and restarts under `logs/control-center/onec-database/files/<logId>.log`, next to the Control Center installation, not inside the reinstallable MCP program folder. Command name/project/creation metadata are stored in the same file, so lookup does not depend on a live in-memory operation. Database/repository/cluster passwords and known request secrets are redacted. Logs can still contain business data emitted by 1C; access follows the MCP endpoint's existing access policy. Existing log files from versions before this feature are not retroactively indexed.

## User sessions through RAC

For a client/server infobase, add the following optional project settings with `upsert_project`:

- `racPath`: path to the platform `rac.exe`; when only `rac.exe` is specified, Control Center also searches next to configured `1cv8c.exe` files;
- `rasHost` and `rasPort`: the existing RAS administration endpoint, normally port `1545`;
- `clusterId`, `clusterUser`, and `clusterPassword`: cluster UUID and administrator credentials;
- `infobaseId`: UUID of the single infobase represented by this project.

Use `list_rac_clusters` and `list_rac_infobases` to discover UUIDs, then save them in the project. `list_rac_sessions` returns session records including the `session` UUID, user and application fields emitted by RAC. `terminate_rac_sessions` accepts only an explicit `sessionIds` array, verifies every UUID against the configured infobase immediately before termination, and requires `allowExecution=true`. The optional `errorMessage` is shown to affected users by the 1C platform.

RAC commands run directly without a command shell, use the same persistent per-project `infobase` lease and operation queue, and are recorded in `list_operations`. Cluster passwords are stored with the project but are redacted from MCP responses, operation snapshots, and command arguments. RAS must already be running; this MCP does not install, start, or reconfigure the cluster administration service.
