# MCP Control Center

Noncommercial use only. Commercial use requires a separate written agreement.
See LICENSE.txt; third-party components retain their own licenses.

Requires Windows x64 and .NET 7 Desktop Runtime. Git is needed to install or
update Git-based MCPs; private repositories also require your Git credentials.

1. Start `McpControlCenter.exe` normally, without administrator rights.
2. Right-click an MCP to Install, Edit its local settings, then Start it.
3. Set a free Unified port (1024-65535) and press Gate.
4. Connect an MCP client using Streamable HTTP at
   `http://127.0.0.1:<Unified port>/mcp`. Services can copy the Unified URL.

For one Active MCP, use the same port at `/mcp/<configured-id>`.
Ordinary MCPs keep native tool names and sessions; embedded Admin is stateless
and requires its Bearer on every request.

- **Services -> Install ASP.NET Core Runtime 7 (x64)** downloads and opens the official Microsoft installer, which may request UAC. .NET 7 is out of support.

Port 0 disables an endpoint. A separate external port for each MCP is optional.
Network access also needs the intended bind address and firewall permissions;
do not expose file/database tools to an untrusted network.

If using the built-in Admin MCP, configure the client's
`Authorization: Bearer <token>` header. Keep the token private.

Settings and `mcps` are stored beside the EXE, so use a writable folder.
Update / Reinstall -> Program preserves each MCP's `.generated` data;
All deletes it. Help on an MCP opens that program's own startup instructions.
The window close button hides CC in the tray; use Exit to close it.
