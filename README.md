# Wazuh SIEM Home Lab — Detecting an RDP Brute-Force Attack

A hands-on SOC analyst home lab built from scratch on Apple Silicon using UTM for virtualization. Deploys a working SIEM, connects a monitored Windows endpoint, launches a real attack from a separate Kali Linux attacker VM, and verifies end-to-end detection on the Wazuh dashboard.

---

## Why I built this

I'm transitioning into a cybersecurity/SOC analyst role and wanted hands-on proof I can deploy, configure, and operate a real SIEM stack — not just talk about the theory.

---

## Architecture

Three virtual machines on a single Mac (Apple Silicon, UTM), all on the same private network:

```text
┌─────────────────┐        ┌──────────────────────┐
│   Ubuntu Server │◄──────►│   Windows 11 (ARM)   │
│  Wazuh Manager +│  logs  │   Wazuh Agent        │
│  Indexer + Dash │        │ (monitored endpoint) │
│  192.168.64.2   │        │   192.168.64.4       │
└────────┬────────┘        └────────────▲─────────┘
         │                                │
         │ dashboard (browser)      RDP brute-force
         │                                │
         │                       ┌────────┴──────────┐
         └──────────────────────►│   Kali Linux      │
                                 │   (attacker VM)   │
                                 │   192.168.64.5    │
                                 └───────────────────┘
```

## Tools & Tech Stack

*   **Wazuh 4.14** — SIEM manager, indexer, and dashboard
*   **Ubuntu Server** — Wazuh server host
*   **Windows 11 (ARM)** — monitored endpoint running the Wazuh agent
*   **Kali Linux (ARM64)** — attacker machine
*   **UTM** — virtualization on Apple Silicon (QEMU-based)
*   **Hydra** — password brute-force tool
*   **Nmap** — network reconnaissance

---

## Step-by-Step Implementation Guide

### Phase 1: Environment Provisioning (UTM on Apple Silicon)
*   Download UTM for macOS (`getutm.app`) to serve as the hypervisor.
*   Ensure all virtual machines are configured to use **Shared Network** mode rather than isolated NAT or VLANs so they can communicate across a common subnet.

### Phase 2: Deploying the Wazuh Server (Ubuntu VM)
*   Download the Ubuntu Server ARM64 ISO from the official Ubuntu website.
*   In UTM, create a new Linux virtual machine, attach the ISO, assign 4GB of RAM, and create a 40GB virtual disk (`wazuh-server-ubuntu`).
*   Complete the Ubuntu Server installation wizard (install OpenSSH server during setup for remote terminal access).
*   Note the internal IP address (e.g., `192.168.64.2`) using `ip a`.
*   Install the all-in-one Wazuh server package via the Ubuntu terminal:

```bash
curl -sO [https://packages.wazuh.com/4.x/wazuh-install.sh](https://packages.wazuh.com/4.x/wazuh-install.sh)
sudo bash ./wazuh-install.sh -a
```
*   Save the generated administrative password and log into the Wazuh Web Dashboard via your Mac browser at `https://<ubuntu-ip>`.

### Phase 3: Provisioning and Onboarding the Windows Endpoint
*   Obtain a Windows 11 ARM64 ISO (via CrystalFetch or Microsoft directly) and spin up a new Windows VM in UTM.
*   Complete the Windows setup wizard (using the network bypass trick during OOBE if necessary).
*   **Fix Network Drivers:** Install UTM's SPICE Guest Tools (mounted via the UTM VM toolbar CD icon) to install the missing virtio network interface card (NIC) driver so the VM gets internet access.
*   **Install the Wazuh Agent:** Open an elevated PowerShell window as Administrator and run the MSI installer script pointing to your Wazuh manager IP:

```powershell
Invoke-WebRequest -Uri "[https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi](https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi)" -OutFile "$env:TEMP\wazuh-agent.msi"
Start-Process msiexec.exe -Wait -ArgumentList "/i", "$env:TEMP\wazuh-agent.msi", "WAZUH_MANAGER=192.168.64.2", "WAZUH_AGENT_NAME=windows-agent"
Start-Service wazuhsvc
```
*   Confirm the agent status shows as active on the Wazuh web dashboard.

### Phase 4: Configuring File Integrity Monitoring (FIM)
*   Edit the agent configuration file on the Windows endpoint (`C:\Program Files (x86)\ossec-agent\ossec.conf`).
*   Add a monitored directory block to track real-time changes:

```xml
<directories check_all="yes" report_changes="yes" realtime="yes">C:\Users\<username>\Desktop\test</directories>
```
*   Restart the Windows Wazuh agent service (`Restart-Service wazuhsvc`) to apply changes.
*   Create, modify, and delete files inside the watched folder to generate real-time FIM alerts on the Wazuh dashboard.

### Phase 5: Deploying the Kali Linux Attacker VM
*   Download the official Kali Linux ARM64 Installer ISO from `kali.org`.
*   Create a new UTM virtual machine using the ARM64 image.
*   **Workaround for ARM black-screen bug:** If the graphical installer fails to render, set the display card to `virtio-ramfb`, add a Serial device (Devices -> New -> Serial) in UTM settings, and run the installation through the text-based serial console.
*   Complete the text installation (Xfce desktop + default tools) and ensure it joins the same Shared Network subnet.

### Phase 6: Simulating Attacks & Verifying Detection
*   **Reconnaissance:** Run an aggressive network scan from Kali against the Windows endpoint IP using `nmap`.
*   **Brute-Force:** Simulate a password attack against the Windows RDP service using `hydra`.
*   **Dashboard Verification:** Navigate to the Wazuh Threat Hunting dashboard to review incoming security events, confirming that authentication failures and network activity are successfully logged and categorized under rule groups like `authentication_failed`.

---

## Challenges I Solved

*   **VM network isolation:** Two VMs on UTM's "Emulated VLAN" mode couldn't reach each other because each received an isolated NAT. Fixed by standardizing all VMs on Shared Network mode.
*   **Windows 11 ARM missing network driver:** A fresh Windows-on-ARM install in UTM lacked a working NIC driver out of the box. Fixed by installing UTM's SPICE Guest Tools package.
*   **Kali ARM64 black-screen installer bug:** A known UTM/Kali ARM64 compatibility issue where the graphical installer fails to render. Worked around by adding a Serial console device and running the installer in text mode.
*   **Silent Wazuh install failures:** The all-in-one installer can silently fail if hardware minimums aren't met, or if the boot media isn't ejected post-install (causing it to loop back into the installer instead of booting the OS).
*   **Windows Firewall blocking recon:** An Nmap scan against the Windows VM returned all ports as "filtered" — serving as real evidence that the default Windows Firewall was functioning correctly, and highlighting that not every attack technique logs the exact same way.

---

## Skills Demonstrated

*   SIEM deployment and configuration
*   Log source onboarding
*   File Integrity Monitoring
*   Attack simulation (reconnaissance + brute-force)
*   Detection engineering fundamentals
*   Linux/Windows administration
*   VM networking troubleshooting

---

## Next Steps

*   Add Grafana for richer dashboards and NetAlertX for network device inventory.
*   Integrate an AI agent to auto-summarize and risk-score FIM alerts.
*   Expand detection coverage beyond RDP (e.g., SSH brute-force, web app attacks via DVWA).
