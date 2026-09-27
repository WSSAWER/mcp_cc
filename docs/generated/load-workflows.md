# Loading workflows (generated from executable models)

Generation: `.agents/build.ps1 -Configuration Debug|Release`; freshness: `.agents/export-workflows.ps1 -Check`. No machine paths/credentials are included. Native Designer compatibility is not proven by simulated tests.

- [binary-configuration-rules.md](binary-configuration-rules.md)
- [database-topology.md](database-topology.md)
- [designer-export.md](designer-export.md)
- [designer-import.md](designer-import.md)
- [designer-path-rules.md](designer-path-rules.md)
- [designer-readiness.md](designer-readiness.md)
- [designer-repository-rules.md](designer-repository-rules.md)
- [features.md](features.md)
- [git-export-rules.md](git-export-rules.md)
- [ibcmd-workspaces.md](ibcmd-workspaces.md)
- [log-retention.md](log-retention.md)
- [sync-rules.md](sync-rules.md)
- [sync-scheduling.md](sync-scheduling.md)
- [temporary-files.md](temporary-files.md)
- [tool-catalog.md](tool-catalog.md)

## designer-import.mmd

```mermaid
stateDiagram-v2
	direction LR
	Preparing --> Acquired : Acquire
	Preparing --> Failed : Fail
	Acquired --> RootCaptured : CaptureRoot
	Acquired --> IndexRead : ReadIndex
	Acquired --> Failed : Fail
	RootCaptured --> IndexRead : ReadIndex
	RootCaptured --> Failed : Fail
	IndexRead --> ManifestRead : ReadManifest
	IndexRead --> Validated : Validate
	IndexRead --> Failed : Fail
	ManifestRead --> Validated : Validate
	ManifestRead --> Failed : Fail
	Validated --> ObjectsCaptured : CaptureObjects
	Validated --> PackagePrepared : PreparePackage
	Validated --> Failed : Fail
	ObjectsCaptured --> PackagePrepared : PreparePackage
	ObjectsCaptured --> Failed : Fail
	PackagePrepared --> Rechecked : Recheck
	PackagePrepared --> Failed : Fail
	Rechecked --> Loaded : Load
	Rechecked --> Failed : Fail
	Loaded --> Verified : Verify
	Loaded --> Failed : Fail
	Verified --> Applied : Apply
	Verified --> Committed : Commit
	Verified --> Completed : Complete
	Verified --> Failed : Fail
	Applied --> Committed : Commit
	Applied --> Completed : Complete
	Applied --> Failed : Fail
	Committed --> Completed : Complete
	Committed --> Failed : Fail
[*] --> Preparing
state "Preparing: Validate saved binding and input identities; reject caller Configuration and ambiguous paths" as Preparing
state "Acquired: Wait for and acquire the database operation lease; own private workspace" as Acquired
state "RootCaptured: Add + automatic repository: capture Configuration NONRECURSIVELY; no forced capture" as RootCaptured
state "IndexRead: Dump ConfigDumpInfo only; read format, root names, UUIDs and versions" as IndexRead
state "ManifestRead: Add only: root-version delta export; require only Configuration.xml and optional index; no full fallback" as ManifestRead
state "Validated: Update: existing names and UUIDs; add: new names and unique UUIDs; validate XML version" as Validated
state "ObjectsCaptured: Update + automatic repository: capture selected existing roots recursively" as ObjectsCaptured
state "PackagePrepared: Verify source hashes; copy only input files; add only: append names to matching groups in Configuration.xml" as PackagePrepared
state "Rechecked: Fresh index: refuse changed identity, root set or selected metadata; no Configuration/CF export" as Rechecked
state "Loaded: Log exact list, sizes and SHA-256; LoadConfigFromFiles listFile + partial; later error is NOT rollback" as Loaded
state "Verified: Fresh post-load index: verify expected root set, UUIDs and unchanged Configuration external properties before apply/commit" as Verified
state "Applied: Optional UpdateDBCfg: all pending changes; Dynamic-; session policy and warningsAsErrors" as Applied
state "Committed: capture_and_commit only: selected roots recursively; add includes Configuration nonrecursively" as Committed
state "Completed: Confirm child exit; clean owned files; keep downloadable logs for 30 days; preserve caller files" as Completed
state "Failed: Record exact error/partial outcome; stop owned process if needed; no rollback/unlock; retain failed files 7 days" as Failed
```

