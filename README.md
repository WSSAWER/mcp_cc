# 1C Database Manager MCP

Standalone C# MCP service for named 1C database/configurator bindings.

Run one persistent Streamable HTTP mediator with `OneCDatabase.exe --onec-db-http --port <port> --onec-db-root <data-root>`. Client MCP sessions may connect and disconnect without owning or terminating its active 1C operation. Control Center is an optional lifecycle/Gate client, not a runtime dependency. The service requires .NET 7 Runtime on Windows.

Runtime project settings are stored outside this source folder:

```text
mcps\onec-database\.generated\projects.json
```

## Validated database structure

### Complete CF/CFE files (Designer)

Two ready MCP commands use the saved, validated Designer connection:

- `infobase_export_configuration(project, filePath, allowExecution=true)` exports
  the **editable** main configuration to `.cf` with `/DumpCfg`.
- `infobase_import_configuration(project, filePath, allowExecution=true)` loads
  a complete `.cf` with `/LoadCfg`. This is **replacement**, not merging or
  a partial object update. Back up the target before replacing it.
- For an extension, pass `extension="ExactExtensionName"` and a `.cfe` path to
  either command. Import can create an absent extension. Export does not create
  one and does not implicitly refresh from a configuration repository.

`filePath` is on the **MCP host**, not the chat computer. Relative paths and
`{APP_DIR}` resolve against `--onec-db-root`; UNC paths require access by the
service account. The input must already be there; these commands do not upload
or download binary payloads over MCP. A file share or another explicit file
transfer mechanism is needed between computers. XML, `.dt`, `.cfu`, empty input,
linked files/directories and wrong CF/CFE target combinations are rejected.
The platform validates the actual binary format and version compatibility.

Both commands return `operationId` and `logId` immediately. Follow
`list_operations` until terminal status, and use `search_log`/`get_log_file`
for the native Designer `/Out` log. `result` reports the host path, byte count,
SHA-256 and `fileExported`, `configurationLoaded`, `databaseUpdated` separately.
The same database queue/lease, PID/resource watchdog and `cancel_operation`
apply. No fixed overall duration limit by default; after 600 seconds continue
checking the operation, not resubmitting it. `timeoutSeconds` is an optional
explicit total deadline.

Options:

| Command | Option | Default | Behaviour |
| --- | --- | --- | --- |
| Export | `overwrite` | `false` | Replace an existing output only explicitly, after a successful nonempty export. Failure preserves the old file. |
| Import | `updateDatabase` | `false` | After successful LoadCfg, apply all pending changes of this configuration/extension with UpdateDBCfg. |
| Import | `terminateSessions` | `false` | Force blocking sessions out only when explicitly requested with `updateDatabase=true`. |
| Import | `warningsAsErrors` | `true` | Treat UpdateDBCfg warnings as errors. |
| Both | `leaseId` | saved/default | Reuse a connection lease; the Designer EXE always comes from the connection's single `configuratorPath`. |

Repository settings alone do not block binary import. Active operations on the
same database serialize execution: the next import waits in `queued` state and
starts after the current operation releases the database gate. No unbind,
capture, unlock or commit is performed implicitly; native 1C restrictions and
explicit connection leases still apply. A failure after
successful LoadCfg **does not roll back** that load: inspect the two result flags
before retrying. Export publishes only a verified nonempty file; private import
copies and incomplete export files are cleaned, while caller files and logs stay.

In the source repository, the executable plan generates
[the CF/CFE workflow](docs/generated/binary-configuration.mmd) and
[the validation rules](docs/generated/binary-configuration-rules.md).
Regression tests invoke public MCP commands and real isolated child processes,
simulating only the 1C platform. They verify arguments, sequencing, partial
failure results, destination preservation, logs, cancellation and serialization;
they do not claim acceptance against a live 1C database.

### Configuration and extension components

Each physical database owns one `configuration` component and its discovered
`extension:<exact name>` components. Call `list_components(project)` first.
The read-only inventory is cached; `refresh=true` reloads it. A reference to an
unknown extension refreshes once and then either resolves it or fails explicitly.
Successful native operations trigger first discovery; extension installation and
ibcmd commands request another discovery. Changed connection settings invalidate
the previous verification. Failed discovery preserves the last inventory and
reports an error; missing extensions keep their settings/history but cannot sync.

