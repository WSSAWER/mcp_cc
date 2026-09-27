# Temporary operation files (generated)

Only explicitly registered private paths are removed. Success and explicit cancellation clean immediately after child exit; failed/crashed operations keep temporary data until 7 days from filesystem CreationTimeUtc. No retention timestamp is stored. Retained/locked paths are checked once per minute and at owner startup. Caller input/output, settings, source checkouts and databases are excluded. PID plus process creation time protects live work and PID reuse. Unknown legacy files are not guessed to be disposable. Command/native logs have separate 30-day retention by CreationTimeUtc; active logs are protected.

| Rule | Condition | Remove |
| --- | --- | --- |
| temp.owner_active | Scope is open and owner is alive or unknown | False |
| temp.child_unconfirmed | Child launch/exit is not confirmed | False |
| temp.failure_week | Failed operation's file/directory was created less than 7 days ago | False |
| temp.finished | Scope closed or owner exited, all children exited | True |
