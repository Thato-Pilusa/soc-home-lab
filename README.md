# SOC Home Lab

A hands-on Security Operations Center (SOC) lab built from scratch to practice the full detection lifecycle: generating security events, centralizing logs, detecting suspicious activity, and investigating alerts end to end, documented as it's built.

This repo is a working build log, not a polished after-the-fact writeup. Commits reflect real progress, real errors, and real fixes.

---

## Why this project exists

Reading about SOC tooling and actually operating it are different skills. This lab is built to close that gap: standing up a SIEM, generating realistic telemetry, tuning detections, and investigating alerts the way an analyst would on the job then documenting the reasoning behind every architectural decision, not just the commands.

---

## Architecture at a glance

```
Endpoint activity
  → Windows Event Log / Sysmon / Linux auth logs
  → Wazuh Agent (local collection)
  → Wazuh Manager (192.168.56.10:1514/tcp)
  → Analysis engine (decoders → rules → alerts)
  → Wazuh Indexer (storage and search)
  → Wazuh Dashboard (visualization)
  → SOC analyst
```

Full network design and rationale: [`architecture/network-diagram.md`](architecture/network-diagram.md)
IP allocation and port reference: [`architecture/ip-addresses.md`](architecture/ip-addresses.md)

---

## Tech stack

| Component | Choice |
|---|---|
| Hypervisor | VirtualBox |
| SIEM | Wazuh (Manager + Indexer + Dashboard) |
| Wazuh server OS | Ubuntu Server 24.04.4 LTS |
| Endpoint 1 | Windows 11 (Wazuh Agent + Sysmon) |
| Endpoint 2 | Ubuntu Server (SSH/auth logs) |
| Attack simulation | Kali Linux (Nmap and other controlled tools, lab-only) |

---

## Lab topology

| Host | Host-only IP | Role |
|---|---|---|
| Windows host | 192.168.56.1 | Analyst workstation |
| wazuh-mgr | 192.168.56.10 | Wazuh Manager / Indexer / Dashboard |
| win11-endpoint | 192.168.56.20 | Monitored Windows endpoint |
| kali | 192.168.56.30 | Controlled testing |
| ubuntu-endpoint | 192.168.56.40 | Monitored Linux endpoint |

All lab traffic is isolated to a VirtualBox host-only network (`192.168.56.0/24`) with no route to the physical LAN, see [`architecture/network-diagram.md`](architecture/network-diagram.md) for why.

---

## Repo structure

```
soc-home-lab/
├── architecture/       # Network design, IP plan, decision rationale
├── setup/              # Step-by-step build documentation
├── detections/         # Custom detection rules and tuning notes
├── investigations/     # Alert walkthroughs and analyst write-ups
├── screenshots/        # Dashboard, endpoint, and investigation evidence
├── diagrams/           # Visual architecture diagrams
├── scripts/            # Helper/automation scripts
└── notes/              # Working notes
```

---

## Build progress

| Milestone | Status |
|---|---|
| VirtualBox ready | ✅ Complete |
| Windows 11 VM | ✅ Complete |
| Lab architecture designed | ✅ Complete |
| Repo structure + core docs | 🔄 In progress |
| Ubuntu Server (wazuh-mgr) installed | ⬜ Not started |
| Wazuh installed | ⬜ Not started |
| Wazuh Agent connected | ⬜ Not started |
| Sysmon deployed | ⬜ Not started |
| First telemetry received | ⬜ Not started |
| First alert investigated | ⬜ Not started |
| Kali added | ⬜ Not started |
| First controlled security test | ⬜ Not started |
| Ubuntu endpoint added | ⬜ Not started |
| Custom detection created | ⬜ Not started |
| Full incident investigation | ⬜ Not started |

---

## About this build

This lab is built in progressive phases with each step documented before moving to the next: VM provisioning, SIEM installation, agent deployment, detection engineering, and finally live investigation of simulated attacks. Every phase folder explains not just *what* was run, but *why* the design trade-offs, the failed attempts, and the fixes.
