# Command log retention (generated)

| Rule | Condition | Delete |
| --- | --- | --- |
| log.active | Active command, background operation or unconfirmed child | False |
| log.recent | Filesystem CreationTimeUtc + 30 days is in the future | False |
| log.expired | Inactive log created at least 30 days ago | True |

Only service command/native logs are considered; no creation date is duplicated in settings. Links and Control Center install.log are excluded. Locked files are retried on the next cleanup.