## binary-configuration.mmd

```mermaid
flowchart LR
  ExportValidate["Export: validate request / connection"]
  ExportValidate --> ExportQueue["queued: wait while same database is busy"]
  ExportQueue -->|success| ExportDumpCfg["/DumpCfg"]
  ExportDumpCfg -->|failure| ExportFailed["failed + log; no later steps"]
  ExportDumpCfg -->|success| ExportDone["verify file/result; clean private staging"]
  ImportValidate["Import: validate request / connection"]
  ImportValidate --> ImportQueue["queued: wait while same database is busy"]
  ImportQueue -->|success| ImportLoadCfg["/LoadCfg"]
  ImportLoadCfg -->|failure| ImportFailed["failed + log; no later steps"]
  ImportLoadCfg -->|success| ImportUpdateDBCfg["/UpdateDBCfg (opt-in)"]
  ImportUpdateDBCfg -->|failure| ImportFailed["failed + log; no later steps"]
  ImportUpdateDBCfg -->|success| ImportDone["verify file/result; clean private staging"]
  ImportLoadCfg -->|updateDatabase=false| ImportDone
```

## designer-readiness.mmd

```mermaid
flowchart TD
  probe[Read fresh ConfigDumpInfo after loader exit]
  probe -->|Probe reports an infobase structure error| Block["Infobase structure error during readiness check; no automatic retry or repair."]
  probe -->|Native output reports unfinished configuration saving| Wait["Configuration saving has not finished; UpdateDBCfg is not started."]
  probe -->|Probe exit 0 and fresh valid ConfigDumpInfo; no saving warning| Ready["Editable configuration is readable; UpdateDBCfg may start."]
  probe -->|Any other probe failure or missing/invalid index| Block["Configuration readiness could not be confirmed; UpdateDBCfg is blocked."]
  Wait -->|30 seconds; cancellation and deadline checked| probe
  Ready --> apply[UpdateDBCfg once]
  Block --> failure[Preserve error and logs; no apply or repair]
```

## git-sync.mmd

```mermaid
stateDiagram-v2
	direction LR
	fetching --> no_changes : Unchanged
	fetching --> waiting_git_retry : AccessUnavailable
	fetching --> waiting_new_commit : RejectedCommit
	fetching --> reserving : UpdateRequired
	fetching --> failed : Fail
	reserving --> importing : Reserved
	reserving --> waiting_database : Busy
	reserving --> failed : Fail
	importing --> checking_updates : Imported
	importing --> failed : Fail
	checking_updates --> importing : NewerCommit
	checking_updates --> applying : CaughtUp
	checking_updates --> failed : Fail
	applying --> completed : Applied
	applying --> failed : Fail
[*] --> fetching
state "fetching: Lock checkout; fetch saved remote/branch with saved SSH; compare last success; validate clean linear source; NO DB lease" as fetching
state "no_changes: No new loadable files: acknowledge version without inventory, Designer or ibcmd" as no_changes
state "waiting_new_commit: Rejected hash unchanged: retain original failure; wait for a newer commit without DB calls" as waiting_new_commit
state "waiting_git_retry: Git/SSH unavailable: retain error; automatic retry after one hour; no DB lease" as waiting_git_retry
state "reserving: Changed files only: resolve component inventory; reserve saved adapter and database queue" as reserving
state "importing: Immutable commit snapshot; validate files; ibcmd config import full source or changed files with partial; tracked PID and private working data" as importing
state "checking_updates: Fetch again AFTER import and BEFORE apply; a newer commit is imported under the same lease" as checking_updates
state "applying: No newer commit: ibcmd config apply force, then config check; preserve last success until BOTH finish" as applying
state "completed: Persist successful commit/history; release lease/checkout; clean owned snapshots after process exit" as completed
state "waiting_database: Database busy: defer to next interval; do not interrupt its owner" as waiting_database
state "failed: Stop later steps; if import began and not cancelled, attempt editable config reset; recovery rules decide retry; failed files 7 days, logs 30 days" as failed
```

