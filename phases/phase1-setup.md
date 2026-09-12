# Phase 1 - Server Deployment and Endpoint Enrollment

## Overview

This phase covers:
- Configuring networking so the server and client can communicate
- Installing Velociraptor on Ubuntu and generating the server config
- Running the server as a persistent service
- Deploying the Windows client
- Running the first triage collection and capturing a clean baseline

---

## Lab Topology

Host Machine (Windows)
|
└── VirtualBox
|
└── Ubuntu Server 22.04 VM (Velociraptor Server)
IP: 192.168.100.3

Velociraptor Client: Windows host machine


---

## Prerequisites

- VirtualBox installed on your Windows machine
- Ubuntu Server 22.04 LTS running as a VM
- Internet connection on the host machine

---

## Step 1 - Configure Networking

By default, VirtualBox assigns VMs a NAT IP (`10.0.2.15`) that isolates them from the host machine. We need to create a dedicated NAT Network called `labnet` that gives the Ubuntu VM a reachable IP, and forward the Velociraptor frontend port to the Windows host so the client can connect.

> **Note:** The GUI stays inside the Ubuntu VM and is accessed from the Ubuntu browser only. This mirrors a real IR deployment where the analyst GUI is never exposed directly to endpoints.

> **Note:** Shut down the Ubuntu VM before running these commands.

On your Windows host, open PowerShell as Administrator:

```powershell
$vbm = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"

# Create labnet
& $vbm natnetwork add --netname labnet --network 192.168.100.0/24 --enable --dhcp on

# Assign the Velociraptor VM to labnet
& $vbm modifyvm "Velociraptor" --nic1 natnetwork --nat-network1 labnet

# Forward only the frontend port - GUI stays inside the Ubuntu VM
& $vbm natnetwork modify --netname labnet --port-forward-4 "frontend:tcp:[]:8000:[192.168.100.3]:8000"

# Confirm
& $vbm natnetwork list
```

Expected output:

Name: labnet
Enabled: Yes
Network: 192.168.100.0/24
Gateway: 192.168.100.1
DHCP Server: Yes
Port-forwarding (ipv4)
frontend:tcp:[]:8000:[192.168.100.3]:8000


---

## Step 2 - Verify Server IP

Boot the Ubuntu VM and confirm the IP address:

```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

Expected output:

inet 192.168.100.3/24 brd 192.168.100.255 scope global dynamic noprefixroute enp0s3


![VM IP Confirmation](../screenshots/01-vm-ip-confirmation.png)

> **Note:** Your IP may differ slightly - it will be somewhere in the `192.168.100.0/24` range. Note it down, you will need it when generating the server config in Step 4.

---

## Step 3 - Install Velociraptor

Create the working directory:

```bash
sudo mkdir -p /opt/velociraptor
cd /opt/velociraptor
```

Update the package list and install wget:

```bash
sudo apt update && sudo apt install -y wget
```

Download the Velociraptor binary. Always check the latest release at:

https://github.com/Velocidex/velociraptor/releases


Then download:

```bash
wget https://github.com/Velocidex/velociraptor/releases/download/v0.77.2/velociraptor-v0.77.2-linux-amd64 -O velociraptor
```

Make it executable and move to PATH:

```bash
chmod +x velociraptor
sudo mv velociraptor /usr/local/bin/velociraptor
```

Verify the installation:

```bash
velociraptor version
```

Expected output:

name: velociraptor
version: 0.77.2
commit: c0c9dd609
build_time: "2026-08-10T00:58:52Z"
compiler: go1.25.3
system: linux
architecture: amd64

![Velociraptor Version](../screenshots/02-velociraptor-version.png)

> **Note:** Always match the binary version between server and client. Mismatched versions cause silent connection failures during client enrollment.
>
> ![Velociraptor Version](../screenshots/02-velociraptor-version.png)

---

## Step 4 - Generate the Server Config

Run the interactive config wizard:

```bash
sudo velociraptor config generate -i
```

Answer the prompts as follows:

| Prompt | Answer |
|---|---|
| OS | Linux |
| Datastore path | `/opt/velociraptor/datastore` |
| Frontend port | `8000` |
| GUI port | `8889` |
| SSL type | Self Signed SSL |
| DNS type | None - Configure DNS manually |
| Public DNS/IP | `192.168.100.3` (your VM IP from Step 2) |
| Username | your choice |
| Password | your choice |

> ![Velociraptor Version](../screenshots/03-config-wizard_1.png)

> **Note:** When the wizard asks for the public DNS/IP, enter the VM IP you confirmed in Step 2 - not `localhost` and not the old NAT IP `10.0.2.15`. Getting this wrong means clients will never be able to connect.

This generates two files in `/opt/velociraptor/`:

- `server.config.yaml` - server configuration
- `client.config.yaml` - client config to deploy to endpoints

Confirm both files exist:

```bash
ls -lh /opt/velociraptor/
```

Expected output:

```
-rw-r--r-- 1 root root  client.config.yaml
-rw-r--r-- 1 root root  server.config.yaml
drwxr-xr-x 2 root root  datastore
```
> ![Velociraptor Version](../screenshots/03-config-wizard_2.png)
>
> markdown
## Step 5 - Test the Server

Run the server in test mode first to confirm it starts cleanly:

```bash
sudo velociraptor --config /opt/velociraptor/server.config.yaml frontend -v
```

Look for these two lines before proceeding:

>GUI is ready to handle TLS requests on https://127.0.0.1:8889/
>Frontend is ready to handle client TLS requests at https://192.168.100.3:8000/


> **Note:** The GUI is bound to `127.0.0.1` only - this is intentional. It means the GUI is only accessible from inside the Ubuntu VM browser, not from the Windows host. This mirrors a real IR deployment where the analyst interface is never exposed directly to endpoints.

![Server Test Run](../screenshots/04-server-test-run.png)
Once confirmed, press `Ctrl+C` to stop the server and proceed to Step 6.

## Step 6 - Run Velociraptor as a systemd Service

The test run confirmed the server starts cleanly. Now set it up as a persistent service so it survives reboots and runs automatically.

Press `Ctrl+C` to stop the test run, then create the service file:

```bash
sudo nano /etc/systemd/system/velociraptor.service
```

Paste exactly this:

```ini
[Unit]
Description=Velociraptor IR Server
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/velociraptor \
  --config /opt/velociraptor/server.config.yaml \
  frontend -v
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Save with `Ctrl+O`, exit with `Ctrl+X`.

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable velociraptor
sudo systemctl start velociraptor
sudo systemctl status velociraptor
```

Expected output:

```
● velociraptor.service - Velociraptor IR Server
     Active: active (running)
