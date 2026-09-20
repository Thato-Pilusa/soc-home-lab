# IP Address Plan

## Notation

This document uses `X.X.X.N` in place of the lab's actual private IP range. The design and reasoning are independent of the real numbers, only the last octet (`N`) is meaningful, since it identifies each host's role consistently.

## Segment

- Network: a single isolated `/24` host-only subnet
- Netmask: `255.255.255.0`
- Gateway on this interface: none, the default route for internet-bound traffic lives on each VM's separate NAT adapter, not on the host-only adapter

VirtualBox's default host-only range is restricted to a narrow block by default. The actual range in use was chosen to avoid needing to edit VirtualBox's global network settings.

---

## Allocation table

| Host | Host-only address | Role | Assignment method |
|---|---|---|---|
| Windows host | X.X.X.1 | Analyst workstation | VirtualBox-assigned |
| wazuh-mgr | X.X.X.10 | Wazuh Manager / Indexer / Dashboard | Static |
| win11-endpoint | X.X.X.20 | Monitored Windows endpoint | Static |
| kali | X.X.X.30 | Controlled testing | Static |
| ubuntu-endpoint | X.X.X.40 | Monitored Linux endpoint | Static |

---

## Why static IPs on infrastructure

Every Wazuh agent configuration file (`ossec.conf`) points at the Manager's address directly, agents don't discover the Manager dynamically. If the Manager's address changed after a DHCP lease renewal, every agent's connection would silently break with no obvious error pointing back to the cause. Static addressing on all four lab VMs removes that failure mode entirely.

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

The upper portion of the subnet (roughly `X.X.X.50` through `X.X.X.254`) is left unallocated for any additional endpoints or tooling added later in the build (e.g. a second attacker VM, a log forwarder, or a threat-intel enrichment service).
