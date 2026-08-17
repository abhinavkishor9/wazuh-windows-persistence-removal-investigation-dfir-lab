# wazuh-windows-persistence-removal-investigation-dfir-lab

## Overview

Persistence removal investigation asks:

Did someone attempt to remove evidence of persistence, and can we reconstruct what existed before it was removed?

For example:

Persistence Exists
      ↓
Attacker / User Removes Persistence
      ↓
Artifact Disappears
      ↓
Residual Telemetry Remains
      ↓
Wazuh + Windows Events
      ↓
Reconstruct Previous State

This is important because attackers may remove:

Scheduled tasks
Services
Registry persistence
Startup entries
Malicious files
Persistence-related configuration

The removal itself can become evidence.

The key DFIR principle is:

Absence of a persistence artifact today does not prove that persistence never existed.

We therefore investigate removal evidence and residual evidence.


In this lab, a harmless scheduled task named `DFIR-Persistence-Test` was created as a controlled persistence mechanism. The task executed a PowerShell payload from `C:\PersistenceRemovalLab\payload.ps1`.

The task was documented, executed, and subsequently removed. Windows Task Scheduler logs, Sysmon Event ID 1, Windows process telemetry, PowerShell activity, and Wazuh endpoint visibility were used to reconstruct the persistence lifecycle.

---

# Lab Objectives

- Determine what evidence remains after a persistence mechanism is removed.
- Preserve and examine the original configuration before deletion.
- Establish the sequence from task registration through execution and removal.
- Attribute persistence-related actions to the responsible user and process.
- Compare live endpoint state with historical Task Scheduler evidence.
- Use process telemetry to identify task-management activity.
- Correlate scheduled-task evidence with Wazuh telemetry.
- Identify gaps where the available logs do not directly prove a specific action.
- Reconstruct the persistence lifecycle using multiple independent artifacts.
- Produce a defensible forensic assessment of the removal activity.

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

# MITRE ATT&CK Context

Scheduled-task persistence can be associated with:

- `T1053.005 – Scheduled Task/Job: Scheduled Task`

Persistence removal can also be relevant to broader defense-evasion and indicator-removal investigations, depending on the circumstances.

The controlled task in this lab was benign.

---

