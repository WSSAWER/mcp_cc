# MCP Control Center

Noncommercial use only. Commercial use requires a separate written agreement.
See LICENSE.txt; third-party components retain their own licenses.

Requires Windows x64 and .NET 7 Desktop Runtime. Git is needed to install or
update Git-based MCPs; private repositories also require your Git credentials.

1. Start `McpControlCenter.exe` normally, without administrator rights.
2. Right-click an MCP to Install, Edit its local settings, then Start it.
3. Set a free Unified port (1024-65535) and press Gate.
4. Connect an MCP client using Streamable HTTP at
   the URL copied by Services (HTTP by default; HTTPS when applied).

For one Active MCP, use the same port at `/<configured-id>/mcp`, for example
`/onec-database/mcp`. The earlier `/mcp/<configured-id>` path still works.
Ordinary MCPs keep native tool names and sessions; embedded Admin is stateless
and requires its Bearer on every request.

- **Services -> Install ASP.NET Core Runtime 7 (x64)** downloads and opens the official Microsoft installer, which may request UAC. .NET 7 is out of support.

Port 0 disables an endpoint. A separate external port for each MCP is optional.
Use **Services -> HTTPS settings** to select/import a Windows certificate or
create a self-signed certificate with client-facing DNS/IP names. Trust its
public CER on clients. **Save only** does not restart; **Save + Apply Unified**
disconnects Unified clients. Separate Gates change on their next Start/Restart.
Each listener serves HTTP or HTTPS, not both; errors never fall back to HTTP.
Private keys stay in Windows storage, not JSON.
Network access also needs the intended bind address and firewall permissions;
do not expose file/database tools to an untrusted network.

If using the built-in Admin MCP, configure the client's
`Authorization: Bearer <token>` header. Keep the token private.

Settings and `mcps` are stored beside the EXE, so use a writable folder.
Update / Reinstall -> Program preserves each MCP's `.generated` data;
All deletes it. Help on an MCP opens that program's own startup instructions.
The window close button hides CC in the tray; use Exit to close it.
