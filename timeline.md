# Investigation Timeline

## Lab 53 – Persistence Removal Investigation

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

---

# Persistence Lifecycle

```text
Investigation Workspace Created
        ↓
payload.ps1 Created
        ↓
DFIR-Persistence-Test Registered
        ↓
Task Definition Exported
        ↓
Task Launched
        ↓
Process Telemetry Generated
        ↓
Task Removed
        ↓
Live Task No Longer Exists
        ↓
Historical Telemetry Reviewed
        ↓
DFIR Assessment
```

---

# Creation Timeline

### 17-08-2026 13:26:47

Task Scheduler Event ID 106 recorded:

```text
User "DESKTOP-9MMM37V\Dell" registered Task Scheduler task "\DFIR-Persistence-Test"
```

Assessment:

The persistence mechanism was successfully registered by the local Dell account.

---

# Execution Timeline

### 17-08-2026 13:30:23

Task Scheduler recorded:

```text
Event ID 110 – Task launched
```

and:

```text
Event ID 325 – Launch request queued
```

Assessment:

The persistence mechanism reached the execution stage.

---

# Process Timeline

### 17-08-2026 13:28:36

Sysmon Event ID 1 recorded:

```text
Image:
C:\WINDOWS\System32\schtasks.exe

ProcessId:
19552

Description:
Task Scheduler Configuration Tool
```

Assessment:

Scheduled-task management activity was visible through process telemetry.

The timestamp does not by itself prove that this specific `schtasks.exe` invocation removed the task.

---

# Removal Timeline

After the persistence mechanism had been documented and executed, the scheduled task was removed with:

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

Assessment:

The task no longer existed in the current Task Scheduler configuration.

---

# Historical Evidence

The persistence mechanism continued to have historical evidence after removal:

```text
task-before-removal.xml
```

and Task Scheduler events including:

```text
106 – Task registered
110 – Task launched
325 – Launch request queued
```

This demonstrates the difference between:

```text
Current State
```

and:

```text
Historical State
```

---

# Wazuh Evidence

Wazuh endpoint:

```text
Agent ID:
001

Agent Name:
DESKTOP-9MMM37V

Status:
Active
```

The captured Wazuh evidence showed endpoint process telemetry but was associated with Adobe Create PDF rather than the scheduled-task removal.

Therefore, it was treated as evidence of Wazuh visibility rather than direct proof of the task deletion.

---

# Evidence Correlation Timeline

```text
Task Registered
      ↓
Task Action Documented
      ↓
Task Definition Exported
      ↓
Task Launched
      ↓
Task Management Process Observed
      ↓
Task Unregistered
      ↓
Task No Longer Exists
      ↓
Historical Evidence Remains
```

---

# Final Timeline Assessment

The evidence establishes that:

- `DFIR-Persistence-Test` existed.
- It was registered by `DESKTOP-9MMM37V\Dell`.
- It was configured to execute PowerShell.
- It was launched.
- Scheduled-task management telemetry was generated.
- The task was later removed.
- The live task no longer existed after removal.
- Historical evidence remained available.

The screenshots do not establish a specific Task Scheduler deletion event or a direct Wazuh event proving the removal.

---

# Final Analyst Conclusion

The lab successfully demonstrated persistence lifecycle reconstruction.

The strongest evidence came from the preserved task definition, Task Scheduler registration and launch events, Sysmon process telemetry, the explicit `Unregister-ScheduledTask` action, and the subsequent absence of the task.

The key DFIR lesson is that **removing persistence changes the current endpoint state, but it does not necessarily eliminate the historical evidence that the persistence mechanism existed.**
