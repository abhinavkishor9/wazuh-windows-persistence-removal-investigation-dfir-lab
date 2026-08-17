# Investigation Notes

## Lab Summary

This investigation examined the lifecycle of a controlled scheduled-task persistence mechanism, from creation through execution and eventual removal.

---

## Analyst Methodology

1. Verify Wazuh agent status.
2. Create investigation workspace.
3. Create harmless persistence payload.
4. Register scheduled task.
5. Verify task existence.
6. Inspect task action.
7. Export task definition.
8. Trigger task execution.
9. Review Task Scheduler telemetry.
10. Review Sysmon Event ID 1.
11. Remove scheduled task.
12. Verify task removal.
13. Search historical Task Scheduler events.
14. Review Wazuh telemetry.
15. Correlate persistence lifecycle evidence.
16. Document evidence limitations.

---

## Investigation Scenario

A SOC analyst receives an alert indicating that a suspicious scheduled task may have been removed from a Windows endpoint.

When the analyst checks the system, the task no longer exists.

The investigation must determine:

Did the task actually exist?
What did the task execute?
When was it created?
When was it executed?
When was it deleted?
Which user/process removed it?
Is there evidence of the task remaining in Windows telemetry after deletion?
Can Wazuh reconstruct the activity?
Does the removal appear to be routine administration or potential cleanup of persistence?

---

## Evidence Collected

### Evidence 1 – Wazuh Agent

Collected:

- Agent ID: `001`
- Agent Name: `DESKTOP-9MMM37V`
- Status: `Active`
- Operating System: Windows 11 Pro
- Wazuh Version: `4.12.0`

Finding:

Confirmed that the Windows endpoint was actively reporting to Wazuh.

---

### Evidence 2 – Investigation Workspace

Collected:

`C:\PersistenceRemovalLab`

Finding:

Established a dedicated workspace for the controlled persistence investigation.

---

### Evidence 3 – Persistence Payload

Collected:

`C:\PersistenceRemovalLab\payload.ps1`

The initial payload contained:

```text
hostname
Get-Date
```

Finding:

Created a harmless PowerShell payload for scheduled-task execution.

---

### Evidence 4 – Scheduled Task Creation

Task Name:

`DFIR-Persistence-Test`

Creation command:

```powershell
$action = New-ScheduledTaskAction `
    -Execute "powershell.exe" `
    -Argument "-NoProfile -File C:\PersistenceRemovalLab\payload.ps1"

$trigger = New-ScheduledTaskTrigger -AtLogOn

Register-ScheduledTask `
    -TaskName "DFIR-Persistence-Test" `
    -Action $action `
    -Trigger $trigger `
    -Description "Controlled DFIR persistence removal lab task"
```

Finding:

The scheduled task was successfully created and reported a `Ready` state.

---

### Evidence 5 – Task Action

Command Used:

```powershell
(Get-ScheduledTask -TaskName "DFIR-Persistence-Test").Actions |
Format-List *
```

Observed:

```text
Execute:
powershell.exe

Arguments:
-NoProfile -File C:\PersistenceRemovalLab\payload.ps1
```

Finding:

The task's persistence action was identified and documented.

---

### Evidence 6 – Task Definition Preservation

Command Used:

```cmd
schtasks /query /tn "DFIR-Persistence-Test" /xml > C:\PersistenceRemovalLab\task-before-removal.xml
```

Collected:

`C:\PersistenceRemovalLab\task-before-removal.xml`

Finding:

A preserved copy of the task configuration was created before removal.

This provides historical evidence of the persistence mechanism even after the live task is deleted.

---

### Evidence 7 – Task Registration

Task Scheduler Operational evidence showed:

```text
Event ID: 106
```

Observed message:

```text
User "DESKTOP-9MMM37V\Dell" registered Task Scheduler task "\DFIR-Persistence-Test"
```

Observed timestamp:

`17-08-2026 13:26:47`

Finding:

The task was registered by the local Dell account.

---

### Evidence 8 – Task Launch

Task Scheduler Operational evidence showed:

```text
Event ID: 110
```

and:

```text
Event ID: 325
```

Observed timestamp:

`17-08-2026 13:30:23`

Finding:

Task Scheduler recorded the task launch and queued launch activity.

---

### Evidence 9 – Sysmon Process Creation

Sysmon Event ID 1 recorded:

```text
Image:
C:\WINDOWS\System32\schtasks.exe

ProcessId:
19552

Description:
Task Scheduler Configuration Tool
```

Observed timestamp:

`17-08-2026 13:28:36`

Finding:

The endpoint generated process telemetry associated with scheduled-task management.

---

### Evidence 10 – Task Removal

Command Used:

```powershell
Unregister-ScheduledTask `
    -TaskName "DFIR-Persistence-Test" `
    -Confirm:$false
```

Finding:

The scheduled task was removed from the current Task Scheduler configuration.

---

### Evidence 11 – Post-removal Verification

Command Used:

```powershell
Get-ScheduledTask -TaskName "DFIR-Persistence-Test"
```

Observed:

```text
No MSFT_ScheduledTask objects found
```

Finding:

The persistence mechanism no longer existed in the live Task Scheduler state.

---

### Evidence 12 – Historical Task Scheduler Search

Command Used:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-TaskScheduler/Operational" -MaxEvents 100 |
Where-Object {$_.Message -match "DFIR-Persistence-Test"} |
Select-Object TimeCreated, Id, Message
```

Observed:

- Event ID 106 – Task registered
- Event ID 110 – Task launched
- Event ID 325 – Launch request queued

Finding:

Historical telemetry remained available after the live task was removed.

A specific deletion event was not demonstrated in the captured results.

---

### Evidence 13 – Wazuh Telemetry

Wazuh Discover was reviewed for:

```text
agent.name: DESKTOP-9MMM37V
```

The endpoint was visible in Wazuh.

The captured Wazuh event displayed an Adobe Create PDF process event rather than direct telemetry for the scheduled-task removal.

Finding:

Wazuh endpoint visibility was confirmed, but the displayed Wazuh event was not used as direct proof of task deletion.

---

## Evidence Correlation

| Evidence | Finding | Significance |
|---|---|---|
| Task definition | `DFIR-Persistence-Test` | Confirms persistence mechanism |
| Task action | PowerShell payload | Shows what the task executed |
| XML backup | Preserved before removal | Maintains historical task state |
| Event 106 | Task registered by Dell | Attribution and creation evidence |
| Event 110 | Task launched | Execution evidence |
| Event 325 | Launch request queued | Task execution context |
| Sysmon 1 | `schtasks.exe`, PID 19552 | Process-level task management |
| Unregister command | Task removed | Removal action |
| Get-ScheduledTask | Task not found | Confirms current state |
| Wazuh | Endpoint telemetry available | Centralized visibility |
| Wazuh displayed event | Adobe Create PDF | Unrelated to task removal |

---
