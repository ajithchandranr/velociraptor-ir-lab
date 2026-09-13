# Phase 2 - Forensic Artifact Collection

## Overview

This phase covers forensic artifact collection across five 
categories - the evidence sources that tell the full story of 
what happened on an endpoint during an incident.

```
1. MFT      - filesystem record, every file ever on the volume
2. Prefetch - execution history, what ran and when
3. EVTX     - Windows event logs
4. Registry - persistence, user activity, system configuration
5. Memory   - running state, injected code, decrypted strings
```

Phase 2 establishes the collection capability before any 
simulated compromise. You need to know how to pull these 
artifacts cleanly before you have something interesting to 
look at.

---

## Scenario

### Scenario A - Phishing to Persistence to Lateral Movement

Full kill chain simulated across Phase 2 and Phase 3:

```
1. Phishing email arrives
2. User opens malicious Office document
3. Macro drops payload to disk
4. Payload establishes persistence via scheduled task
5. Beacons to C2
6. Lateral movement attempt via PsExec/WMI
```

### Scenario B - Helpline/Targeted Surveillance

Covered in a later phase - triage collection with full audit 
trail, consent documentation, and stalkerware/spyware indicators.

---

## Prerequisites

```
- Phase 1 complete - server running, physical machine enrolled
- FlareVM added to labnet (192.168.100.4)
- Velociraptor client deployed on FlareVM and running as SYSTEM
- FlareVM-Clean-Baseline snapshot taken
```

---

## Lab Topology

```
Host Machine (Windows) - 192.168.1.x
    |
    └── VirtualBox - labnet (192.168.100.0/24)
            |
            ├── Ubuntu 22.04 VM - Velociraptor Server
            │       IP: 192.168.100.3
            │       GUI: https://127.0.0.1:8889 (VM browser only)
            │
            └── ACR-FlareVM - Victim Machine
                    IP: 192.168.100.4
                    Role: Simulated compromised endpoint
                    Snapshot: FlareVM-Clean-Baseline
```

---

## FlareVM Setup

FlareVM was added to labnet and assigned `192.168.100.4` via 
DHCP. The Velociraptor client was deployed identically to the 
physical machine client in Phase 1.

```powershell
# Add FlareVM to labnet (run on Windows host, VM must be off)
$vbm = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"
& $vbm modifyvm "ACR-FlareVM" --nic1 natnetwork --nat-network1 labnet
```


Once booted, the Velociraptor client was transferred from the 
Ubuntu VM via Python HTTP server, installed as a Windows service, 
and the VM appeared in the GUI within 30 seconds.

Baseline collection was run on FlareVM matching the Phase 1 
baseline artifact set. Clean snapshot taken before any 
simulated activity.

---

## Artifact 1 - MFT Collection

### What the MFT Is

Every file on an NTFS volume has a record in the Master File 
Table. Think of it as the filesystem's database - one record 
per file containing:

```
- Full file path
- File size
- Four timestamps from $STANDARD_INFORMATION:
    Created, Modified, Accessed, Changed
- Four timestamps from $FILE_NAME:
    Created, Modified, Accessed, Changed
- MFT record number
- Parent directory reference
- Whether the file is active or deleted
```

The MFT covers every file that ever existed on the volume - 
including deleted ones, until the record is reused.

