# Wazuh SIEM Home Lab — Detecting an RDP Brute-Force Attack

A hands-on SOC analyst home lab built from scratch on Apple Silicon using UTM for virtualization. Deploys a working SIEM, connects a monitored Windows endpoint, launches a real attack from a separate Kali Linux attacker VM, and verifies end-to-end detection on the Wazuh dashboard.

## Why I built this

I'm transitioning into a cybersecurity/SOC analyst role and wanted hands-on proof I can deploy, configure, and operate a real SIEM stack — not just talk about the theory.

## Architecture

Three virtual machines on a single Mac (Apple Silicon, UTM), all on the same private network:

```
┌─────────────────┐        ┌──────────────────────┐
│   Ubuntu Server  │◄──────►│   Windows 11 (ARM)    │
│  Wazuh Manager +  │  logs  │   Wazuh Agent         │
│  Indexer + Dashboard      │   (monitored endpoint) │
│  192.168.64.2    │        │   192.168.64.4        │
└────────┬─────────┘        └────────────▲──────────┘
         │                                │
         │ dashboard (browser)      RDP brute-force
         │                                │
         │                       ┌────────┴──────────┐
         └──────────────────────►│   Kali Linux        │
                                  │   (attacker VM)     │
                                  │   192.168.64.5      │
                                  └─────────────────────┘
```

## Tools & tech stack

- **Wazuh 4.14** — SIEM manager, indexer, and dashboard
- **Ubuntu Server 26.04** — Wazuh server host
- **Windows 11 (ARM)** — monitored endpoint running the Wazuh agent
- **Kali Linux (ARM64)** — attacker machine
- **UTM** — virtualization on Apple Silicon (QEMU-based)
- **Hydra** — password brute-force tool
- **Nmap** — network reconnaissance

## What I built

1. **Deployed a Wazuh SIEM server** on Ubuntu, including the manager, indexer, and web dashboard, all self-hosted in a VM.
2. **Connected a Windows endpoint** as a monitored agent, resolving real driver and networking issues along the way (see Challenges below).
3. **Configured File Integrity Monitoring (FIM)** on the Windows agent, verified detection of file add/modify/delete events in real time.
4. **Built a separate Kali Linux attacker VM** and used it to run reconnaissance (Nmap) and an RDP brute-force attack (Hydra) against the Windows endpoint.
5. **Verified end-to-end detection**: Wazuh correctly classified and alerted on 7 failed RDP login attempts (rule ID 60122, "Logon Failure - Unknown user or bad password") sourced from the Kali attacker's IP, distinguishing them from the legitimate login.

## Key result

Wazuh's Threat Hunting dashboard showed:
- **7** authentication failure events
- **2** authentication success events
- Correctly grouped under the `authentication_failed` rule group
- Each event timestamped and attributable to the source (Kali VM)

*(Screenshot: dashboard showing 7 failed logon attempts — add yours here)*
*(Screenshot: Nmap scan output showing Windows Firewall blocking unsolicited probes — add yours here)*
*(Screenshot: FIM detecting a file change in near-real time — add yours here)*

## Challenges I solved

- **VM network isolation**: Two VMs on UTM's "Emulated VLAN" mode can't reach each other — each gets an isolated NAT. Fixed by standardizing all VMs on "Shared Network" mode.
- **Windows 11 ARM missing network driver**: A fresh Windows-on-ARM install in UTM has no working NIC driver out of the box. Fixed by installing UTM's SPICE Guest Tools package.
- **Kali ARM64 black-screen installer bug**: A known UTM/Kali ARM64 compatibility issue where the graphical installer never renders. Worked around it by adding a Serial console device and running the installer in text mode.
- **Silent Wazuh install failures**: The all-in-one installer can silently fail if hardware minimums aren't met, or if the boot media isn't ejected after install (causing it to re-run the installer instead of booting the OS).
- **Windows Firewall blocking recon**: An Nmap scan against the Windows VM returned all ports "filtered" — useful evidence that default Windows Firewall was working as intended, and a reminder that not every attack technique generates a log entry the same way.

## Skills demonstrated

SIEM deployment and configuration · log source onboarding · File Integrity Monitoring · attack simulation (recon + brute-force) · detection engineering fundamentals · Linux/Windows administration · VM networking troubleshooting

## Next steps

- [ ] Add Grafana for richer dashboards and NetAlertX for network device inventory
- [ ] Integrate an AI agent to auto-summarize and risk-score FIM alerts
- [ ] Expand detection coverage beyond RDP (e.g. SSH brute-force, web app attacks via DVWA)