```
![Service Running](../screenshots/05-service-running.png)

## Step 7 - Access the GUI

Open a browser inside the Ubuntu VM and navigate to:

https://127.0.0.1:8889


Log in with the credentials you set during the config wizard.

![GUI Login](../screenshots/06-gui-login.png)

---

---

## A Note on Real IR Deployments

In this lab, the Velociraptor server runs inside a VirtualBox VM accessible only on the local network. In a real IR engagement the setup is different but the workflow is identical.

<pre>
Internet
    |
    ├── Endpoint (any location)
    ├── Endpoint (any location)
    └── Endpoint (any location)
            |
            └── All connect to ──► Velociraptor Server
                                   (Cloud VM with public IP)
                                   Analyst accesses GUI
                                   over VPN or SSH tunnel
</pre>

The server runs on a cloud instance (AWS, Azure, GCP) with a real public IP. The client config has that public IP baked in. When the client is installed on any endpoint anywhere in the world it phones home to the server over port 8000.

For remote helpline scenarios - where an individual contacts you suspecting compromise - the analyst sends a pre-packaged collector: a single executable with the client config already embedded. The person runs it with one double-click. Their device appears in the GUI within seconds and live triage begins immediately, regardless of where in the world they are.

What we build in this lab mirrors that workflow exactly. VirtualBox NAT stands in for the internet. Everything else - the artifacts, the queries, the IR process - is the same.

---

## Step 8 - Deploy the Windows Client

The client needs two things on the Windows machine:
- The Velociraptor Windows executable
- The `client.config.yaml` generated in Step 4

### Transfer the client config to Windows

On the Ubuntu VM, serve the file temporarily:

```bash
cd /opt/velociraptor
sudo python3 -m http.server 9999
```

Add a temporary port forward on the Windows host so it can reach the file server. Open PowerShell as Administrator:

```powershell
$vbm = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"
& $vbm natnetwork modify --netname labnet --port-forward-4 "fileserver:tcp:[]:9999:[192.168.100.3]:9999"
```

Download the config:

```powershell
Invoke-WebRequest `
  -Uri "http://127.0.0.1:9999/client.config.yaml" `
  -OutFile "C:\Users\$env:USERNAME\client.config.yaml"
```

Confirm it downloaded:

```powershell
Get-Item "C:\Users\$env:USERNAME\client.config.yaml"
```

Expected output:
-a---- 12-09-2026 2691 client.config.yaml


Stop the Python server on Ubuntu with `Ctrl+C`.

### Update the server URL in the config

The config currently points to `192.168.100.3:8000` - your Windows machine cannot reach that IP directly through the NAT network. Update it to your Windows host's LAN IP so the port forward routes it correctly.

Find your Windows LAN IP:

```powershell
ipconfig | findstr "IPv4"
```

Use the first IP shown - ignore any virtual adapter IPs from VMware or VirtualBox.

Open the config and update the server URL:

```powershell
notepad "C:\Users\$env:USERNAME\client.config.yaml"
```

Find:

```yaml
Client:
  server_urls:
  - https://192.168.100.3:8000/
```

Change to:

```yaml
Client:
  server_urls:
  - https://<your-windows-lan-ip>:8000/
```

Save and close.

### Download the Windows executable

```powershell
Invoke-WebRequest `
  -Uri "https://github.com/Velocidex/velociraptor/releases/download/v0.77.2/velociraptor-v0.77.2-windows-amd64.exe" `
  -OutFile "C:\Users\$env:USERNAME\velociraptor.exe"
```

### Install as a Windows service

Run PowerShell as Administrator:

```powershell
# Create the required directory
New-Item -ItemType Directory -Path "C:\Program Files\Velociraptor" -Force

cd C:\Users\$env:USERNAME

# Install as service
.\velociraptor.exe --config client.config.yaml service install

# Verify
Get-Service -Name Velociraptor
```

Expected output:

Status Name DisplayName

Running Velociraptor Velociraptor


> **Why install as a service and not run manually?** Running manually gives the client only your user's privileges. You will get `Access is denied` errors on protected system processes like `csrss.exe` and `wininit.exe`. Installed as a Windows service it runs as SYSTEM - full visibility across all processes, no access errors, and it survives reboots automatically.

Within 30 seconds of the service starting, check the Velociraptor GUI in the Ubuntu VM browser. Your Windows machine should appear in the **Clients** tab.
