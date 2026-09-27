# Sync scheduling (generated)

First matching rule wins. The active-cycle guard runs inside the database schedule lock.

| Rule ID | Condition | Outcome | Message |
| --- | --- | --- | --- |
| sync.schedule.running_same | Cycle running; sync_auto_enable repeats the enabled interval | Unchanged | Keep the existing cycle and schedule unchanged. |
| sync.schedule.running | Cycle running; any other scheduling request | Reject | sync_running: inspect sync_job_status_get; do not start a duplicate. |
| sync.schedule.auto | No running cycle; sync_auto_enable | EnableAutomatic | Enable automatic polling with the requested interval; schedule now. |
| sync.schedule.now_auto | No running cycle; sync_once_start; automatic mode already enabled | ExpediteAutomatic | Schedule now; preserve automatic mode and its saved interval. |
| sync.schedule.now_once | No running cycle; sync_once_start; new or disabled job | RunOnce | Schedule one cycle without enabling automatic mode; preserve the saved interval. |
