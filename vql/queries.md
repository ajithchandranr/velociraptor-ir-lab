# VQL Query Reference

All VQL queries used across Phases 1, 2, and 3 of this lab.
Run these in the **Notebooks** tab on any enrolled client in the Velociraptor GUI.

---

## MFT - Windows.NTFS.MFT

**Files created in the last 7 days:**
```sql
SELECT FileName, FullPath, Created0x10, Created0x30
FROM source(artifact="Windows.NTFS.MFT")
WHERE Created0x10 > now() - 7 * 24 * 3600
ORDER BY Created0x10 DESC
```

**Timestomping indicator - SI and FN timestamps don't match:**
```sql
SELECT FileName, FullPath,
  Created0x10 AS SI_Created,
  Created0x30 AS FN_Created
FROM source(artifact="Windows.NTFS.MFT")
WHERE Created0x10 < Created0x30
AND NOT IsDir
LIMIT 100
```

**Locate a specific file by name:**
```sql
SELECT FileName, OSPath, Created0x10, Created0x30, FileSize, InUse
FROM source(artifact="Windows.NTFS.MFT")
WHERE FileName =~ "(?i)payload"
```

**Scripts in user-writable locations:**
```sql
SELECT FileName, OSPath, Created0x10, FileSize
FROM source(artifact="Windows.NTFS.MFT")
WHERE OSPath =~ "(?i)(appdata|temp|downloads|public)"
AND FileName =~ "(?i)(\.ps1|\.bat|\.vbs|\.hta|\.js)"
ORDER BY Created0x10 DESC
```

---

## Prefetch - Windows.Forensics.Prefetch

**All executions sorted by most recent:**
```sql
SELECT Executable, ExecutablePath,
  LastRunTimes, RunCount,
  CreationTime, ModificationTime
FROM source(artifact="Windows.Forensics.Prefetch")
ORDER BY LastRunTimes DESC
LIMIT 50
```

**Suspicious execution paths - user-writable locations:**
```sql
SELECT Executable, ExecutablePath, LastRunTimes, RunCount
FROM source(artifact="Windows.Forensics.Prefetch")
WHERE ExecutablePath =~ "(?i)(temp|appdata\\local\\temp|downloads|public|programdata)"
ORDER BY LastRunTimes DESC
```

**PowerShell execution history:**
```sql
SELECT Executable, ExecutablePath, LastRunTimes, RunCount
FROM source(artifact="Windows.Forensics.Prefetch")
WHERE Executable =~ "(?i)powershell"
ORDER BY LastRunTimes DESC
```

**Single execution entries - ran once and never again:**
```sql
SELECT Executable, ExecutablePath, LastRunTimes, RunCount
FROM source(artifact="Windows.Forensics.Prefetch")
WHERE RunCount = 1
ORDER BY LastRunTimes DESC
```

---

## Event Logs - Windows.EventLogs.Evtx

**Successful logons (Event ID 4624):**
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

**Process creation with command line (Event ID 4688):**
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

**New services installed (Event ID 7045):**
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

**Scheduled task created (Event ID 4698):**
```sql
SELECT timestamp(epoch=System.TimeCreated.SystemTime) AS EventTime,
  EventData.TaskName AS TaskName,
  EventData.SubjectUserName AS User
FROM source(artifact="Windows.EventLogs.Evtx")
WHERE System.EventID.Value = 4698
ORDER BY EventTime DESC
```

**PowerShell script block logging (Event ID 4104):**
```sql
SELECT timestamp(epoch=System.TimeCreated.SystemTime) AS EventTime,
  EventData.ScriptBlockText AS ScriptBlock,
  EventData.Path AS Path
FROM source(artifact="Windows.EventLogs.Evtx")
WHERE System.EventID.Value = 4104
ORDER BY EventTime DESC
LIMIT 20
```

**Sysmon process creation (Event ID 1):**
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

**Sysmon process creation - PowerShell only:**
```sql
SELECT timestamp(epoch=System.TimeCreated.SystemTime) AS EventTime,
  EventData.Image AS Image,
  EventData.CommandLine AS CommandLine,
  EventData.ParentImage AS ParentImage,
  EventData.IntegrityLevel AS IntegrityLevel
FROM source(artifact="Windows.EventLogs.Evtx")
WHERE System.Channel = "Microsoft-Windows-Sysmon/Operational"
AND System.EventID.Value = 1
AND EventData.Image =~ "(?i)powershell"
ORDER BY EventTime DESC
LIMIT 10
```

