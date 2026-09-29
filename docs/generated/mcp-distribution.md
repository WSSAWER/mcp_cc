# MCP distribution policy

Generated from McpCatalogPolicy. The local `mcp-distribution.json` is not part of downloaded configuration updates and is never overwritten by the application updater. Applies to factory, shared and installed definitions, catalog refresh, save, install and start. Missing policy keeps the normal full edition. A local owner can intentionally change/remove this file; this is not an authentication boundary.

| ID | Condition | Result |
| --- | --- | --- |
| CAT-01 | No distribution manifest: ordinary unrestricted installation | Allow |
| CAT-02 | Restricted installation and exact MCP ID is in its allow-list | Allow |
| CAT-03 | Restricted installation and MCP ID is not in its allow-list | Block |
| CAT-04 | Manifest is malformed or unsupported: reject load/update, never fall back | Block |
