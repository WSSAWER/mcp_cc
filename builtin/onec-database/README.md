# 1C Database Manager MCP

Built-in MCP Control Center profile for named 1C database/configurator bindings.

The executable MCP server is `McpControlCenter.exe --onec-db-mcp`; this folder is a local install source marker for the Control Center profile.

Runtime project settings are stored outside this source folder:

```text
mcps\onec-database\.generated\projects.json
```

The MCP supports:

- `ibcmd infobase config` operations: `generation_id`, `reset`, `apply`, `check`, `import`, `export`, `custom`;
- extension installation/update through `1cv8 DESIGNER`: `/LoadCfg`, `/UpdateDBCfg`, `-Extension`;
- extension flag updates through `ibcmd extension update`: `active`, `safe-mode`, `unsafe-action-protection`;
- 1C Designer configuration repository operations: `create`, `add_user`, `unbind_cfg`, `update_cfg`, `dump_cfg`, `report`, `lock`, `unlock`, `commit`, `set_label`, `custom`;
- recursive export from the information-base configuration: `export_infobase_object_recursive` discovers the metadata tree and exports the root plus its child forms, layouts, requisites, commands, and other child metadata as one managed operation;
- RAS/RAC administration through the platform `rac.exe`: discovery of clusters and infobases, listing sessions for one configured infobase, and explicit termination of selected session UUIDs;
- long-running command execution with `timeoutSeconds`;
- managed process operation tracking: `list_operations` shows operation id, PID, status, elapsed time and log path; `cancel_operation` terminates a running operation and its process tree;
- background execution for long operations: pass `background=true` to `run_ibcmd`, `run_repository_command`, or `run_extension_update_flags` to return immediately with `operationId`;
- safe argument-array command construction without shell.

For partial repository operations, prefer `objects`, an array of full root metadata names. Control Center creates the required 1C XML selection and marks every root with `includeChildObjects=true`. `objectsPath` is an advanced alternative and must point to an existing XML selection in the 1C `Objects` format. Examples of root names:

```text
Обработка.Потребности_ТОИР
Справочник.Номенклатура
Документ.ЗаказПокупателя
```

`dump_cfg` exports a complete repository configuration and does not accept an object selection.

To recursively export one object from the information-base configuration, call `export_infobase_object_recursive` with `project`, `objectName`, `outputPath`, and `allowExecution=true`. For example, `objectName=Обработка.Потребности_ТОИР` produces that root object and all its child metadata under `outputPath`. The operation:

- uses hierarchical XML format;
- first runs `/DumpConfigToFiles -configDumpInfoOnly` in a private `.generated` workspace;
- selects the root and every child entry from `ConfigDumpInfo.xml` and writes a UTF-8 BOM `listFile`;
- runs `/DumpConfigToFiles -listFile` into the requested folder;
- exposes both platform processes as one managed operation with a single operation id, lease, queue position, timeout, and final status;
- defaults to background execution and a one-hour timeout;
- participates in the same persistent Designer lease and per-project queue as repository operations;
- never clears the output folder and does not silently enable incremental `-update` mode.

Use `list_operations` to read its active step, current/last PID, state, elapsed time, and log path. Use `cancel_operation` for a controlled process-tree termination.

Repository project settings include designer connection arguments, repository address, repository user, and repository password. Passwords are never returned by `get_project`; only `repositoryPasswordConfigured` is returned.

## User sessions through RAC

For a client/server infobase, add the following optional project settings with `upsert_project`:

- `racPath`: path to the platform `rac.exe`; when only `rac.exe` is specified, Control Center also searches next to configured `1cv8c.exe` files;
- `rasHost` and `rasPort`: the existing RAS administration endpoint, normally port `1545`;
- `clusterId`, `clusterUser`, and `clusterPassword`: cluster UUID and administrator credentials;
- `infobaseId`: UUID of the single infobase represented by this project.

Use `list_rac_clusters` and `list_rac_infobases` to discover UUIDs, then save them in the project. `list_rac_sessions` returns session records including the `session` UUID, user and application fields emitted by RAC. `terminate_rac_sessions` accepts only an explicit `sessionIds` array, verifies every UUID against the configured infobase immediately before termination, and requires `allowExecution=true`. The optional `errorMessage` is shown to affected users by the 1C platform.

RAC commands run directly without a command shell, use the same persistent per-project `infobase` lease and operation queue, and are recorded in `list_operations`. Cluster passwords are stored with the project but are redacted from MCP responses, operation snapshots, and command arguments. RAS must already be running; this MCP does not install, start, or reconfigure the cluster administration service.