**Sysmon network connections (Event ID 3):**
```sql
SELECT timestamp(epoch=System.TimeCreated.SystemTime) AS EventTime,
  EventData.Image AS Image,
  EventData.DestinationIp AS DestIP,
  EventData.DestinationPort AS DestPort,
  EventData.SourceIp AS SourceIP
FROM source(artifact="Windows.EventLogs.Evtx")
WHERE System.Channel = "Microsoft-Windows-Sysmon/Operational"
AND System.EventID.Value = 3
AND EventData.Image =~ "(?i)powershell"
ORDER BY EventTime DESC
LIMIT 20
```

**Sysmon network connections - all processes:**
```sql
SELECT timestamp(epoch=System.TimeCreated.SystemTime) AS EventTime,
  EventData.Image AS Image,
  EventData.DestinationIp AS DestIP,
  EventData.DestinationPort AS DestPort,
  EventData.SourceIp AS SourceIP
FROM source(artifact="Windows.EventLogs.Evtx")
WHERE System.Channel = "Microsoft-Windows-Sysmon/Operational"
AND System.EventID.Value = 3
AND NOT EventData.DestinationIp =~ "^(10\.|172\.|192\.168\.|127\.)"
ORDER BY EventTime DESC
LIMIT 50
```

---

## Registry - Windows.Sysinternals.Autoruns

**All enabled persistence entries:**
```sql
SELECT timestamp(epoch=Time) AS Time,
  Location, Entry, Enabled, Category,
  Description, Signer, ImagePath, MD5, SHA256
FROM source(artifact="Windows.Sysinternals.Autoruns")
WHERE Enabled = "enabled"
ORDER BY Category
```

**Unsigned or third-party signed entries:**
```sql
SELECT timestamp(epoch=Time) AS Time,
  Location, Entry, Category,
  Signer, ImagePath, MD5, SHA256
FROM source(artifact="Windows.Sysinternals.Autoruns")
WHERE Enabled = "enabled"
AND NOT Signer =~ "Microsoft Windows"
AND NOT Signer =~ "Microsoft Corporation"
ORDER BY Category
```

**Scheduled tasks only:**
```sql
SELECT timestamp(epoch=Time) AS Time,
  `Entry Location` AS Location,
  Entry, Category, Enabled,
  Signer, `Image Path` AS ImagePath,
  `Launch String` AS LaunchString
FROM source(artifact="Windows.Sysinternals.Autoruns")
WHERE Category = "Scheduled Tasks"
ORDER BY Time DESC
```

**Scheduled task hunt by name pattern:**
```sql
SELECT timestamp(epoch=Time) AS Time,
  `Entry Location` AS Location,
  Entry, Category, Enabled,
  Signer, `Image Path` AS ImagePath,
  `Launch String` AS LaunchString
FROM source(artifact="Windows.Sysinternals.Autoruns")
WHERE Category = "Scheduled Tasks"
AND Entry =~ "(?i)edge"
```

**NTUser hive:**
```sql
SELECT OSPath, Data, Mtime, Username
FROM source(artifact="Windows.Registry.NTUser")
LIMIT 50
```

---

## Process List - Windows.System.Pslist

**All processes with parent and hash:**
```sql
SELECT CreateTime, Pid, Ppid, Name, CommandLine, Exe, Username
FROM source(artifact="Windows.System.Pslist")
LIMIT 50
```

**With SHA256 for threat intel:**
```sql
SELECT CreateTime, Pid, Ppid, Name, CommandLine, Hash.SHA256 AS SHA256
FROM source(artifact="Windows.System.Pslist")
LIMIT 50
```

**User context only - exclude system processes:**
```sql
SELECT Pid, Ppid, Name, CommandLine, Username, CreateTime
FROM source(artifact="Windows.System.Pslist")
WHERE Username !~ "NT AUTHORITY"
LIMIT 50
```

---

## VQL Operator Reference

| Operator | Meaning | Example |
|---|---|---|
| `=~` | Regex match | `Name =~ "(?i)powershell"` |
| `!~` | Regex NOT match | `Username !~ "NT AUTHORITY"` |
| `=` | Exact match | `System.EventID.Value = 4688` |
| `>` | Greater than | `Created0x10 > now() - 86400` |
| `<` | Less than | `Created0x10 < Created0x30` |
| `(?i)` | Case insensitive flag | `=~ "(?i)payload"` |
| `now()` | Current timestamp in seconds | `now() - 7 * 24 * 3600` |
| `LIMIT` | Row limit | `LIMIT 50` |
| `ORDER BY` | Sort results | `ORDER BY EventTime DESC` |
