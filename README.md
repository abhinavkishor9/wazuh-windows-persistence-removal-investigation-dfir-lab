# Windows Persistence Removal Investigation with Wazuh (DFIR Lab 53)

## Overview

Persistence mechanisms are often investigated not only by looking for what currently exists on an endpoint, but also by determining whether a persistence mechanism existed previously and was later removed.

In this lab, a harmless scheduled task named `DFIR-Persistence-Test` was created as a controlled persistence mechanism. The task executed a PowerShell payload from `C:\PersistenceRemovalLab\payload.ps1`.

The task was documented, executed, and subsequently removed. Windows Task Scheduler logs, Sysmon Event ID 1, Windows process telemetry, PowerShell activity, and Wazuh endpoint visibility were used to reconstruct the persistence lifecycle.

---

# Lab Objectives

- Understand persistence removal from a DFIR perspective.
- Establish the state of a scheduled-task persistence mechanism.
- Preserve the task definition before removal.
- Identify when the task was registered and executed.
- Determine the process associated with task management.
- Confirm that the persistence mechanism was removed.
- Compare endpoint state before and after removal.
- Correlate persistence activity with Sysmon and Windows telemetry.
- Identify evidence that remains after a persistence mechanism is deleted.
- Document evidence limitations without making unsupported conclusions.

---

# Lab Environment

| Component          | Value                                      |
| ------------------ | ------------------------------------------ |
| Host OS            | Windows 11 Pro                             |
| SIEM               | Wazuh 4.12                                 |
| Endpoint Agent     | Wazuh Agent                                |
| Endpoint Name      | DESKTOP-9MMM37V                            |
| Agent ID           | 001                                        |
| Persistence Type   | Scheduled Task                             |
| Task Name          | DFIR-Persistence-Test                      |
| Lab Directory      | C:\PersistenceRemovalLab                   |
| Payload            | payload.ps1                                |
| Primary Telemetry  | Task Scheduler / Sysmon / Windows Security |
| Investigation UI   | Wazuh Discover                             |

---

# Tools Used

- PowerShell
- Task Scheduler
- `Register-ScheduledTask`
- `Get-ScheduledTask`
- `Get-ScheduledTaskInfo`
- `Start-ScheduledTask`
- `Unregister-ScheduledTask`
- `schtasks.exe`
- Windows Event Viewer
- Task Scheduler Operational Log
- Sysmon Event ID 1
- Windows Security Event ID 4688
- Wazuh Discover
- Wazuh Agent

---

# Investigation Scenario

A SOC analyst suspects that a scheduled task may have been used as a persistence mechanism on a Windows endpoint.

When the analyst examines the current system state, the task is no longer present. The investigation must determine whether the task previously existed, what it executed, when it ran, and whether the removal itself can be reconstructed from historical telemetry.

The investigation therefore compares the known pre-removal state with the post-removal state and uses historical evidence to reconstruct the persistence lifecycle.

---

# Investigation Workflow

```text
Create Persistence
        ↓
Document Persistence
        ↓
Preserve Task Definition
        ↓
Execute Task
        ↓
Collect Process Telemetry
        ↓
Remove Persistence
        ↓
Verify Task Is Gone
        ↓
Search Historical Telemetry
        ↓
Correlate With Wazuh
        ↓
Reconstruct Timeline
        ↓
Assess Persistence Removal
```

---

# Investigation Steps

### Step 1 – Verify Wazuh Agent

On the Wazuh server:

```bash
sudo /var/ossec/bin/agent_control -i 001
```

Observed:

- Agent ID: `001`
- Agent Name: `DESKTOP-9MMM37V`
- Status: `Active`
- Operating System: Windows 11 Pro
- Wazuh Version: `4.12.0`

---

### Step 2 – Create Investigation Workspace

```powershell
New-Item -Path "C:\PersistenceRemovalLab" -ItemType Directory -Force
```

The workspace was used to hold the controlled payload and preserved task definition.

---

### Step 3 – Create Harmless Persistence Payload

A harmless PowerShell payload was created:

```powershell
@'
hostname
Get-Date
'@ | Set-Content "C:\PersistenceRemovalLab\payload.ps1"
```

