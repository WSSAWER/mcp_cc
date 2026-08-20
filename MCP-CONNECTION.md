# MCP client connection guide

Portable product files:

- `McpControlCenter.exe` - Control Center UI.
- `mcp_configs.json` - shared MCP definitions.
- `CONTROL-CENTER-HELP.md` - UI help.
- `MCP-CONNECTION.md` - this guide.

MCP implementations are not bundled. They are installed under `mcps\<id>` only when the user runs **Install**.

## URLs

Control Center provides one preferred endpoint for clients:

- **Unified URL**: one Streamable HTTP MCP endpoint that aggregates tools from every currently running Active MCP.

Use **Copy unified URL** in the toolbar or row context menu. By default it is `http://<Gate bind IP>:11500/mcp`, with the port configured in **Unified port**.

Each MCP also keeps two diagnostic/compatibility endpoints:

- **Internal URL**: direct local upstream URL, useful for local diagnostics.
- **Gate URL**: individual Control Center HTTP gate URL for this one MCP.

Use the row context menu:

- **Copy internal URL** copies the local upstream MCP URL.
- **Copy Gate URL** copies the configured gate MCP URL.
- **Copy unified URL** copies the preferred single MCP URL.

For network access, set **Gate bind IP**, save it, run **Services -> Enable network access**, then restart already-running MCPs if their individual gate bind changed. Use the copied Unified URL in the MCP client. The unified endpoint lists only MCPs that are enabled and already running; it does not auto-start MCPs.

## Configuration

The visible MCP rows come from `mcp_configs.json` and installed `mcps\<id>\mcp.json` runtime copies.

If an MCP needs machine-specific paths or names, they are configured as install variables in **Edit -> Install**. Normal **Install** asks for variables marked with **Prompt** and saves the answers. **Update / Reinstall** reuses the saved values.

Do not edit connection URLs by hand unless you intentionally changed the MCP ports or paths in Control Center.

## Vanessa Automation MCP

For Vanessa Automation use the `Vanessa Automation MCP` profile. It gates the native Vanessa Automation Streamable HTTP MCP server.

A client should connect to the copied **Unified URL**. The direct local upstream URL and individual Gate URL are for diagnostics/compatibility only.

Before connecting a client, make sure the Vanessa Automation side is prepared:

- `client_mcp.cfe` MCP extension is installed in the test manager infobase;
- VanessaExt is enabled;
- `VAExtension*.cfe` is installed in the tested infobase;
- the MCP endpoint answers `tools/list`.

Control Center exposes this as one `Vanessa Automation MCP` profile. The profile **Install** action clones Vanessa Automation and downloads the required release artifacts into `mcps\vanessa-automation\.install`. In **Edit -> Install** configure the 1C start path, the test manager start arguments, extension names, and the names of two `1C Database Manager` projects: one for the manager infobase and one for the tested client infobase.

The numbered profile commands install/update the MCP extension and VAExtension through `1C Database Manager`. The manager projects hold the actual `1cv8/ibcmd` paths and database connection arguments. These commands require confirmation because they update 1C infobases.

If Vanessa Automation is already running its MCP server on the configured upstream host/internal port, Control Center can adopt it. For the current local Vanessa server this is `localhost:9874`, but these are profile settings, not global defaults. If a matching 1C/Vanessa process is already running but the port is not open yet, Control Center waits for that process instead of launching a duplicate 1C session. Only when no matching process is found does Start use the configured 1C command to open Vanessa Automation with `runMcp`.

The profile stores full 1C connection arguments, not only an infobase string. Use `/IBName"BaseName"` for a registered local base, or `/IBConnectionString "..."` when the base is configured by connection string.

Known Vanessa MCP contract:

- health root: `GET http://<host>:<port>/` -> `200 text/plain` with `MCP server`;
- MCP endpoint: `POST http://<host>:<port>/mcp`;
- transport: Streamable HTTP, `text/event-stream` responses;
- `initialize` returns `Mcp-Session-Id`; pass that header to `notifications/initialized`, `tools/list`, and `tools/call`;
- do not use root `POST`, `/sse`, or `/messages`.
