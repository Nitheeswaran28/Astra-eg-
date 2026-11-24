# SOC-1 Windows Essentials

## 1. Lab Overview

Investigate how Windows represents accounts, running programs and startup configuration. Compare what the machine is configured to do with what its event logs say happened.

## 2. Lab Environment

One Windows lab machine with a graphical desktop, local management consoles and PowerShell. The exercises inspect local state.

## 3. Workflow Diagram

Accounts → services and scheduled tasks → audit events → PowerShell queries → Registry

## 4. Tools Used

| Tool | Purpose |
|---|---|
| Local Users and Groups / Local Security Policy | Review membership, password policy and auditing |
| Services / Task Scheduler | Inspect service identities and scheduled actions |
| Event Viewer / PowerShell | Filter records and inspect live state |
| Registry Editor | Inspect startup and service configuration |

## 5. Task-by-Task Walkthrough

### T1 — Users and groups

**Goal.** Inspect local account membership and policies.

1. Use Win+R to open the first console. Inspect Users, Groups and the properties of a selected user.
2. Open the second console and compare Account Policies → Password Policy with Account Lockout Policy.

```cmd
lusrmgr.msc
secpol.msc
```

**Command explained.** The first console manages local identities. The second shows rules that apply to them.

**Expected finding.** Membership helps explain privileges. A lockout threshold governs repeated failed attempts.

### T2 — Processes and services

**Goal.** Identify programs that can start automatically.

1. Open Services and inspect a service's startup type, identity and executable.
2. Open `shell:startup` from Run. In Task Scheduler, expand Task Scheduler Library → Microsoft → Windows and inspect triggers and actions.

```cmd
services.msc
taskschd.msc
```

**Command explained.** Services lists background components. Task Scheduler ties an action to a time or event.

**Expected finding.** A startup entry describes what can launch. The running state and task history provide separate evidence.

### T3 — Event logs

**Goal.** Locate authentication and account-management records.

1. In Local Security Policy inspect Local Policies → Audit Policy and Advanced Audit Policy Configuration.
2. Open Event Viewer. Under Windows Logs review Security, System and Application. Use Filter Current Log to narrow by ID.
3. If present, inspect Applications and Services Logs → Microsoft → Windows → Sysmon → Operational.

```cmd
eventvwr.msc
```

**Command explained.** The Security channel contains audited actions. IDs 4624 and 4625 identify successful and failed logons. ID 4634 marks logoff, 4720 account creation and 4732 addition to a security-enabled local group.

**Expected finding.** Use the timestamp, target user and subject fields to distinguish the account affected from the actor.

### T4 — PowerShell investigation

**Goal.** Query account, process, service, task and event records.

1. Replace angle-bracket values with names from the lab. Inspect local groups and members.
2. Compare a named service's state, account and executable path. Search scheduled tasks for names containing update.
3. Filter Security events to recent account creations.

```powershell
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember <group_name>
Get-Process
Get-Process -Name "<process-name>"
Get-Service
Get-Service -Name "<service_name>"
Get-CimInstance Win32_Service | Where-Object {$_.Name -eq "<service_name>"} | Select-Object Name, DisplayName, State, StartMode, StartName, PathName
Get-ScheduledTask
Get-ScheduledTask | Where-Object {$_.TaskName -like "*update*"}
Get-WinEvent -ListLog *
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4720} -MaxEvents 20 | Select-Object TimeCreated, Id, ProviderName, Message
```

**Command explained.** Where-Object filters each incoming record. Select-Object keeps specified fields. The hashtable restricts the event query at the source and MaxEvents bounds the output.

**Expected finding.** A matching 4720 record supplies the time and account details for an account-creation investigation.

### T5 — Registry basics

**Goal.** Inspect machine-wide startup configuration.

1. Open Registry Editor using Win+R and `regedit`. Inspect the machine-wide Run key.
2. In PowerShell read the Run values and inspect a known service key. Replace the service placeholder with its short name.

```powershell
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
Get-Item "HKLM:\SYSTEM\CurrentControlSet\Services\<ServiceName>"
```

**Command explained.** HKLM is the machine-wide hive. Get-ItemProperty reads values and Get-Item retrieves a key.

**Expected finding.** Compare the stored configuration with the service path and running process from the previous task.

## 6. Key Concepts (Beginner-Friendly)

An account identifies a principal. A group bundles permissions. An audit policy determines which activities produce evidence.

## 7. SOC / Real-World Relevance

Correlate an unexpected group membership with the Security record showing when it changed. Compare startup entries with service accounts and executable paths during triage.

## 8. Quick Command Cheat Sheet

| Command | What It Does |
|---|---|
| `lusrmgr.msc` | Inspect local account membership and policies. |
| `services.msc` | Identify programs that can start automatically. |
| `eventvwr.msc` | Locate authentication and account-management records. |
| `Get-LocalUser` | Query account, process, service, task and event records. |
| `Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"` | Inspect machine-wide startup configuration. |

## 9. Common Mistakes & Troubleshooting

- **Empty Security search.** Check the selected log, audit policy and event ID. A matching event must have occurred during retention.
- **Missing service key.** Use the service name returned by PowerShell. Its display name can differ.

## 10. Conclusion

Local consoles establish an inventory.
Event records supply a timeline.
PowerShell and Registry checks help corroborate each finding.