The payload contained only basic host and time information commands.

---

### Step 4 – Create Scheduled Task

The scheduled task was created with:

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

Observed task state:

`Ready`

---

### Step 5 – Verify the Persistence

```powershell
Get-ScheduledTask -TaskName "DFIR-Persistence-Test"
```

The task existed under the root task path.

Then:

```powershell
Get-ScheduledTaskInfo -TaskName "DFIR-Persistence-Test"
```

The task information was collected for the baseline.

---

### Step 6 – Inspect the Task Action

```powershell
(Get-ScheduledTask -TaskName "DFIR-Persistence-Test").Actions |
Format-List *
```

Observed action:

```text
Execute   : powershell.exe
Arguments : -NoProfile -File C:\PersistenceRemovalLab\payload.ps1
```

This established exactly what the persistence mechanism was configured to execute.

---

### Step 7 – Preserve the Task Definition

```cmd
schtasks /query /tn "DFIR-Persistence-Test" /xml > C:\PersistenceRemovalLab\task-before-removal.xml
```

The XML file was preserved before deletion.

Observed artifact:

```text
C:\PersistenceRemovalLab\task-before-removal.xml
```

This provides a forensic copy of the persistence configuration after the live task is removed.

---

### Step 8 – Generate Execution Evidence

The task was manually triggered:

```powershell
Start-ScheduledTask -TaskName "DFIR-Persistence-Test"
```

The investigation attempted to validate execution through the payload and Task Scheduler telemetry.

Task Scheduler Operational events showed the task lifecycle, including registration and launch activity.

---

### Step 9 – Review Task Scheduler Operational Log

Open:

```text
Event Viewer
→ Applications and Services Logs
→ Microsoft
→ Windows
→ TaskScheduler
→ Operational
```

Observed evidence included:

- Event ID 106 – Task registered
- Event ID 110 – Task launched
- Event ID 325 – Launch request queued

The registration event showed:

```text
User "DESKTOP-9MMM37V\Dell" registered Task Scheduler task "\DFIR-Persistence-Test"
```

The task was registered at:

`17-08-2026 13:26:47`

The task was launched at approximately:

`17-08-2026 13:30:23`

---

### Step 10 – Review Sysmon Event ID 1

Sysmon Event ID 1 was reviewed for task-management process activity.

Observed:

```text
Image:
C:\WINDOWS\System32\schtasks.exe

ProcessId:
19552

Description:
Task Scheduler Configuration Tool

Logged:
17-08-2026 13:28:36
```

This provided process-level evidence associated with scheduled-task management.

---

### Step 11 – Remove the Persistence

The scheduled task was removed:

```powershell
Unregister-ScheduledTask `
    -TaskName "DFIR-Persistence-Test" `
    -Confirm:$false
```

The task was then checked:

```powershell
Get-ScheduledTask -TaskName "DFIR-Persistence-Test"
```

Observed result:

```text
No MSFT_ScheduledTask objects found
```

This confirmed that the task no longer existed in the current Task Scheduler state.

---

### Step 12 – Search Historical Task Scheduler Events

