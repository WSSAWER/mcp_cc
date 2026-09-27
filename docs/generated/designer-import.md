# Designer XML import (generated)

Main configuration and a named extension use the same pipeline; every native command receives the same extension scope and saved connection. Optional branches are selected by action, repositoryMode and updateDatabase, not raw caller arguments.

| State | Executed meaning |
| --- | --- |
| Preparing | Validate saved binding and input identities; reject caller Configuration and ambiguous paths |
| Acquired | Wait for and acquire the database operation lease; own private workspace |
| RootCaptured | Add + automatic repository: capture Configuration NONRECURSIVELY; no forced capture |
| IndexRead | Dump ConfigDumpInfo only; read format, root names, UUIDs and versions |
| ManifestRead | Add only: root-version delta export; require only Configuration.xml and optional index; no full fallback |
| Validated | Update: existing names and UUIDs; add: new names and unique UUIDs; validate XML version |
| ObjectsCaptured | Update + automatic repository: capture selected existing roots recursively |
| PackagePrepared | Verify source hashes; copy only input files; add only: append names to matching groups in Configuration.xml |
| Rechecked | Fresh index: refuse changed identity, root set or selected metadata; no Configuration/CF export |
| Loaded | Log exact list, sizes and SHA-256; LoadConfigFromFiles listFile + partial; later error is NOT rollback |
| Verified | Fresh post-load index: verify expected root set, UUIDs and unchanged Configuration external properties before apply/commit |
| Applied | Optional UpdateDBCfg: all pending changes; Dynamic-; session policy and warningsAsErrors |
| Committed | capture_and_commit only: selected roots recursively; add includes Configuration nonrecursively |
| Completed | Confirm child exit; clean owned files; keep downloadable logs for 30 days; preserve caller files |
| Failed | Record exact error/partial outcome; stop owned process if needed; no rollback/unlock; retain failed files 7 days |
