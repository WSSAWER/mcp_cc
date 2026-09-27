# Designer XML export selection (generated)

First matching selection rule wins. The existing managed operation lifecycle controls queue, PID, cancellation and completion. No repository refresh, capture or database update is performed.

| Rule ID | Condition | Outcome | Message |
| --- | --- | --- | --- |
| export.invalid_name | Empty, oversized or multiline name | Reject | Pass Configuration or one root metadata name, not a path or a name list. |
| export.configuration | Exact Configuration or Конфигурация (case insensitive) | ConfigurationBundle | One Designer process; listFile contains only Configuration; return Configuration.xml and its root Ext files, never other roots. |
| export.configuration_suffix | Configuration or Конфигурация followed by a dot/suffix | Reject | Use exactly Configuration, not Configuration.xml or a configuration name with a suffix. |
| export.named_root | Named metadata root with a type separator | NamedRoot | Discover ConfigDumpInfo, then export the selected root and its standalone child files; embedded parts stay inside parent XML. |
| export.missing_type | Any other name | Reject | objectName must be Configuration or a full metadata name, for example CommonModule.Example. |
| export.multiple_roots | Nonempty objects array; each name passes the rules above | One operation, at most one discovery and one export | Union standalone entries; Configuration never expands to other roots. |
| export.configuration_destination | Any selection containing Configuration: destination absent or empty | Block otherwise | Configuration export requires a new or empty outputPath; existing files are never removed or overwritten. |
