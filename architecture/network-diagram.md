# Network Architecture

## Design goal

Every VM in this lab needs internet access for updates and package installs, but attack simulation traffic (port scans, exploit attempts) must never leave the host or touch the physical home network. This lab uses a **dual-adapter design** on every VM to separate those two concerns cleanly.

---

## VirtualBox networking modes — comparison

| Mode | Internet | VM-to-VM | Host-to-VM | Reaches physical LAN |
|---|---|---|---|---|
| NAT | Yes | No | No | No |
| Bridged | Yes | Yes | Yes | **Yes** |
| Host-only | No | Yes | Yes | No |
| Internal | No | Yes | No | No |

No single mode covers both requirements (internet access + isolated lab traffic), so each VM runs two virtual network adapters.

---

## Decision: two adapters per VM

- **Adapter 1 — NAT:** outbound internet only, for OS updates and package installs. No lab traffic ever crosses this adapter.
- **Adapter 2 — Host-only:** the lab backbone. All Wazuh agent telemetry, all Kali test traffic, and dashboard access go over this segment.

### Why not Bridged
Bridged mode would put every VM on the real home network's IP range. A Kali port scan or exploit test would then be visible to — and could affect — any other device on that LAN. Unacceptable for a lab meant to run offensive tooling.

### Why not Internal-only
Internal networking blocks host-to-VM communication entirely. The Windows host itself needs to reach the Wazuh Dashboard over HTTPS in a browser, which Internal mode would prevent.

**Host-only** is the only mode that isolates lab traffic from the physical network while still letting the host reach the dashboard and letting VMs reach each other.

---

## Data flow

```
Endpoint activity
  → Windows Event Log / Sysmon / Linux auth logs
  → Wazuh Agent (local collection)
  → Wazuh Manager  192.168.56.10:1514/tcp
  → Analysis engine (decoders → rules → alerts)
  → Wazuh Indexer (storage and search)
  → Wazuh Dashboard (visualisation)
  → SOC analyst
```

Each monitored endpoint runs a Wazuh Agent that forwards local log data to the Wazuh Manager over the host-only segment. The Manager decodes and evaluates events against detection rules, indexes matches for search, and surfaces alerts on the Dashboard.

---

## Topology diagram

```
                         ┌─────────────────────────┐
                         │      Windows Host        │
                         │   192.168.56.1 (host-only)│
                         │   Analyst workstation     │
                         └────────────┬─────────────┘
                                      │
                     Host-only network: 192.168.56.0/24
                                      │
        ┌─────────────┬──────────────┼───────────────┬──────────────┐
        │              │              │               │              │
  ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐  ┌──────▼──────┐
  │ wazuh-mgr │  │win11-     │  │  kali     │  │ ubuntu-     │
  │ .10       │  │endpoint   │  │  .30      │  │ endpoint    │
  │ Manager/  │  │.20        │  │  Testing  │  │ .40         │
  │ Indexer/  │  │Agent +    │  │  (Nmap,   │  │ SSH/auth    │
  │ Dashboard │  │Sysmon     │  │  lab-only)│  │ logs        │
  └─────┬─────┘  └─────┬─────┘  └───────────┘  └──────┬──────┘
        │              │                              │
        └──────── NAT adapter (each VM) ───────────────┘
                        │
                  Internet (updates only —
                  no lab traffic)
```

Each box also carries a separate NAT adapter (not shown as shared) purely for outbound internet access — it is not part of the lab traffic path.

---

## Ports in use

| Port | Protocol | Purpose |
|---|---|---|
| 1514 | TCP | Agent → Manager event stream |
| 1515 | TCP | Agent enrolment |
| 443 | TCP | Wazuh Dashboard (HTTPS) |
| 9200 | TCP | Wazuh Indexer API (internal) |
| 22 | TCP | SSH administration |

Full IP allocation table: [`ip-addresses.md`](ip-addresses.md)
