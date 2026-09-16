# Single Designer path migration (generated)

One saved configuratorPath per connection. Legacy configuratorPaths is accepted only while reading old configuration files, never by MCP commands. Ambiguous values are preserved as diagnostics and block Designer only. Set configuratorPath explicitly to resolve them.

| Explicit path | Distinct legacy paths | Malformed | Decision |
| --- | --- | --- | --- |
| True | 2 | False | designer.path.KeepExplicit |
| False | 0 | False | designer.path.Missing |
| False | 1 | False | designer.path.MigrateSingle |
| False | 2 | False | designer.path.BlockAmbiguous |
| False | 0 | True | designer.path.BlockAmbiguous |
