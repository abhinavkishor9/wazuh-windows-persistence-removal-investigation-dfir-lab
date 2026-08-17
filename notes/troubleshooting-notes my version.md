# Troubleshooting Notes

## Issue 1 – Task Payload Did Not Create execution.txt

### Problem

The task was triggered, but the expected:

`C:\PersistenceRemovalLab\execution.txt`

file was not created.

### Cause

The initial payload contained only:

```text
hostname
Get-Date
```

Those commands displayed information but did not write the output to a file.

### Resolution

A revised payload was created that writes execution information to `execution.txt`.

The original payload itself was still sufficient for studying task configuration and Task Scheduler telemetry.

---

## Issue 2 – Get-ScheduledTask Returned an Error After Removal

### Observation

After executing:

```powershell
Unregister-ScheduledTask -TaskName "DFIR-Persistence-Test" -Confirm:$false
```

the verification command returned:

```text
No MSFT_ScheduledTask objects found
```

### Interpretation

This was expected.

The task had been successfully removed from the current Task Scheduler inventory.

### Resolution

Treat the error as confirmation that the task no longer exists rather than as a failed investigation command.

---

## Issue 3 – Task Scheduler Search Did Not Show a Deletion Event

### Problem

The Task Scheduler search returned registration and launch events but did not display a specific deletion event.

### Observed

- Event ID 106 – Task registered
- Event ID 110 – Task launched
- Event ID 325 – Launch request queued

### Resolution

Do not claim that a deletion event was captured.

Use the following evidence instead:

- The `Unregister-ScheduledTask` command
- The post-removal `Get-ScheduledTask` result
- Historical registration and launch events
- Preserved XML task definition

Document the missing deletion event as an evidence limitation.

---

## Issue 4 – Sysmon Event ID 1 Showed schtasks.exe

### Observation

Sysmon recorded:

```text
C:\WINDOWS\System32\schtasks.exe
```

with Process ID:

`19552`

### Interpretation

This is relevant to scheduled-task management activity.

### Resolution

Correlate the event using:

- Timestamp
- Process ID
- Command line
- User
- Task Scheduler events

Do not automatically assume that every `schtasks.exe` event represents task deletion.

---

## Issue 5 – Wazuh Event Appeared Unrelated

### Observation

The captured Wazuh event showed an Adobe Create PDF process.

### Interpretation

This event did not directly correspond to `DFIR-Persistence-Test`.

### Resolution

Treat it as general endpoint telemetry only.

For a direct Wazuh correlation, search specifically for:

```text
DFIR-Persistence-Test
```

```text
schtasks.exe
```

or the relevant Task Scheduler events.

---

## Issue 6 – Task Scheduler Events Appeared in Different Timestamp Windows

### Problem

The process and Task Scheduler events were not all displayed at exactly the same timestamp.

### Cause

Task registration, launch, process creation, and other events occur at different stages of the task lifecycle.

### Resolution

Use a timeline window rather than requiring identical timestamps.

Correlate based on:

- Task name
- User
- Process
- Process ID
- Timestamp
- Event type

---

## Issue 7 – Task Definition Needed to Be Preserved Before Removal

### Problem

Once the scheduled task is deleted, its live configuration is no longer available through `Get-ScheduledTask`.

### Resolution

Export the task first:

```cmd
schtasks /query /tn "DFIR-Persistence-Test" /xml > C:\PersistenceRemovalLab\task-before-removal.xml
```

This preserves the task configuration for later analysis.

---

## Issue 8 – Task LastRunTime Was Not Useful

### Observation

`Get-ScheduledTaskInfo` returned an unexpected historical-looking `LastRunTime` value.

### Resolution

Use Task Scheduler Operational events as the primary execution timeline.

The event log provided explicit registration and launch events that were more useful for reconstructing the lab activity.

---

