# velociraptor-ir-lab

A hands-on Velociraptor DFIR lab built from scratch on VirtualBox.

Each phase builds a real investigative capability - not just how 
to run the tool, but why each artifact matters during an active 
IR engagement. Scenarios and techniques come directly from real 
MSSP SOC work, not textbook examples.

![GUI Login](/screenshots/06-gui-login.png)
---

## Phases

| Phase | Topic | Status |
|---|---|---|
| 1 | Server deployment, endpoint enrollment, baseline triage collection | Complete |
| 2 | MFT collection and timeline building | In progress |
| 3 | Simulated compromise and detection | Coming |
| 4 | Live response workflow | Coming |
| 5 | Threat hunting notebook | Coming |

---

## Lab Topology

```
Host Machine (Windows)
    |
    └── VirtualBox
            |
            └── Ubuntu 22.04 VM
                    IP: 192.168.100.3 (NAT Network)
                    Role: Velociraptor Server
                    GUI: https://127.0.0.1:8889 (VM browser only)

Velociraptor Client: Windows host machine
Connects via: 127.0.0.1:8000 (port forward)
```



## Stack

- Velociraptor 0.77.2
- Ubuntu Server 22.04 LTS
- VirtualBox 7.x
- Windows 11 Pro (client)

---

## Why Velociraptor

In MSSP IR work, you regularly pivot to endpoints that are not 
integrated into the SIEM. Velociraptor fills that gap - deploy 
the agent, enroll the endpoint, and within minutes you have full 
forensic visibility without waiting on change management or log 
onboarding.

---

## Contents

- `phase1/` - Server deployment and endpoint enrollment
- `phase2/` - Forensic artifact collection (coming)
- `phase3/` - Custom VQL artifact development (coming)
- `phase4/` - Live response workflow (coming)
- `phase5/` - Threat hunting notebook (coming)
