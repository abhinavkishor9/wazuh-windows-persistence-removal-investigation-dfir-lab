# Investigation Notes

## Lab Summary

This investigation examined the lifecycle of a controlled scheduled-task persistence mechanism, from creation through execution and eventual removal.

A harmless PowerShell payload was associated with the scheduled task `DFIR-Persistence-Test`. The task definition was preserved before removal, Task Scheduler telemetry was reviewed, Sysmon process activity was examined, and Wazuh endpoint visibility was assessed.

The post-removal state confirmed that the task no longer existed, while historical artifacts preserved evidence of the task's previous existence and activity.

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

A Windows endpoint is suspected of having a persistence mechanism that is no longer present.

The analyst needs to determine whether the persistence mechanism previously existed, what it executed, when it was registered, whether it ran, and whether evidence of its removal remains.

The investigation uses preserved task configuration and historical endpoint telemetry to reconstruct the persistence lifecycle.

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

## DFIR Analysis

The persistence lifecycle can be reconstructed from multiple evidence sources.

The task was created and registered by the Dell account, its action was documented, and its XML definition was preserved. Task Scheduler logs then confirmed registration and launch activity.

After `Unregister-ScheduledTask` was executed, the task no longer appeared in the live Task Scheduler inventory.

Historical Task Scheduler telemetry and the exported task definition remained valuable because they preserved information that was no longer visible in the current system state.

---

## Analyst Observations

- Persistence artifacts should be preserved before deletion when possible.
- Task Scheduler Operational logs can provide historical evidence.
- Event ID 106 established task registration.
- Event ID 110 established launch activity.
- Event ID 325 provided launch-queue context.
- Sysmon Event ID 1 showed `schtasks.exe` activity.
- The live task disappeared after removal.
- Historical task telemetry remained after deletion.
- The displayed Wazuh event was unrelated to the persistence task.
- A specific deletion event was not confirmed from the captured evidence.
- Current-state absence does not prove that persistence never existed.

---

## Investigation Assessment

The evidence confirms that:

- `DFIR-Persistence-Test` existed.
- It was registered by `DESKTOP-9MMM37V\Dell`.
- Its action executed PowerShell.
- It generated Task Scheduler execution telemetry.
- It was later removed.
- The task no longer existed after removal.

The evidence does not independently establish:

- A specific Task Scheduler deletion-event record.
- A malicious reason for the removal.
- Malicious intent associated with the scheduled task.

The activity was intentionally created and removed as part of the lab.

---

## Conclusion

The investigation demonstrated how a deleted persistence mechanism can still be reconstructed using preserved artifacts and historical telemetry.

The strongest evidence came from the task definition, Task Scheduler registration and launch events, Sysmon process telemetry, and the post-removal verification.

The lab reinforced the DFIR principle that **current endpoint state and historical endpoint state are not the same thing**.
