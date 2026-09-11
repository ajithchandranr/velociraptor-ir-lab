# Phase 1 - Server Deployment and Endpoint Enrollment

## Overview

This phase covers:
- Setting up the Ubuntu Server VM in VirtualBox
- Configuring networking so the server and client can communicate
- Installing Velociraptor and generating the server config
- Deploying the Windows client
- Running the first triage collection and capturing a clean baseline

## Lab Topology

```
Host Machine (Windows)
    |
    └── VirtualBox
            |
            └── Ubuntu Server 22.04 VM (Velociraptor Server)

Velociraptor Client: Windows host machine
```


## Prerequisites

- VirtualBox installed on your Windows machine
- Ubuntu Server 22.04 LTS ISO downloaded
- Internet connection on the host machine

---

## Step 1 - Configure Networking

By default, VirtualBox assigns VMs a NAT IP (`10.0.2.15`) that isolates them from the host machine and from each other. We need to create a dedicated NAT Network called `labnet` that allows communication between the Ubuntu VM and the Windows host via port forwarding.

> **Note:** Shut down the Ubuntu VM before running these commands.

On your Windows host, open PowerShell as Administrator:

```powershell
$vbm = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"

# Create labnet
& $vbm natnetwork add --netname labnet --network 192.168.100.0/24 --enable --dhcp on

# Assign the Velociraptor VM to labnet
& $vbm modifyvm "Velociraptor" --nic1 natnetwork --nat-network1 labnet
```

Boot the Ubuntu VM back up, then confirm the new IP:

```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

Expected output: inet 192.168.100.3/24 brd 192.168.100.255 scope global dynamic noprefixroute enp0s3


> **Note:** Your IP may differ slightly - it will be somewhere in the `192.168.100.0/24` range. Note it down, you will need it in the next step.
