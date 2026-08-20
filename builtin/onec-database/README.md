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
- long-running command execution with `timeoutSeconds`;
- safe argument-array command construction without shell.

Repository project settings include designer connection arguments, repository address, repository user, and repository password. Passwords are never returned by `get_project`; only `repositoryPasswordConfigured` is returned.
