# IP Address Plan

## Segment

- **Network:** `192.168.56.0/24`
- **Netmask:** `255.255.255.0`
- **Gateway on this interface:** none the default route for internet-bound traffic lives on each VM's separate NAT adapter, not on the host-only adapter.

VirtualBox's default host-only range is restricted to `192.168.56.0/21`. Staying inside `192.168.56.x` avoids any need to edit VirtualBox's global network settings.

---

## Allocation table

| Host | Host-only IP | Role | Assignment method |
|---|---|---|---|
| Windows host | 192.168.56.1 | Analyst workstation | VirtualBox-assigned |
| wazuh-mgr | 192.168.56.10 | Wazuh Manager / Indexer / Dashboard | **Static** |
| win11-endpoint | 192.168.56.20 | Monitored Windows endpoint | Static |
| kali | 192.168.56.30 | Controlled testing | Static |
| ubuntu-endpoint | 192.168.56.40 | Monitored Linux endpoint | Static |

---

## Why static IPs on infrastructure

Every Wazuh agent configuration file (`ossec.conf`) points at the Manager's IP address directly agents don't discover the Manager dynamically. If the Manager's IP changed after a DHCP lease renewal, every agent's connection would silently break with no obvious error pointing back to the cause. Static addressing on all four lab VMs removes that failure mode entirely.

---

## Ports in use

| Port | Protocol | Purpose |
|---|---|---|
| 1514 | TCP | Agent → Manager event stream |
| 1515 | TCP | Agent enrolment |
| 443 | TCP | Wazuh Dashboard (HTTPS) |
| 9200 | TCP | Wazuh Indexer API (internal) |
| 22 | TCP | SSH administration |

---

## Addressing reserved for future growth

`192.168.56.50`–`192.168.56.254` are left unallocated for any additional endpoints or tooling added later in the build (e.g. a second attacker VM, a log forwarder, or a threat-intel enrichment service).