> **Further reading:** [The MFT - The One Artifact I Check First on Every Windows Investigation](https://mohitdhabuwala17.medium.com/the-mft-the-one-artifact-i-check-first-on-every-windows-investigation-bc1fc32a730e)

---

### Why Two Timestamp Sets Matter

The `$STANDARD_INFORMATION` timestamps are easy to manipulate. 
Tools like Timestomp can set them to any arbitrary value with 
no special privileges. Attackers use this to make a malicious 
file appear to have existed on the system long before it was 
actually dropped.

The `$FILE_NAME` timestamps require kernel-level access to 
modify and are therefore much harder to fake.

When the two sets don't match - that discrepancy is a 
timestomping indicator:

```
$STANDARD_INFORMATION Created:  2019-01-01 00:00:00  ← manipulated
$FILE_NAME Created:             2026-09-13 06:22:11  ← real
```

Phase 3 builds a VQL hunt to detect this discrepancy at scale.

---

### Understanding MFT Record Fields

A single MFT record looks like this:

```
EntryNumber:  0          ← MFT record number, 0 = the MFT itself
InUse:        true       ← file exists, not deleted
Links:        5          ← number of directory references
FullPath:     \\.\C:\$MFT
FileName:     $MFT
FileSize:     295698432  ← size in bytes
IsDir:        false      ← not a directory
```

Timestamp fields:

```
Created0x10   ← $STANDARD_INFORMATION Created  (easy to manipulate)
Modified0x10  ← $STANDARD_INFORMATION Modified
Accessed0x10  ← $STANDARD_INFORMATION Accessed
Changed0x10   ← $STANDARD_INFORMATION Changed

Created0x30   ← $FILE_NAME Created  (harder to fake)
Modified0x30  ← $FILE_NAME Modified
Accessed0x30  ← $FILE_NAME Accessed
Changed0x30   ← $FILE_NAME Changed
```

Additional flags:

```
IsReparsePoint  ← symlink or junction point
IsEncrypted     ← EFS encrypted
IsCompressed    ← NTFS compressed
IsSparse        ← sparse file
NameType        ← DOS+Win32 means file has both long and short name
```

---

### Running the Collection

```
FlareVM client > New Collection > Windows.NTFS.MFT > Launch
```

Default parameters collect the entire C: volume. On a 30GB 
used volume expect 300-500MB of output and a collection time 
of 3-5 minutes.

![MFT Collection](../screenshots/phase2-01-mft-collection.png)

---

### VQL - What to Look at First

Run these in the Notebooks tab after collection completes:

```sql
-- All files created in the last 7 days
SELECT FileName, FullPath, Created0x10, Created0x30
FROM source(artifact="Windows.NTFS.MFT")
WHERE Created0x10 > now() - 7 * 24 * 3600
ORDER BY Created0x10 DESC
```

```sql
-- Timestomping indicator - SI and FN timestamps don't match
SELECT FileName, FullPath, 
  Created0x10 AS SI_Created,
  Created0x30 AS FN_Created
FROM source(artifact="Windows.NTFS.MFT")
WHERE Created0x10 < Created0x30
AND NOT IsDir
LIMIT 100
```

> **Note:** `Created0x10` is `$STANDARD_INFORMATION`.
> `Created0x30` is `$FILE_NAME`. When SI is earlier than FN
> on a recently dropped file, suspect timestomping.

---

### Baseline Analysis - What Normal Looks Like

Running the timestomping detection query on a clean FlareVM 
before any simulated compromise produces results like these:

![MFT Collection](../screenshots/phase2-01-mft-collection_1.png)

### The Rule of Thumb

```
Round number timestamp + unexpected file location = investigate
2001-01-01 00:00:00 on a file in AppData or Temp  = immediate flag
2001-01-01 00:00:00 on a Windows system cab        = probably benign
```

This baseline is the reference point. When the simulated 
compromise runs in Phase 3 and this query is re-run, any new 
entries that were not present here are the malicious files.
The delta is the signal.

---

## Artifact 2 - Prefetch

### What Prefetch Is

Windows Prefetch records execution history. Every time a binary 
runs, Windows creates or updates a `.pf` file in 
`C:\Windows\Prefetch`.

Prefetch survives process termination and reboot. An attacker 
can delete their payload from disk - Prefetch still shows it ran.

Each prefetch file contains:

```
- Executable name
- Full path to the executable
- First run time
- Last run time - stored as an array, last 8 run times recorded
- Run count - total number of executions
- Files and directories accessed during execution
- Hash of the executable path
```

> **Note:** Prefetch is enabled by default on workstations.
> Disabled by default on Windows Server editions.

> **Further reading:** [Windows Forensics - Prefetch](https://medium.com/@omaymaW/windows-forensics-prefetch-8447dbb6cd9b)

---

### Running the Collection

> **Note:** The artifact name changed in v0.77.2. Use
> `Windows.Forensics.Prefetch` - not `Windows.Analysis.Prefetch`.

```
FlareVM client > New Collection > Windows.Forensics.Prefetch > Launch
```

Default parameters are fine. Collection returns 282 rows on 
a clean FlareVM.

![Prefetch Collection](../screenshots/phase2-02-prefetch-collection.png)

---

### Understanding Prefetch Record Fields

```
Executable:       WMIPRVSE.EXE
ExecutablePath:   \DEVICE\HARDDISKVOLUME3\WINDOWS\SYSTEM32\WBEM\WMIPRVSE.EXE
LastRunTimes:     ["2026-09-13T08:16:01Z", "2026-09-13T08:09:18Z", ...]
RunCount:         42
CreationTime:     2025-12-27  ← when the prefetch file was first created
ModificationTime: 2026-09-13  ← when it was last updated
Hash:             0XE8B8DD29  ← hash of the executable path
```

`LastRunTimes` is an array - Windows stores the last 8 execution 
times per binary. During IR this lets you reconstruct a timeline 
of when a tool was used, not just that it was used.

![Prefetch Fields](../screenshots/phase2-02-prefetch-collection_1.png)

---

### VQL - What to Look at First

```sql
-- All executions sorted by most recent
SELECT Executable, ExecutablePath, 
  LastRunTimes, RunCount, 
  CreationTime, ModificationTime
FROM source(artifact="Windows.Forensics.Prefetch")
ORDER BY LastRunTimes DESC
LIMIT 50
```

```sql
-- Suspicious execution paths - temp and user-writable locations
SELECT Executable, ExecutablePath, LastRunTimes, RunCount
FROM source(artifact="Windows.Forensics.Prefetch")
WHERE ExecutablePath =~ "(?i)(temp|appdata\\local\\temp|downloads|public|programdata)"
ORDER BY LastRunTimes DESC
```

---

### Baseline Analysis - What Normal Looks Like

![Prefetch Baseline](../screenshots/phase2-02-prefetch-collection_2.png)

Two representative entries from a clean FlareVM:

| Executable | RunCount | Notes |
|---|---|---|
| WMIPRVSE.EXE | 42 | WMI Provider Host - spawns constantly, high run count is normal |
| VSSVC.EXE | 6 | Volume Shadow Copy - ran today due to VirtualBox snapshot activity |

Suspicious path query returned no results on clean FlareVM:

```
Baseline: no executions from temp or user-writable locations.
Any hit post-compromise is immediately suspicious.
```

---

### What to Look for During the Simulated Compromise

```
New entry in AppData\Local\Temp    = payload executed from temp
RunCount of 1                      = ran once, possibly deleted after
CreationTime = compromise window   = first execution aligns with attack
Single entry in LastRunTimes       = ran once and never again
```

When the phishing payload executes in Phase 3, this artifact 
will show the executable name, full path, and exact time it 
ran - even if the file is deleted from disk before collection.

---

## Artifact 3 - Windows Event Logs (EVTX)

### What EVTX Covers

Windows Event Logs are the primary audit trail for security 
events on a Windows endpoint. By default, critical security 
events are not fully logged - audit policy must be explicitly 
configured before meaningful IR data is available.

> **Helpline note:** In a helpline engagement, audit policy 
> will rarely be pre-configured on the target device. Collection 
> relies on whatever Windows has logged by default. Detection 
> coverage will be narrower - native logs only, no Sysmon.

---

### Audit Policy Configuration

Before any simulated compromise activity, configure audit policy 
on FlareVM to ensure full event coverage across the kill chain.

```powershell
# Process creation with full command line
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f

# Logon and logoff
auditpol /set /subcategory:"Logon" /success:enable /failure:enable

# Scheduled task auditing
auditpol /set /subcategory:"Other Object Access Events" /success:enable /failure:enable

# Account management
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable

# File system access
auditpol /set /subcategory:"File System" /success:enable /failure:enable

# Privilege use
auditpol /set /subcategory:"Sensitive Privilege Use" /success:enable /failure:enable

# PowerShell script block logging
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" /v EnableScriptBlockLogging /t REG_DWORD /d 1 /f
```

> **Note:** Take a snapshot after configuring audit policy and 
> before installing Sysmon:
>
> ```
> Name: FlareVM-Audit-Configured-NoSysmon
> ```
>
> This is the Scenario B starting point - mirrors a helpline 
> engagement where Sysmon is not present on the device.

---

### Sysmon Installation

Sysmon provides significantly richer telemetry than native 
Windows auditing alone. Installed using the SwiftOnSecurity 
community baseline config - the industry standard starting point.

```powershell
# Download Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
  -OutFile "C:\Windows\Temp\Sysmon.zip"

# Extract
Expand-Archive -Path "C:\Windows\Temp\Sysmon.zip" `
  -DestinationPath "C:\Windows\Temp\Sysmon"

# Download SwiftOnSecurity config
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
  -OutFile "C:\Windows\Temp\sysmonconfig.xml"

# Install
cd C:\Windows\Temp\Sysmon
.\Sysmon64.exe -accepteula -i ..\sysmonconfig.xml
```

**Why Sysmon over native auditing:**

| Field | Native 4688 | Sysmon Event ID 1 |
|---|---|---|
| Process GUID | No | Yes - tracks process across events |
| Parent command line | No | Yes |
| File hashes | No | MD5 + SHA256 + IMPHASH |
| Integrity level | No | Yes |
| IMPHASH | No | Yes - import hash for malware family grouping |

`IMPHASH` is a hash of a binary's import table rather than its 
file content. Two malware samples compiled differently but 
calling the same Windows API functions share the same IMPHASH - 
useful for clustering related malware families even when file 
hashes differ.

> **Note:** Take a snapshot after Sysmon installation:
>
> ```
> Name: FlareVM-Sysmon-Installed-Baseline
> ```
>
> This is the Scenario A starting point for the MSSP simulation.

---

### Running the Collection

```
FlareVM client > New Collection > Windows.EventLogs.Evtx > Launch
```

Default parameters collect all event log channels.

![EVTX Collection](../screenshots/phase2-03-evtx-collection.png)

---

### Key Event IDs for the Kill Chain

| Event ID | Channel | What it captures |
|---|---|---|
| 4624 | Security | Successful logon |
| 4625 | Security | Failed logon |
| 4688 | Security | Process creation with command line |
| 4698 | Security | Scheduled task created |
| 4702 | Security | Scheduled task modified |
| 4720 | Security | User account created |
| 4732 | Security | Account added to group |
| 4663 | Security | File accessed |
| 7045 | System | New service installed |
| 4103 | PowerShell | Module logging |
| 4104 | PowerShell | Script block logging |
| 1 | Sysmon | Process creation |
| 3 | Sysmon | Network connection |
| 11 | Sysmon | File created |
| 13 | Sysmon | Registry value set |

---

### VQL - What to Look at First

**Successful logons:**

```sql
SELECT timestamp(epoch=System.TimeCreated.SystemTime) AS EventTime,
  System.Computer AS Computer,
  EventData.SubjectUserName AS Subject,
  EventData.TargetUserName AS LogonUser,
  EventData.TargetDomainName AS Domain,
  EventData.LogonType AS LogonType,
  EventData.IpAddress AS SourceIP,
  EventData.ProcessName AS Process
FROM source(artifact="Windows.EventLogs.Evtx")
WHERE System.EventID.Value = 4624
ORDER BY EventTime DESC
LIMIT 50
```

![Logon Events](../screenshots/phase2-03-evtx-collection_logon.png)

**Process creation with command line:**

```sql
SELECT timestamp(epoch=System.TimeCreated.SystemTime) AS EventTime,
  System.Computer AS Computer,
  EventData.SubjectUserName AS User,
  EventData.NewProcessName AS Process,
  EventData.CommandLine AS CommandLine,
  EventData.ParentProcessName AS ParentProcess
FROM source(artifact="Windows.EventLogs.Evtx")
WHERE System.EventID.Value = 4688
ORDER BY EventTime DESC
LIMIT 50
```

![Process Creation](../screenshots/phase2-03-evtx-collection_process_creation.png)

**New services installed:**

```sql
SELECT timestamp(epoch=System.TimeCreated.SystemTime) AS EventTime,
  System.Computer AS Computer,
  EventData.ServiceName AS ServiceName,
  EventData.ImagePath AS ImagePath,
  EventData.ServiceType AS ServiceType,
  EventData.StartType AS StartType,
  EventData.AccountName AS AccountName
FROM source(artifact="Windows.EventLogs.Evtx")
WHERE System.EventID.Value = 7045
ORDER BY EventTime DESC
LIMIT 20
```

![New Services](../screenshots/phase2-03-evtx-collection_new_services_installed.png)

**Sysmon process creation - full telemetry:**

```sql
SELECT timestamp(epoch=System.TimeCreated.SystemTime) AS EventTime,
  System.Computer AS Computer,
  EventData.Image AS Image,
  EventData.CommandLine AS CommandLine,
  EventData.User AS User,
  EventData.IntegrityLevel AS IntegrityLevel,
  EventData.Hashes AS Hashes,
  EventData.ParentImage AS ParentImage,
  EventData.ParentCommandLine AS ParentCommandLine,
  EventData.ProcessGuid AS ProcessGuid,
  EventData.ParentProcessGuid AS ParentProcessGuid
FROM source(artifact="Windows.EventLogs.Evtx")
WHERE System.Channel = "Microsoft-Windows-Sysmon/Operational"
AND System.EventID.Value = 1
ORDER BY EventTime DESC
LIMIT 50
```

![Sysmon Events](../screenshots/phase2-03-evtx-collection_sysmon.png)

---

### Baseline Analysis - What Normal Looks Like

Confirmed present on a clean FlareVM after audit policy 
configuration and Sysmon installation:

```
4624  - Service logon events (Logon Type 5) from services.exe
4688  - Process creation events with full command line
7045  - Velociraptor service install captured
Sysmon Event ID 1 - Process creation with full telemetry
```

**Logon type reference:**

```
Type 2   = Interactive       - user at keyboard
Type 3   = Network           - remote access, file share
Type 4   = Batch             - scheduled task execution
Type 5   = Service           - service account (normal on clean machine)
Type 7   = Unlock            - screen unlock
Type 10  = RemoteInteractive

```
---


## Artifact 4 - Registry Hives

### What the Registry Covers for IR

The registry stores configuration, user activity, and 
persistence mechanisms. Key locations:

| Key | What it stores |
|---|---|
| `HKLM\...\CurrentVersion\Run` | System-wide autorun entries |
| `HKCU\...\CurrentVersion\Run` | Per-user autorun entries |
| `HKLM\SYSTEM\...\Services` | Installed services |
| `HKCU\...\Explorer\RecentDocs` | Recently opened files |
| `HKCU\...\Explorer\RunMRU` | Commands run via Run dialog |

### Running the Collection

```
FlareVM client > New Collection > Windows.Registry.NTUser > Launch
```

For persistence-focused collection:

```
FlareVM client > New Collection > Windows.Registry.Sysinternals.Autoruns > Launch
```

![Registry Collection](../screenshots/phase2-04-registry-collection.png)

### VQL - What to Look at First

```sql
-- All autorun entries
SELECT Hive, KeyPath, Name, Data
FROM source(artifact="Windows.Registry.NTUser")
WHERE KeyPath =~ "CurrentVersion\\\\Run"
```

---

## Artifact 5 - Memory Acquisition

### What Memory Collection Gives You

A memory image captures the running state of the system at 
the moment of collection:

```
- Every running process including injected code
- Decrypted strings and configuration from in-memory malware
- Network connections at time of capture
- Loaded DLLs including reflectively loaded ones
- Attacker tooling that never touches disk
```

### Running the Collection

```
FlareVM client > New Collection > Windows.Memory.Acquisition > Launch
```

![Memory Collection](../screenshots/phase2-05-memory-collection.png)

> **Note:** Memory acquisition produces output equal to the VM's 
> RAM allocation. A 4GB RAM VM produces a ~4GB image. Ensure the 
> server has sufficient disk space before running.

> **Note:** Run memory acquisition last. It captures point-in-time 
> state - collect all other artifacts first so memory reflects the 
> most complete picture of system activity.

---

## Collection Summary

| Artifact | Velociraptor Artifact Name | Key IR Value |
|---|---|---|
| MFT | `Windows.NTFS.MFT` | File timeline, timestomping detection |
| Prefetch | `Windows.Analysis.Prefetch` | Execution history including deleted files |
| Event Logs | `Windows.EventLogs.Evtx` | Authentication, process creation, persistence |
| Registry | `Windows.Registry.NTUser` | Autorun entries, user activity |
| Memory | `Windows.Memory.Acquisition` | In-memory malware, injected code |

---

## Take a Snapshot

After all five collections complete on the clean FlareVM:

```
VirtualBox > ACR-FlareVM > right-click > Take Snapshot
Name: FlareVM-Phase2-Collections-Complete
```

Phase 3 starts from here - simulated compromise, then 
re-collection and analysis against this baseline.
