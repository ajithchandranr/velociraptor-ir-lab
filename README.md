# Velociraptor IR Lab

A hands-on Velociraptor DFIR lab built from scratch - covering server deployment, endpoint enrollment, forensic artifact collection, and detection engineering across multiple phases.

The scenarios and techniques here come directly from real IR work, not textbook examples.

![GUI Login](/screenshots/06-gui-login.png)

---

## Why This Exists

In incident response, the gap that hurts most isn't a lack of tools - it's a lack of visibility at the wrong moment.

Two scenarios drove this project:

**The MSSP gap** - You're mid-investigation and the machine isn't in the SIEM. No logs. No telemetry. Change management kicks in and you're writing interim reports with a gap in the middle where that endpoint should be.

**The helpline scenario** - A journalist or activist contacts a digital security helpline suspecting their device is compromised. No prior agent deployment. No organisational relationship with the endpoint. You need visibility fast, with full transparency to the person whose device you're examining.

Velociraptor closes both gaps. This lab documents how.

---

## Lab Environment

```
Host Machine (Windows)
    |
    └── VirtualBox - labnet (192.168.100.0/24)
            |
            ├── Ubuntu Server 22.04 (Velociraptor Server)
            │       IP: 192.168.100.3
            │
            └── ACR-FlareVM (Victim Machine)
                    IP: 192.168.100.4
```

- Velociraptor version: v0.77.2
- Server OS: Ubuntu Server 22.04 LTS
- Victim VM: FlareVM (Windows 11)

---

## Phases

| Phase | Status | Description |
|---|---|---|
| [Phase 1 - Server Deployment and Endpoint Enrollment](phases/phase1-setup.md) | Complete | Server setup, networking, client deployment, baseline collection |
| [Phase 2 - Forensic Artifact Collection](phases/phase2-forensic-collection.md) | Complete | MFT, Prefetch, EVTX, Registry, Memory acquisition |
| [Phase 3 - Simulated Compromise and Detection](phases/phase3-compromise.md) | Complete | LNK dropper, C2 beacon, persistence, artifact-based detection |

---

## VQL Reference

All VQL queries used across phases, organised by artifact:

[vql/queries.md](vql/queries.md)

---

## Tools Used

| Tool | Purpose |
|---|---|
| Velociraptor v0.77.2 | Endpoint visibility and artifact collection |
| Volatility 3 v2.28.0 | Memory forensics |
| Sysmon + SwiftOnSecurity config | Enhanced Windows telemetry |
| VirtualBox | Lab virtualisation |
| Python HTTP server | Simulated C2 listener |

---
