# 1C Database Manager MCP

Built-in MCP Control Center profile for named 1C database/configurator bindings.

The executable MCP server is `McpControlCenter.exe --onec-db-mcp`; this folder is a local install source marker for the Control Center profile.

Runtime project settings are stored outside this source folder:

```text
mcps\onec-database\.generated\projects.json
```

The MCP supports:

- a project-first workflow: call `list_projects`, select a binding, then call `get_project_actions` to see which Designer, repository, ibcmd, and RAC actions are actually configured;
- ready repository actions: `repository_get_objects`, `repository_update_objects`, `repository_lock_objects`, `repository_unlock_objects`, and `repository_commit_objects`; these accept a project name and action data while executable, infobase, repository address, and credentials come from the saved project;
- `ibcmd infobase config` operations: `generation_id`, `reset`, `apply`, `check`, `import`, `export`, `custom`;
- extension installation/update through `1cv8 DESIGNER`: `/LoadCfg`, `/UpdateDBCfg`, `-Extension`;
- extension flag updates through `ibcmd extension update`: `active`, `safe-mode`, `unsafe-action-protection`;
- 1C Designer configuration repository operations: `create`, `add_user`, `unbind_cfg`, `update_cfg`, `dump_cfg`, `report`, `lock`, `unlock`, `commit`, `set_label`, `custom`;
- recursive export from the information-base configuration: `export_infobase_object_recursive` discovers the metadata tree and exports the root plus its child forms, layouts, requisites, commands, and other child metadata as one managed operation;
- loading selected root objects into the information-base configuration: `infobase_load_objects` discovers the root XML and all child XML/BSL files in a hierarchical dump, creates the `listFile`, runs `/LoadConfigFromFiles`, and optionally `/UpdateDBCfg`;
- RAS/RAC administration through the platform `rac.exe`: discovery of clusters and infobases, listing sessions for one configured infobase, and explicit termination of selected session UUIDs;
- long-running command execution with `timeoutSeconds`;
- managed process operation tracking: `list_operations` shows operation id, PID, status, elapsed time and log path; `cancel_operation` terminates a running operation and its process tree;
- background execution for long operations: ready repository and load actions always return immediately with `operationId`; Advanced fallback commands support the explicit `background` option;
- safe argument-array command construction without shell.

For partial repository operations, prefer `objects`, an array of full root metadata names. Control Center creates the required 1C XML selection and marks every root with `includeChildObjects=true`. `objectsPath` is an advanced alternative and must point to an existing XML selection in the 1C `Objects` format. Examples of root names:

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
5. Call `infobase_load_objects` with the dump folder and root object names. The MCP selects the complete file set and updates the database configuration by default.
6. Call `repository_commit_objects` to commit the selected roots, or `repository_unlock_objects` to release them without a commit.

`run_repository_command`, `build_repository_arguments`, `run_ibcmd`, and `build_ibcmd_arguments` are Advanced fallback tools for operations not represented by a ready action. They are not required for ordinary get/lock/load/commit work.

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
