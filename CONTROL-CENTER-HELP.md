# MCP Control Center help

MCP Control Center manages local MCP servers and the built-in HTTP gate. It does not configure AI models, llama-server slots, GPU, model context, or artificial MCP connection limits.

For MCP clients, the preferred connection is the **Services -> Copy unified URL** address. It exposes one `/mcp` endpoint with tools from every Active MCP that is currently running.

## Main screen

- **Active** includes the MCP in bulk Start/Restart actions.
- **Start on launch** in **Edit -> Common** starts this MCP automatically when Control Center opens. **Active** still controls bulk Start/Restart and Unified visibility; it does not block an explicit Start on launch profile.
- **Status** is based on saved PID first, then process/port discovery.
- **Health**, **Commands**, and **Gate** are checked only while the MCP is running.
- **Gate** reports what Control Center can prove: `Local` means loopback-only, `Bound?` means the gate is listening locally but remote access was not verified, `Remote` means the configured external probe reached the gate, `Blocked` means that probe failed, `No bind` means the gate is not listening on the configured IP/port, and `Unavailable` means the gate check failed.
- **Internal port** is the local upstream port used by the worker.
- **External port** is the gate port used by MCP clients.
- **Last message** shows the latest useful process, health, command, or install message.

## Row context menu

- **Start / Stop / Restart / Check** affects only the clicked MCP row.
- **Copy internal URL** copies the local upstream MCP URL.
- **Copy Gate URL** copies the configured gate MCP URL.
- **Edit** opens MCP settings. For not-installed MCPs, only install/config/command settings are editable.
- **Install** downloads or restores a missing MCP from its configured npm/Git/built-in source. It does not start the MCP.
- **Update / Reinstall -> Program** removes program files and temporary install files, keeps generated files, then installs again.
- **Update / Reinstall -> All** removes the whole MCP folder, including generated files, then installs again.
- **Remove -> Program / Generated / All** removes the selected file scope without deleting the shared MCP definition.
- **Delete selected** removes the shared definition and MCP files.
- **Help** opens documentation for the clicked MCP. MCP-specific setup belongs there, not in this Control Center help.

## Application help

The toolbar **Help** menu opens Control Center documentation from the application folder:

- `CONTROL-CENTER-HELP.md` - this UI and folder-layout help.
- `CONTROL-CENTER-CLI.md` - command line, SSH, and admin MCP reference.
- `MCP-CONNECTION.md` - MCP client connection instructions.

## Install variables

In **Edit -> Install**, install variables define machine-specific tokens used by an MCP.

- `Name`: token name.
- `Value`: saved path/value.
- `Description`: text used in the install prompt.
- `Prompt`: if checked, normal **Install** asks for the described value and saves the answer.

**Update / Reinstall** does not ask again. It reuses the saved values. Tokens are substituted into MCP paths, launch arguments, user commands, post-install commands, log paths, and environment values.

## File layout

Each MCP lives under `mcps\<id>`:

- `server` - runnable MCP program files. Control Center starts MCPs and user commands with this folder as the working directory.
- `.install` - install/build/source materials used to create or repair `server`, when the MCP needs them.
- `.generated` - persistent working data created by the MCP or its commands: profiles, indexes, local databases, caches, and other runtime state.

Use **Program** reinstall for normal updates. Use **All** only when persistent generated data should also be removed.

Control Center action logs are stored under `logs\control-center\<mcp-id>`. MCP stderr logs stay near the MCP folder so the UI can show the latest process messages.

## Services menu

- **Add MCP** adds a shared MCP definition. Install files afterwards if an npm package, Git URL, Python install, or built-in source is configured.
- **Copy unified URL** copies the single aggregated MCP URL for all currently running MCPs.
- **Update all MCP configurations** downloads the latest `mcp_configs.json` from the same configured update Git payload used for Control Center releases. It updates MCP commands and product metadata and adds new shared MCP definitions. It keeps local paths, ports, Active flags, runtime folders, and install variable values.
- **Gate** saves the bind IP, unified MCP port, and output limit. The unified endpoint restarts immediately; individual MCP gates use changed settings on next Start/Restart.
- **Gate remote check settings** saves an optional public host and remote probe URL. The probe URL can use `{url}`, `{host}`, `{port}`, and `{path}` placeholders; a 2xx probe response changes Gate to `Remote`.
- **Enable network access** adds Windows Firewall rules for configured individual gate ports and the unified MCP port. Already-running MCPs must be restarted to use a changed bind IP. A local `Bound?` status still requires a remote probe or checking the copied URL from another device because firewall profile, VPN, router, and host network policy can block remote clients.
- **Update Control Center from Git** clones or updates the configured update payload repository, closes the current UI, copies only Control Center application files, and starts the updated executable. The repository must contain the portable update files in its root, not the full source project. It keeps `mcps`, `logs`, runtime PID files, gate settings, and admin token. Existing `mcp_configs.json` is read and re-saved through the app JSON serializer; new MCP definitions from the updated default config are added without overwriting existing local definitions.
- **Add npm/Git/Python to PATH** registers an already installed tool in PATH. It does not install Node.js, Git, or Python.

## Control Center Admin MCP

`Control Center Admin` is a built-in C# MCP for administering this Control Center instance. In the default product config it is Active and marked **Start on launch**, so the admin service starts with Control Center. It is still hidden from Unified tools unless the client supplies the admin token.

The admin token is generated locally on first use:

- `admin-token.json`

The token is the admin password. Do not paste it into public logs or shared chats. Give it to an AI agent only through the MCP client configuration or as the `token` argument for admin tools in a trusted local session.

Unified MCP hides admin tools by default. To expose them in Unified `tools/list`, pass the token as a Bearer token:

- `Authorization: Bearer <token>`

No URL token, custom header, or `initialize.params` token is supported. In Bearer-authorized Unified sessions, admin tool calls receive the token internally and do not expose a `token` argument in `tools/list`.

Admin tool calls are audited here:

- `logs\control-center\admin-mcp-audit.jsonl`

The audit log records tool name, sanitized arguments, status, and timestamp. The token is always written as `***`.

Initial admin tools:

- `list_mcps`
- `get_mcp_config`
- `update_mcp_config`
- `get_status`
- `start_mcp`
- `stop_mcp`
- `restart_mcp`
- `read_logs`
- `update_mcp_configurations`
- `update_control_center`
- `run_command`
- `audit_tail`

`run_command` requires both a valid token and `dangerouslyAllow=true`, plus an explicit working directory and timeout.

## Command line / SSH administration

The same administrative backend is available from the executable for command line and SSH usage:

- `McpControlCenter.exe --cli list`
- `McpControlCenter.exe --cli status --id <mcp-id>`
- `McpControlCenter.exe --cli start --id <mcp-id>`
- `McpControlCenter.exe --cli stop --id <mcp-id>`
- `McpControlCenter.exe --cli restart --id <mcp-id>`
- `McpControlCenter.exe --cli update-config --id <mcp-id> --patch-json "{\"enabled\":true,\"startOnLaunch\":true}"`
- `McpControlCenter.exe --cli update-configs`
- `McpControlCenter.exe --cli update-app --apply`

The CLI intentionally does not expose `run_command` / shell execution.

The full CLI/admin reference is in `CONTROL-CENTER-CLI.md` next to the executable.