`configure_component(project, component, git?, repository?, allowExecution=true)`
stores the selected component's sources. Omitted sections remain unchanged;
`clearGit=true` / `clearRepository=true` explicitly clear a source and stop its
future jobs. Validation uses real Git access or a read-only Designer repository
report. Failed candidates are saved as drafts, without replacing effective
settings. Component repository passwords are protected with Windows DPAPI.

For repository sources, supply `address`, `user`, `password`. Designer always
uses the connection's single `configuratorPath`. Configuration and extensions do not implicitly share
repository credentials. The probe verifies that a nonempty report can be read;
it does **not** prove the existing IB binding or attach the IB to a repository.
That binding must already match. Native integration against a real bound
repository still requires acceptance testing; simulated process tests do not
prove platform behavior for captured objects.

`sync_now(project, component, type="repository", allowExecution=true)` schedules
recursive repository retrieval followed by database update. Retrieval uses
`/ConfigurationRepositoryUpdateCfg -force` **without `-revised`**: it does not
request replacement of captured objects. No capture, commit, unlock or automatic
binding is performed. UpdateDBCfg applies **all pending local changes** of the
selected component. User sessions are not forcibly terminated. If retrieval
fails or is cancelled, database update does not start; no rollback is implied.

The owner loads the configuration into persistent C# database instances. JSON is
storage, not a command template. Each instance owns its bindings, validated adapter
access, connection lease, execution queue, operation/PID registry and sync-state
lock. HTTP sessions and CLI callers use the same instance. Aliases of the same
server/IB share it; conflicting SQL database or cluster/IB identifiers in aliases
block execution instead of silently selecting one. Different bases have independent
queues.

Set normal connection data with `upsert_project`:

- Designer/IB: `configuratorPath`, `infobaseServer`, `infobaseName`,
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
against the configured `--onec-db-root`, not the process's current directory.
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

Start the persistent HTTP owner (directly or through Control Center) before using its CLI entry points. CLI
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

Each saved connection has exactly one `configuratorPath` (a string), used by all Designer, repository and CF/CFE commands, including background synchronization. Configure it with `upsert_project(name, configuratorPath)`; command calls never select an executable. `configuratorIndex` and new `configuratorPaths` inputs are rejected, not ignored. Relative EXE paths resolve against the MCP data root.

On reading an old configuration file, one distinct nonempty path from `ConfiguratorPaths` migrates automatically. Multiple different or malformed paths are preserved in `DesignerPathMigrationIssue`; Designer is blocked for that connection until one `configuratorPath` is explicitly saved. Other connections and independently valid ibcmd settings remain usable. An explicit new path wins over an old list. The active list and old per-component indexes disappear on normal save. See generated `docs/generated/designer-path-rules.md`.

Both commands use the saved, validated database binding and its Designer executable. `files` are paths **on the MCP machine**, not client-machine paths or inline content. Upload/copy files first when using a remote MCP. Both adding and updating accept the usual Designer hierarchy **or a flat staging directory**. The common normalizer reads the owning root's type, name and UUID from XML, even if the XML file has been renamed, and builds the canonical load hierarchy inside the private workspace. This applies to every supported root metadata type, not just roles.

For example, a staging directory with a Role XML named `incoming.xml` and `Rights.xml` becomes `Roles/<XML-name>.xml` and `Roles/<XML-name>/Ext/Rights.xml`. Supply the entire staging directory or explicitly list the root XML and loose properties. Selecting only the root XML also includes its named companion directory and a sibling `Ext` directory; unrelated loose siblings are not silently selected. With several roots, use a separate staging directory per root or companion folders named after each root. Shared loose properties, conflicting destination paths and child metadata XML without an identifiable relative child directory are rejected **before any Designer/repository command**.

