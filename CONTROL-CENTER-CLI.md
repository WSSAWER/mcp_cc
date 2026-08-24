# MCP Control Center CLI and admin functions

This file is shipped with the product next to `McpControlCenter.exe`.

Use the CLI for local command line, PowerShell, scheduled tasks, or SSH sessions. Run commands from the folder containing `McpControlCenter.exe`, or pass `--root <appDir>` when controlling another portable installation.

## Basic syntax

```powershell
.\McpControlCenter.exe --cli <command> [options]
```

Most commands print JSON.

Examples:

```powershell
.\McpControlCenter.exe --cli list
.\McpControlCenter.exe --cli status --id code-index
.\McpControlCenter.exe --cli start --id code-index
.\McpControlCenter.exe --cli stop --id code-index
.\McpControlCenter.exe --cli restart --id code-index
```

`run_command` / shell execution is intentionally not available through CLI.

## MCP service commands

### list

Lists MCP definitions known to this Control Center root.

```powershell
.\McpControlCenter.exe --cli list
```

Returns id, name, Active flag, Start on launch flag, ports, MCP path, and installed state.

### status

Returns installed/running status. Without `--id`, returns all MCPs.

```powershell
.\McpControlCenter.exe --cli status
.\McpControlCenter.exe --cli status --id vanessa-automation
```

### start / stop / restart

Controls one MCP through the same runtime PID records and start/stop logic used by the UI.

```powershell
.\McpControlCenter.exe --cli start --id filesystem
.\McpControlCenter.exe --cli stop --id filesystem
.\McpControlCenter.exe --cli restart --id filesystem
```

`restart` stops a running MCP and starts it again. For stopped MCPs, it starts them.

### logs

Reads available MCP log tails.

```powershell
.\McpControlCenter.exe --cli logs --id code-index --lines 200
```

## Port commands

### set-ports

Sets the internal upstream port and external Gate port for one MCP.

```powershell
.\McpControlCenter.exe --cli set-ports --id code-index --internal-port 11605 --external-port 11505
```

Default behaviour:

- saves the new ports;
- if the MCP is stopped, nothing else is started;
- if the MCP is running, Control Center stops it and starts it again with the new ports;
- if restart fails, Control Center restores the previous ports and attempts to restart the MCP on the previous ports.

Save without hot apply:

```powershell
.\McpControlCenter.exe --cli set-ports --id code-index --internal-port 11615 --external-port 11515 --no-apply
```

Allowed MCP ports are `1024..65535`; internal and external ports must differ.

### set-gate

Sets common Gate settings. When the UI is running, it receives a reload signal and restarts the Unified endpoint on the new settings.

```powershell
.\McpControlCenter.exe --cli set-gate --unified-port 11500 --bind-address 127.0.0.1 --output-limit 60000
```

Optional fields:

```powershell
.\McpControlCenter.exe --cli set-gate --public-host 192.168.17.70 --remote-probe-url "https://example/probe?url={url}"
```

`unified-port` may be below `1024`; in that case the value is saved but the Unified endpoint remains closed until an openable port is set.

## Configuration commands

### get-config

Returns one full MCP definition.

```powershell
.\McpControlCenter.exe --cli get-config --id vanessa-automation
```

### update-config

Patches safe fields in one MCP definition.

```powershell
.\McpControlCenter.exe --cli update-config --id code-index --patch-json "{\"enabled\":true,\"startOnLaunch\":true}"
```

Common patch fields:

- `enabled`
- `startOnLaunch`
- `name`
- `internalPort`
- `externalPort`
- `mcpPath`
- `healthPath`
- `workingDirectory`
- `entry`
- `rawCommandArguments`
- `environment`

For port changes prefer `set-ports`, because it has validation and optional hot apply.

## Update commands

### update-configs

Downloads the latest `mcp_configs.json` from the configured public payload Git repository and refreshes shared MCP definitions.

```powershell
.\McpControlCenter.exe --cli update-configs
```

Optional source override:

```powershell
.\McpControlCenter.exe --cli update-configs --repository git@github.com:WSSAWER/mcp_cc.git --branch main
```

Keeps local paths, ports, Active flags, runtime folders, and install variable values.

### update-app

Prepares or applies a Control Center update from the configured public payload Git repository.

Prepare only:

```powershell
.\McpControlCenter.exe --cli update-app
```

Apply update:

```powershell
.\McpControlCenter.exe --cli update-app --apply
```

`--apply` closes/replaces app files through the updater flow. MCP folders, logs, runtime PID files, gate settings, and admin token are kept.

## Admin information and audit

### admin-info

Shows local admin file paths without revealing the token.

```powershell
.\McpControlCenter.exe --cli admin-info
```

### audit-tail

Reads the admin audit log tail.

```powershell
.\McpControlCenter.exe --cli audit-tail --lines 50
```

## Admin MCP / Unified access

The same backend is exposed to AI clients through Unified MCP.

Default regular connection:

```text
http://<host>:<unified-port>/mcp
```

Admin tools are hidden unless the client sends:

```text
Authorization: Bearer <admin-token>
```

The token is stored locally in:

```text
admin-token.json
```

With Bearer token, Unified exposes `control-center-admin.*` tools. Without Bearer token, admin tools are not listed.

Admin MCP tools include:

- `list_mcps`
- `get_mcp_config`
- `update_mcp_config`
- `set_mcp_ports`
- `set_gate_settings`
- `get_status`
- `start_mcp`
- `stop_mcp`
- `restart_mcp`
- `read_logs`
- `update_mcp_configurations`
- `update_control_center`
- `audit_tail`
- `run_command`

`run_command` is available only through authenticated admin MCP, requires `dangerouslyAllow=true`, and is audited. It is not available through CLI.

## UI functions summary

- **Start**: starts every installed Active MCP that is not already running.
- **Stop all**: stops every running MCP.
- **Restart**: restarts running Active MCPs and starts stopped Active MCPs.
- **Refresh status**: updates table status, PID, health, tools, and Gate state without starting/stopping MCPs.
- **Services -> Add MCP**: adds a shared MCP definition.
- **Services -> Copy unified URL**: copies the single aggregate MCP endpoint.
- **Services -> Add npm/Git/Python to PATH**: registers existing tools in user PATH.
- **Services -> Gate remote check settings**: configures public host and optional external probe.
- **Services -> Enable network access**: configures HTTP.sys/firewall access for external Gate ports.
- **Services -> Update all MCP configurations**: refreshes MCP product definitions from Git.
- **Services -> Update Control Center from Git**: updates the Control Center executable payload.
- **Help -> CLI and admin help**: opens this file.
