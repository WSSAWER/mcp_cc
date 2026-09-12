# CF/CFE decisions (generated from executable rules)

Every required condition must hold. Otherwise the operation is blocked, not silently adjusted. Repository settings alone never deny an import. Active work on the same database is serialized by DatabaseInstance.Acquire: the next operation waits in queued state until the current owner releases the gate. Native 1C restrictions and explicit connection leases still apply.

| Rule ID | Required condition | If false | Message |
| --- | --- | --- | --- |
| binary.ExecutionConsent | allowExecution=true | Block | Binary configuration operations require allowExecution=true. Import replaces the complete editable configuration or extension, not selected objects. |
| binary.FileType | No extension: .cf; explicit extension: .cfe | Block | Main configuration requires .cf; specify extension explicitly for a .cfe file. |
| binary.NonemptyInput | Import input exists on the MCP host and is nonempty | Block | Configuration input was not found or is empty on the MCP host. |
| binary.OverwriteConsent | Export destination is absent, or overwrite=true | Block | Destination already exists; use overwrite=true explicitly. |
| binary.ApplyBeforeTerminate | terminateSessions=false, or updateDatabase=true | Block | terminateSessions requires updateDatabase=true. |
