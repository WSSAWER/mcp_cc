# MCP Control Center

Requires Windows x64 and .NET 7 Desktop Runtime. Git is needed to install or
update Git-based MCPs; private repositories also require your Git credentials.

1. Start `McpControlCenter.exe` normally, without administrator rights.
2. Right-click an MCP to Install, Edit its local settings, then Start it.
3. Set a free Unified port (1024-65535) and press Gate.
4. Connect an MCP client using Streamable HTTP at
   `http://127.0.0.1:<Unified port>/mcp`. Services can copy the Unified URL.

Port 0 disables an endpoint. A separate external port for each MCP is optional.
Network access also needs the intended bind address and firewall permissions;
do not expose file/database tools to an untrusted network.

If using the built-in Admin MCP, configure the client's
`Authorization: Bearer <token>` header. Keep the token private.

Settings and `mcps` are stored beside the EXE, so use a writable folder.
Update / Reinstall -> Program preserves each MCP's `.generated` data;
All deletes it. Help on an MCP opens that program's own startup instructions.
The window close button hides CC in the tray; use Exit to close it.