Named companion folders and object-relative `Forms`, `Templates`, nested `Subsystems`, `Ext`, etc. retain their hierarchy and ownership. A BSL/child file selects only that file for updates; the owning root XML must be available beside the named object folder (or explicitly included). Adding a root selects its complete supplied bundle. Normalization never renames, moves or deletes caller files; `INPUT FILE` log entries record each original path, normalized path and hash.

```json
{"project":"MyProject","connectionType":"designer","files":["work/CommonModules/MyModule/Ext/Module.bsl"],"allowExecution":true}
```

`infobase_update_files` rejects the entire request if any root is missing in the current configuration or its UUID differs. `infobase_add_objects` accepts only new roots and rejects mixed new/existing roots and UUID collisions. **Pass only the new object files. Do not obtain or edit Configuration yourself:** caller-supplied `Configuration`, `Configuration.xml`, root `Ext` files and whole configuration-dump directories are rejected before any Designer/capture command.

For addition, the MCP exports the current Configuration and metadata index. The root-only `/DumpConfigToFiles -listFile` selection contains `Configuration`, so Designer also supplies its external properties. The MCP preserves the **complete exported bundle** (`Configuration.xml` plus every `Ext` file, recursively, including modules, interfaces, binary files and parent `.cf` files). There is no hardcoded file count or extension filter. It copies that bundle byte-for-byte into a private load package, edits only the manifest entries and adds the new object bundles. New entries go at the **end of their existing type group** in `ChildObjects`; an absent group is inserted in metadata type order. They are not appended indiscriminately at the end of Configuration. A second export rejects concurrent manifest or external-property changes before loading. Caller files are not changed; changed inputs during preparation abort the operation.

Both file commands accept `repositoryMode`:

| Mode | Capture | After successful load, verification and optional DB update |
| --- | --- | --- |
| `manual` | None; required locks must already exist | No repository command |
| `capture` | Existing roots recursively; for additions, only Configuration nonrecursively | Keep locks; do not commit |
| `capture_and_commit` | Same as `capture` | Commit selected roots recursively; for additions include Configuration nonrecursively |

The selected roots are derived from validated input files, not a second user-maintained list. Automatic modes require valid saved repository settings and use saved credentials. No forced capture, retrieval/revision or automatic unlock is performed. Optional `comment` is accepted only with `capture_and_commit`. Without the parameter, previous behaviour is preserved: updates use `manual`; additions use `capture` when repository settings exist, otherwise `manual`. Explicit `manual` disables automatic capture for additions too.

Example: `infobase_update_files(project="MyProject", connectionType="designer", files=["<server staging folder>"], repositoryMode="capture_and_commit", comment="Update selected objects", updateDatabase=false, terminateSessions=false, allowExecution=true)`. Replace the mode with `capture` and omit `comment` to leave changes captured for review. `updateDatabase` is independent from repository commit.

**Recursive commit includes all pending changes within selected roots, not just the supplied files.** Existing-object updates never capture/commit Configuration. New-object addition commits Configuration without its descendants, followed by only the new roots recursively. A capture failure can leave partial locks; a later failure can leave loaded configuration changes. No rollback/unlock is attempted. A failed commit can also have partial native effects: inspect the repository before retrying. Operation `result` exposes `repositoryMode`, `captureStatus`, `commitStatus`, `repositoryCommitted`, `configurationLoaded`, `databaseUpdated`, and cleanup state. Full selections, native errors and steps remain in the downloadable log. The executable workflow diagram and decision table are in `docs/generated/designer-import.mmd` and `docs/generated/designer-repository-rules.md`.

Repository setting state is built with the in-memory database instance at configuration load and refreshed after any saved profile change (including credentials). `get_project_runtime.repository` exposes `not_configured`, `configured`, or `invalid_settings`, the check time and `source=saved_project_settings`. This is not proof of live repository authentication or discovery of an unconfigured IB binding (`liveAccessVerified=false`). Repository operations still fail on actual access errors; availability is not guessed from a nonempty path.

