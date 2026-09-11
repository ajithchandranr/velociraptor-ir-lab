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
                    IP: 192.168.100.3

Velociraptor Client: Windows host machine
```


## Prerequisites

- VirtualBox installed on your Windows machine
- Ubuntu Server 22.04 LTS ISO downloaded
- Internet connection on the host machine

---
