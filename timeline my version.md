# Investigation Timeline

| Time | Activity | Evidence | Significance |
|---|---|---|---|
| 13:25 | Investigation workspace created | PowerShell | Establishes controlled lab location |
| 13:26:47 | Scheduled task registered | Task Scheduler Event 106 | Confirms task creation and user attribution |
| 13:28:00 | Task definition exported | XML artifact | Preserves pre-removal persistence state |
| 13:28:36 | `schtasks.exe` process observed | Sysmon Event ID 1 | Scheduled-task management telemetry |
| 13:30:23 | Task launch request observed | Task Scheduler Event 325 | Execution context |
| 13:30:23 | Task launched | Task Scheduler Event 110 | Confirms scheduled-task execution |
| After execution | Task removed | `Unregister-ScheduledTask` | Persistence removal action |
| After removal | Task no longer found | `Get-ScheduledTask` | Confirms current-state removal |
| After removal | Historical events reviewed | Task Scheduler Operational | Previous activity remains available |
| Investigation | Wazuh endpoint reviewed | Wazuh Discover | Centralized endpoint visibility |