The generated load list includes the full Configuration bundle first and then the new objects; **`-partial` alone is not protection against losing Configuration's omitted external properties**. Existing-object updates do not load Configuration or its external files. The MCP verifies the root set and configuration UUID after loading. By default it then updates the database configuration with dynamic update disabled, warnings treated as errors, and permission to terminate blocking sessions. **Designer database update applies all pending changes of the selected editable configuration, including changes that predate this package.** Optional `terminateSessions=false` prohibits forced termination; `updateDatabase=false` changes only the editable configuration. `extension` selects an extension. Platform support/repository locks remain enforced: the MCP does not bypass support restrictions or acquire someone else's locks.

The response is an operation, not a completion claim. By default there is no total-duration cutoff (`timeoutSeconds=0`); the owned loader PID/resource watchdog and explicit cancellation still apply. Set a positive timeout only when a total deadline is intended. Follow `list_operations`, `list_command_logs`/`search_log`, and `get_log_file`. The exact transmitted file list, byte counts and SHA-256 hashes are recorded as `LOAD FILE` entries in the downloadable command log. After successful load, verification and optional database update, the private `.generated/designer-files/<operationId>` tree is deleted (exports, index, load package and selection lists). Original caller files and permanent logs are never deleted. Failed operations retain their workspace for diagnosis; the log identifies its path. A successful load followed by a failed database update is **not a rollback**; inspect the native log before retrying. The older `infobase_load_objects` uses the same guarded update workflow and cannot add roots.

Verification: run `.test/smoke-all.ps1` with `smoke-onec-database-designer-files.ps1` for isolated process-level tests. The opt-in `smoke-onec-database-designer-live.ps1` uses `MCP_DESIGNER_TEST_MODE=probe` for read-only connection/export, or `write` for a real test database. Set `MCP_DESIGNER_TEST_EXE`, `SERVER`, `DATABASE`, `USER`, `PASSWORD`, and `ROOT` in the test process environment (the shared prefix applies to each name). The write test adds a uniquely named harmless common module, updates it, reads it back, and verifies mixed-root rejection without forcibly terminating sessions. It retains the test object and artifacts for inspection; never point it at a production database.

## Component synchronization

Configure a Git source once with `configure_component`:

- `repositoryUrl`: network endpoint to fetch, without embedded credentials;
- `branch`: explicit branch to follow;
- `checkoutPath`: local checkout on the MCP host;
- `sourceRelativePath`: folder inside the checkout containing `Configuration.xml`;
- `gitSshCredentialId`: optional prepared SSH identity.