## git-export.mmd

```mermaid
stateDiagram-v2
	direction LR
	Preparing --> ReadingIndex : Read
	Preparing --> Failed : Fail
	ReadingIndex --> Exporting : Export
	ReadingIndex --> SavingCheckpoint : Save
	ReadingIndex --> Failed : Fail
	Exporting --> Publishing : Publish
	Exporting --> Failed : Fail
	Publishing --> Committing : Commit
	Publishing --> Failed : Fail
	Committing --> SavingCheckpoint : Save
	Committing --> Failed : Fail
	SavingCheckpoint --> Completed : Complete
	SavingCheckpoint --> Failed : Fail
[*] --> Preparing
```

## repository-sync.mmd

```mermaid
flowchart TD
  validate[Validate saved repository and Designer binding] --> queue[Acquire one database queue and Designer lease]
  queue -->|success| retrieving_repository["UpdateCfg recursively; no Objects/no revised; force confirms incoming changes; captured objects are not replaced"]
  retrieving_repository -->|failure or cancellation| error[Stop later steps; log partial outcome; no rollback]
  retrieving_repository -->|success| updating_database["UpdateDBCfg applies ALL pending changes; Dynamic-; SessionTerminate disable; no commit/unlock"]
  updating_database -->|failure or cancellation| error[Stop later steps; log partial outcome; no rollback]
  updating_database -->|success| done[Record success; release lease; clean owned files after process exit]
  error --> retention[Failed files: 7 days; explicit cancel: clean after exit; logs: 30 days]
```

## sync-scheduling.mmd

```mermaid
flowchart TD
  request[Scheduling request] --> r0
  r0{"sync.schedule.running_same: Cycle running; sync_auto_enable repeats the enabled interval"} -->|yes| o0["Keep the existing cycle and schedule unchanged."]
  r0 -->|no| r1
  r1{"sync.schedule.running: Cycle running; any other scheduling request"} -->|yes| o1["sync_running: inspect sync_job_status_get; do not start a duplicate."]
  r1 -->|no| r2
  r2{"sync.schedule.auto: No running cycle; sync_auto_enable"} -->|yes| o2["Enable automatic polling with the requested interval; schedule now."]
  r2 -->|no| r3
  r3{"sync.schedule.now_auto: No running cycle; sync_once_start; automatic mode already enabled"} -->|yes| o3["Schedule now; preserve automatic mode and its saved interval."]
  r3 -->|no| r4
  r4{"sync.schedule.now_once: No running cycle; sync_once_start; new or disabled job"} -->|yes| o4["Schedule one cycle without enabling automatic mode; preserve the saved interval."]
```

## database-operation-overview.mmd

```mermaid
flowchart LR
  subgraph detail_active["Внутри active"]
    starting["starting"]
    queued["queued"]
    running["running"]
  end
  active["active: starting / queued / running"]
  completed["completed"]
  failed["failed"]
  timed_out["timed_out"]
  cancelled["cancelled"]
  hung["hung"]
  active -->|"Succeed"| completed
  active -->|"Fail"| failed
  active -->|"DeadlineExpired"| timed_out
  active -->|"Cancel"| cancelled
  active -->|"Inactivity"| hung
  starting -->|"Queue"| queued
  starting -->|"ProcessStarted"| running
  queued -->|"Queue"| queued
  queued -->|"ProcessStarted"| running
  running -->|"Queue"| queued
  running -->|"ProcessStarted"| running
  initial(("начало")) --> starting
```

## temporary-files.mmd

```mermaid
flowchart TD
  r0{"temp.owner_active: Scope is open and owner is alive or unknown"} -->|yes| retain[Retain and retry next minute]
  r0 -->|no| r1
  r1{"temp.child_unconfirmed: Child launch/exit is not confirmed"} -->|yes| retain[Retain and retry next minute]
  r1 -->|no| r2
  r2{"temp.failure_week: Failed operation's file/directory was created less than 7 days ago"} -->|yes| retain[Retain and retry next minute]
  r2 -->|no| r3
  r3{"temp.finished: Scope closed or owner exited, all children exited"} -->|yes| remove[Remove owned paths; retry on file lock]
```