The Task Scheduler Operational log was searched:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-TaskScheduler/Operational" -MaxEvents 100 |
Where-Object {$_.Message -match "DFIR-Persistence-Test"} |
Select-Object TimeCreated, Id, Message
```

Historical activity associated with the task remained visible.

The captured results showed registration and launch events.

A specific deletion event was not demonstrated in the captured results.

---

### Step 13 – Review Wazuh

Wazuh Discover was reviewed for:

```text
agent.name: DESKTOP-9MMM37V
```

The endpoint was confirmed as active.

The captured Wazuh process telemetry contained endpoint process information, but the shown event was associated with Adobe Create PDF rather than the controlled scheduled-task deletion.

Therefore, the Wazuh evidence was treated as confirmation of endpoint visibility rather than direct proof of task removal.

---

# Key Findings

- Wazuh Agent `001` was active.
- The test task `DFIR-Persistence-Test` was successfully created.
- The task was configured to execute PowerShell.
- The payload was located at `C:\PersistenceRemovalLab\payload.ps1`.
- A task-definition XML backup was preserved before removal.
- Task Scheduler Event ID 106 showed registration by `DESKTOP-9MMM37V\Dell`.
- Registration occurred at approximately `17-08-2026 13:26:47`.
- Task Scheduler Event ID 110 and Event ID 325 showed launch activity at approximately `17-08-2026 13:30:23`.
- Sysmon Event ID 1 recorded `schtasks.exe` with Process ID `19552`.
- The task was removed using `Unregister-ScheduledTask`.
- A subsequent `Get-ScheduledTask` query confirmed that the task no longer existed.
- Historical Task Scheduler evidence remained available after removal.
- A specific task-deletion event was not demonstrated in the captured screenshots.
- The captured Wazuh event was unrelated to the scheduled-task removal.

---

# Evidence Correlation

| Evidence | Source | Observation | DFIR Value |
|---|---|---|---|
| Wazuh Agent | Wazuh Manager | Agent `001` active | Endpoint visibility |
| Persistence Workspace | PowerShell | `C:\PersistenceRemovalLab` | Controlled evidence location |
| Persistence Task | Task Scheduler | `DFIR-Persistence-Test` existed | Establishes persistence |
| Task Action | PowerShell | PowerShell payload configured | Identifies execution |
| Task Backup | `schtasks.exe` | XML preserved | Preserves pre-removal state |
| Task Registration | Task Scheduler Event 106 | Dell registered task | Attribution |
| Task Launch | Task Scheduler Event 110 | Task launched | Execution evidence |
| Launch Queue | Task Scheduler Event 325 | Launch request queued | Execution context |
| Task Management | Sysmon Event ID 1 | `schtasks.exe`, PID 19552 | Process evidence |
| Task Removal | PowerShell | `Unregister-ScheduledTask` | Current-state change |
| Post-removal check | PowerShell | Task no longer exists | Confirms removal |
| Wazuh Telemetry | Wazuh Discover | Endpoint activity visible | SIEM visibility |

---

# DFIR Analysis

The investigation demonstrates why persistence-removal analysis should preserve the persistence mechanism before deleting it.

The scheduled task existed, its action was documented, and an XML representation of the task was preserved. Task Scheduler telemetry then provided historical evidence of registration and launch activity.

After `Unregister-ScheduledTask` was executed, the task disappeared from the current Task Scheduler state. However, historical events and the preserved XML definition continued to provide evidence that the task previously existed.

The investigation therefore demonstrates the difference between **current endpoint state** and **historical forensic evidence**.

---

# Important Evidence Limitation

The captured Task Scheduler search showed task registration and launch events, but a specific task-deletion event was not present in the displayed results.

Similarly, the captured Wazuh event shown in the evidence was associated with Adobe Create PDF rather than `DFIR-Persistence-Test`.

Therefore, the investigation should conclude:

> The persistence mechanism was successfully created and later removed, and historical evidence confirms its previous existence and activity. The screenshots do not independently demonstrate a specific Wazuh task-deletion event.

---

# MITRE ATT&CK Context

Scheduled-task persistence can be associated with:

- `T1053.005 – Scheduled Task/Job: Scheduled Task`

Persistence removal can also be relevant to broader defense-evasion and indicator-removal investigations, depending on the circumstances.

The controlled task in this lab was benign.

---

# Skills Practiced

- Persistence Investigation
- Scheduled Task Analysis
- Persistence Removal Analysis
- Task Scheduler Event Analysis
- Sysmon Event ID 1
- Process Attribution
- PowerShell
- Windows Event Analysis
- Wazuh Discover
- Historical Evidence Reconstruction
- Evidence Preservation
- Timeline Construction
- DFIR Documentation

---

# Outcome

Successfully created, documented, executed, and removed a controlled scheduled-task persistence mechanism.

The investigation preserved the task definition before deletion, correlated Task Scheduler registration and launch events with process telemetry, confirmed that the task no longer existed after removal, and demonstrated how historical evidence can survive after a persistence mechanism has disappeared from the endpoint.

The lab also reinforced the importance of distinguishing confirmed telemetry from evidence that was not captured or could not be directly attributed.