The checkout's `origin` may use HTTPS while the selected SSH identity uses
`git@host:owner/repo.git` (or `ssh://git@host/owner/repo.git`). Conventional
hosted URLs with the same host and case-sensitive repository path are equivalent;
optional `.git` and default ports are normalized. Other hosts, paths, SSH users,
nondefault ports and ambiguous/escaped paths are not silently equated. Local
sources still require an exact match. See the [Git URL formats](https://git-scm.com/docs/git-fetch#_git_urls).
Validation never rewrites `origin`. Fetch always uses the configured endpoint,
and a selected SSH key must still match its exact prepared URL and checkout.
The origin identity is checked both before and after fetch.

The service checks the exact branch through noninteractive `git ls-remote`,
clones a missing/empty checkout, and verifies its root, origin, branch, clean
state and metadata. An extension source must name the selected extension.
Existing remotes, ownership, branches and local changes are not rewritten to
make validation pass. Paths escaping the checkout or traversing directory links
are rejected. Source checks have a 120-second network/probe deadline, separate
from database loading. A failed candidate does not enable synchronization.

### Start, observe and stop

Git polling does not reserve the database: fetch and comparison with the last
successfully applied commit happen first. An unchanged commit finishes without
ibcmd, Designer, inventory queries or a connection lease. A changed commit waits
for the database only if it changes loadable configuration files, then
imports/applies through the existing queue. Documentation/index-only commits
are acknowledged in sync history without a database load.
The extension inventory is cached in the database instance. It is read once at
startup/first use, invalidated on a new explicit connection or connection-settings
change, and refreshed after native errors. Reusing a lease and successful ordinary
commands do not invalidate it. Explicit `list_components(refresh=true)` and an
unknown component also request a refresh. No inventory polling occurs per sync tick.
See generated [Git cycle](docs/generated/git-sync.mmd) and
[decision rules](docs/generated/sync-rules.md); build regenerates both.

`sync_now(project, component, type, allowExecution=true)` schedules an immediate cycle.
For an already automatic job it preserves automatic mode and its saved interval;
for a new/stopped job it runs once without enabling polling. It never disables
automatic mode. A running cycle rejects duplicate requests without changing its
schedule. The response reports the effective `automatic` mode and interval.
See generated [scheduling rules](docs/generated/sync-scheduling.md) and
[decision flow](docs/generated/sync-scheduling.mmd).
`sync_auto(project, component, type, intervalSeconds, allowExecution=true)`
creates/enables a persistent job. `type` is `git` or `repository`; component is
explicitly `configuration`, `extension:<name>`, or an ID from `list_components`.
For automatic jobs, an integer interval of 5..86400 seconds is required.
Paths, branch and credentials are not repeated in these calls.

Both return a `jobId`; scheduling is **not** completed loading.
`sync_info(project, jobId?)` reads all jobs or one job without contacting Git or
1C. It reports phase, next check, real native `operationId`/PID when available,
last successful version/time, error, and log URI. `GitFailure` retains the rejected
commit, original error, operation/log IDs and validated source/connection revision;
`LastPollLogId` points to the latest Git-only poll without replacing that error log.
During validation/fetch there
may be no native operation yet. Use `get_log_file` / `search_log` for command
logs. After 600 seconds inspect status; elapsed time alone is not failure.

`stop_sync(project, jobId, allowExecution=true)` disables future cycles and
allows the current load to finish. Explicit `cancel_operation(operationId)` is
a different action. Jobs are independent of chat/MCP sessions. Idle enabled jobs
resume after service restart and revalidate their settings. A failed Git import/apply
with confirmed reset (exit 0, before any successful apply, not cancelled) enters
`waiting_new_commit`: polling continues at the configured interval, but the rejected
hash is never automatically imported again. A different fetched commit is validated
and loaded normally. Polling does not mark the failed commit successful or erase its
log. Transient Git poll errors retain this safe checkpoint and retry at the same interval.
The checkpoint survives service restart; stopped jobs remain stopped. `sync_now` /
`sync_auto` is an explicit resumption that permits retrying even the same hash.
Interrupted loading, failed/missing reset, post-apply check failures, changed recovery
scope, and legacy errors without rollback evidence remain `paused_error` until review
and explicit resumption. A reset of the editable configuration is not proof of rolling
back changes already applied to the database. One-shot jobs never start periodic polling.

One physical DB has one write queue, even if multiple project names refer to it.
Its components have independent schedules, but their native writes are
serialized. Different DBs can load concurrently. Two enabled source types for
one component are rejected with `source_conflict`. An idle retained Designer
lease does not block a Git/ibcmd cycle; the cycle does not close that lease.
The same retained route or actual running operations cause waiting.
The next check is scheduled after cycle completion, not during a running load.

### Git load sequence and history

Every Git cycle, including `sync_now`, fetches the configured branch and
fast-forwards a clean checkout. It never resets/stashes work or merges divergent
history. A matching last-successful hash causes no database import. A new hash
is imported from an immutable snapshot under
`.generated/sync-work/<project scope>/<direction scope>/sync-snapshots/<invocation>`.
Git loaders, repository synchronization and component probes also receive separate
working directories under their project/direction's `commands/<invocation>`.
Names include a readable prefix and stable hash to prevent sanitized-name collisions.
Only the current invocation's temporary files are removed after completion/failure;
other projects, configured Git checkouts, base data directories and persistent
sync journals are not moved or deleted. `SYNC WORKSPACE` in the command log records
the actual working directory.

Each named `run_ibcmd` operation, extension flag update, Git synchronization
(import/apply/check/reset) and component discovery receives a fresh server-data directory:
`<configured dataPath>/runs/run-<short project name>-<GUID>`.
Its validated C# connection uses this directory as `--data`, without copying old
`session-data`. SQL server/database/authentication and the explicit absolute
`--database-path` of a file IB stay unchanged; `ibcmd.exe` is not copied.
`IBCMD DATA` records the exact path in the command log. After native process exit,
only this invocation's data is removed. Cleanup first tries normal deletion, then
clears ReadOnly attributes in the owned subtree and retries, without following links.
A cleanup failure is logged with the retained path; a later launch never reuses it.
Background requests return an operation ID without deleting the running task's
workspace. Cleanup checks the original process handle; if it is alive or its exit
cannot be confirmed, files are retained with a warning. Only owned directories
are eligible for cleanup. Advanced `operation=custom` remains an explicit raw
diagnostic command: its arguments and user-specified data are not rewritten/deleted.
The generated cleanup decisions are in [ibcmd workspace rules](docs/generated/ibcmd-workspaces.md).
This prevents reuse of stale session-data, but cannot guarantee that every native
filesystem error is eliminated. Existing paused jobs are not resumed automatically.
The first import is full; later imports use changed files where safe.
Manifest/deletion changes require a full import; divergent history is rejected
for review rather than silently overwriting the database.

After import, Git is checked again **before apply/check**. New commits are
imported under the same operation and lease until the latest fetch matches the
imported snapshot; only then are apply/check run. Intermediate hashes are
`superseded`, never successful. This is polling: a push after the final fetch
belongs to the next cycle. The successful hash advances only after all native
steps succeed. Operation-owned snapshots are cleaned up after the cycle.

`get_component_sync_history(project, component)` returns the successful hash,
failures/logs since that success, and pending local checkout commits/files.
It does not fetch. Aliases share the original history location. Settings/jobs
are stored atomically under `.generated/components/<database identity hash>`;
journals remain under `.generated/sync/<original project>/<history target>`.
Legacy target/auto settings are migrated without deleting their JSON/history.
Conflicting sources are reported and paused; disabled jobs stay disabled.
Old target-based MCP tools and `repository_sync_auto` are replaced, not run
beside the new scheduler.

The CLI uses the same owner process and contract:

```text
OneCDatabase.exe --onec-db-sync --root "<installation>" --project "<project>" --component "extension:<name>" --type git --allow-execution
OneCDatabase.exe --onec-db-sync-status --root "<installation>" --project "<project>" --job-id "<returned ID>"
```

The first call schedules, it does not block until import completion.
Old scripts passing `--target` must be regenerated or adjusted to this contract.

### Git SSH identity

1. `prepare_git_ssh(sshUrl, folder, allowExecution=true)` creates an Ed25519
   identity and returns `gitSshCredentialId`, its **public** key and fingerprint.
   Add the public key as a read-only deploy key at the Git host.
2. Independently verify the Git host key, then pass a verified local
   `knownHostsPath` to `connect_git_ssh(gitSshCredentialId, knownHostsPath,
   allowExecution=true)`. This verifies access and prepares a missing checkout.
3. Save this ID in the component's Git source. A selected identity is exclusive:
   strict host-key checking, batch mode and no fallback to another identity.
   Unknown/changed host keys fail.

Private keys remain in
`%ProgramData%/McpControlCenter/Secrets/Git/<gitSshCredentialId>/id_ed25519`,
accessible to the creating service account, SYSTEM and administrators.
They are not included in portable JSON, application files, replies or logs.
Keys are unencrypted files for unattended SSH; ACLs and read-only repository
permissions are essential. A new host/account needs its own identity.
Without an ID, noninteractive Git uses the MCP service account's configured
agent/helper. A desktop user's Pageant session does not establish service access.

### Process supervision

The loader has no default total-duration deadline. Every 60 seconds the owned
process tree is sampled; 180 seconds without CPU or read/write/other I/O progress
causes a suspected-hang stop and a logged `hung` result. Any increasing counter
resets the whole idle period; unavailable counters are not treated as zero.
This remains a heuristic: a client can wait for remote DB work. Inspect the
database/log before resuming after an interruption.

To recursively export one object from the information-base configuration, call `export_infobase_object_recursive` with `project`, `objectName`, `outputPath`, and `allowExecution=true`. For example, `objectName=Обработка.Потребности_ТОИР` produces that root object and all its child metadata under `outputPath`. The operation:

- uses hierarchical XML format;
- first runs `/DumpConfigToFiles -configDumpInfoOnly` in a private `.generated` workspace;
- selects the root and its standalone child files from the direct `ConfigVersions/Metadata` entries and writes a UTF-8 BOM `listFile`; nested `Metadata` entries (attributes, tabular sections and their fields) are included in their parent XML, never requested as separate files;
- runs `/DumpConfigToFiles -listFile` into the requested folder;
- exposes both platform processes as one managed operation with a single operation id, lease, queue position, timeout, and final status;
- defaults to background execution without an execution deadline (`timeoutSeconds=0`);
- participates in the same persistent Designer lease and per-project queue as repository operations;
- never clears the output folder and does not silently enable incremental `-update` mode.

Use `list_operations` to read its active step, current/last PID, state, elapsed time, and log path. Use `cancel_operation` for a controlled process-tree termination.

Repository commands use the validated IB/Designer binding and the repository address, user, and password. Passwords are never returned by `get_project`; only password-configured indicators are returned.

## Long-running operations

Execution and observation are separate. All managed 1C command timeouts default to `0` (no total-duration cutoff), including repository commands, recursive exports, ibcmd and extension installation. Two hours or more can be normal for a large configuration. A positive `timeoutSeconds` is an explicit **hard deadline that kills the owned process tree**, not a client wait interval. Do not pass `600` or `1200` just to wait.

Use background execution for long commands and retain the returned `operationId` and `logId`. Ready repository/file/sync commands already return an operation; `run_extension_install` also defaults to `background=true`. Advanced commands such as `run_ibcmd`/`run_repository_command` should be called with `background=true` to keep the MCP connection available for status requests. CLI extension installation retains synchronous execution, without an implicit deadline.

After **600 seconds**, request `list_operations` and inspect the matching operation's `status`, `currentStep`, PID, activity and `lastError`; use `search_log` or `get_log_file` for details. Continue checking while the state is `starting`, `queued` or `running`. For automatic synchronization, `sync_info` exposes the current operation as well. Snapshots include `statusCheckAfterSeconds=600`, `statusCheckDue` and `nextAction`. This threshold does not cancel the operation, change its status or imply a hang. Terminal elapsed time stays fixed.

A client/transport timeout or an incomplete log is not a result: locate the original operation before retrying. Do not cancel or launch a duplicate based solely on elapsed time. Explicit cancellation and actual process failures remain distinct from total duration.

Automatic inactivity interruption applies to **all managed command processes**, including Designer/repository/export, ibcmd and RAC, not only imports. Every 60 seconds it samples the entire owned process tree. After 180 seconds without CPU or read/write/other I/O progress it stops that tree automatically and reports `hung` with an error in the command log. **Any one increasing counter resets the entire idle period** (CPU, read, write or other I/O; operations or bytes). These Windows I/O counters include file, network and device activity; they are not separate disk/network byte meters. Unknown/unreadable counters and counter resets never count as zero activity. Each check writes exactly one `ACTIVITY` log line with all counters, `observation`, `activeCounters`, `resetIdleTimer` and `idleSeconds`; interruption adds a separate `HUNG` error. The operation snapshot identifies the I/O scope and watchdog thresholds. This remains an inactivity heuristic, not proof that work on a remote server is stuck; inspect the database after interruption.

## Command logs and files

Every tool command is logged by default, including failures before process startup. All executable commands for the information base, database, Designer, repository, RAC and Git synchronization record operation ID, PID, steps, redacted arguments, stdout/stderr, exit status and error details. Designer's native `/Out` log is merged when its process finishes. Background execution continues appending to the original command log after the initial MCP reply.

Use the following tools without any server filesystem access:

- `list_command_logs`: find executions by exact `project`, `command`, or `operationId`; newest first. `limit` is 1–100; pass `nextCursor` as `before` for the next page.
- `search_log`: supply `logId` and literal `text` (case-insensitive). Search covers the complete on-disk log, including old entries omitted from file snapshots. Results contain line numbers; `startLine`/`nextLine` and `maxMatches` support continuation. Long matching lines keep their last 2,000 characters; a page contains at most 60,000 characters of snippets.
- `get_log_file`: supply `logId` to receive the last 60,000 Unicode characters as a UTF-8 `.log` file (embedded MCP resource, base64 bytes). Earlier content is omitted, not the end. No remote drive mapping is required.

For example: run `repository_update_objects`, save its `operationId` and `logId`, check `list_operations`, then call `search_log(logId, text="ERROR")` or `get_log_file(logId)`. If the initial reply was lost, find the execution through `list_command_logs(project, command="repository_update_objects")`. A log existing or reaching `endOfSnapshot` does not mean the operation completed: use its status in `list_operations`.

Each tool reply also provides a `resource_link` and `structuredContent.logId`. Standard MCP clients can use `resources/list` and `resources/read`; Unified routes these requests using its returned resource URI. Both resource reads and `get_log_file` keep the last 60,000 characters, without splitting Unicode surrogate pairs. Resource `_meta.truncatedStart` reports omitted older content; `returnedChars` and `sourceBytes` describe the snapshot. The complete file on disk is never shortened by a read. Command stdout/stderr returned inline also keeps its end rather than its beginning. The client determines how to save/display a file resource. When an operation is running, read again after completion for its final messages.

Files are retained across sessions and restarts under `logs/control-center/onec-database/files/<logId>.log` within `--onec-db-root`, not inside the reinstallable MCP program folder. Command name/project/creation metadata are stored in the same file, so lookup does not depend on a live in-memory operation. Database/repository/cluster passwords and known request secrets are redacted. Logs can still contain business data emitted by 1C; access follows the MCP endpoint's existing access policy. Existing log files from versions before this feature are not retroactively indexed.

## User sessions through RAC

For a client/server infobase, add the following optional project settings with `upsert_project`:

- `racPath`: path to the platform `rac.exe`; when only `rac.exe` is specified, the service also searches next to configured `1cv8c.exe` files;
- `rasHost` and `rasPort`: the existing RAS administration endpoint, normally port `1545`;
- `clusterId`, `clusterUser`, and `clusterPassword`: cluster UUID and administrator credentials;
- `infobaseId`: UUID of the single infobase represented by this project.

Use `list_rac_clusters` and `list_rac_infobases` to discover UUIDs, then save them in the project. `list_rac_sessions` returns session records including the `session` UUID, user and application fields emitted by RAC. `terminate_rac_sessions` accepts only an explicit `sessionIds` array, verifies every UUID against the configured infobase immediately before termination, and requires `allowExecution=true`. The optional `errorMessage` is shown to affected users by the 1C platform.

RAC commands run directly without a command shell, use the same persistent per-project `infobase` lease and operation queue, and are recorded in `list_operations`. Cluster passwords are stored with the project but are redacted from MCP responses, operation snapshots, and command arguments. RAS must already be running; this MCP does not install, start, or reconfigure the cluster administration service.

## Standalone build and upgrade

Run `.agents/build.ps1 -Configuration Debug`, then `.test/smoke-all.ps1`.
After verification build Release. Portable output is in
`src/bin/Release/net7.0-windows/win-x64/portable` and needs no build tools at runtime.
The installable Git branch is `release`; it contains the binary and product files.
Repository: `https://github.com/WSSAWER/onec_database.git`. Source is maintained
on `main`; `.agents/publish-release.ps1` verifies the Release runtime and publishes
only the portable payload to `release`. Run the complete Debug smoke suite first.

Keep the same `--onec-db-root` when replacing the old builtin host. This preserves
projects, operation logs, synchronization state and the single-owner lock. The
legacy lock/pipe names deliberately remain compatible: a second old/new owner
must fail instead of running loaders concurrently. Keep the same Windows account
for DPAPI secrets. Shutdown/update of a running service requires explicit action.

`--export-workflows --output <folder> [--check]` exports deterministic Mermaid
from the actual operation lifecycle. CC may include these generated snapshots in
its viewer, but does not reference the database runtime assembly.
