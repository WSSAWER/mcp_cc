# Designer readiness before UpdateDBCfg (generated)

Read only ConfigDumpInfo through the same Designer/connection/extension, inside the existing operation lease. Repeat every 30 seconds only for an explicit unfinished-save diagnostic. Cancellation and the caller's explicit deadline still apply; no automatic repair or UpdateDBCfg retry.

| Rule ID (first match) | Condition | Outcome | Message |
| --- | --- | --- | --- |
| designer.ready.structure-error | Probe reports an infobase structure error | Block | Infobase structure error during readiness check; no automatic retry or repair. |
| designer.ready.saving | Native output reports unfinished configuration saving | Wait | Configuration saving has not finished; UpdateDBCfg is not started. |
| designer.ready.readable | Probe exit 0 and fresh valid ConfigDumpInfo; no saving warning | Ready | Editable configuration is readable; UpdateDBCfg may start. |
| designer.ready.failure | Any other probe failure or missing/invalid index | Block | Configuration readiness could not be confirmed; UpdateDBCfg is blocked. |
