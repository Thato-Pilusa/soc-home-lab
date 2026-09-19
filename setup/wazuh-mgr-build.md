# Setup: wazuh-mgr — VM Build, Networking & Wazuh Installation

This document covers building the `wazuh-mgr` VM from scratch: VirtualBox provisioning, dual-adapter networking, static IP configuration, and installing Wazuh (Indexer + Manager + Dashboard). It includes the real troubleshooting encountered along the way — left in deliberately, since diagnosing and resolving issues is a core part of SOC analyst work.

---

## 1. VM Provisioning

Created in VirtualBox per the spec in [`architecture/ip-addresses.md`](../architecture/ip-addresses.md):

| Setting | Value |
|---|---|
| Name | wazuh-mgr |
| OS | Ubuntu Server 24.04.4 LTS |
| Base Memory | 5120 MB *(see note below — plan called for 4 GB)* |
| Processors | 2 |
| Disk | 50 GB, VDI, dynamically allocated |

**Note on RAM:** the architecture plan specified 4096 MB. In practice, `free -h` inside the guest showed only ~3.3 GB usable at that allocation — kernel/firmware overhead (this VM has EFI enabled) ate into the rest. Since Wazuh's Indexer reserves JVM heap on startup and needs genuine 4 GB+ to avoid degraded dashboard performance, Base Memory was increased to 5120 MB to guarantee real 4 GB+ usable inside the guest. Confirmed afterward: `free -h` reported 4.3 GB total.

---

## 2. Networking

Two adapters, per the architecture design ([full rationale here](../architecture/network-diagram.md)):

- **Adapter 1 — NAT:** outbound internet only, for package installs.
- **Adapter 2 — Host-only:** static `192.168.56.10/24`, the lab backbone.

### Issue: host-only network created on the wrong subnet

VirtualBox auto-created its default host-only adapter on `192.168.187.0/24` instead of the planned `192.168.56.0/24`. Rather than redesign the IP plan around VirtualBox's default, the host-only adapter's IPv4 address was edited directly in VirtualBox's Host Network Manager to `192.168.56.1/24`, matching the original plan. The VM's Adapter 2 setting itself needed no change — it was already correctly attached to the same host-only network, just under the corrected range.

### Issue: "Failed to open/create the internal network" on boot

After editing the host-only adapter's IP, the VM failed to boot with `VERR_INTNET_FLT_IF_NOT_FOUND`. Root cause: Windows' network driver binding for the VirtualBox Host-Only Ethernet Adapter had been knocked loose by the IP change. Fixed by disabling and re-enabling the adapter in Windows' Network Connections panel — no VirtualBox-side reinstall needed.

### Static IP via Netplan

Target: `enp0s8` (host-only) → `192.168.56.10/24`, no gateway on this interface (default route stays on the NAT adapter, `enp0s3`).

Working config (`/etc/netplan/00-installer-config.yaml`), using YAML flow-style syntax rather than nested block indentation:

```yaml
network: {version: 2, ethernets: {enp0s3: {dhcp4: true, dhcp6: true, match: {macaddress: "08:00:27:72:8c:6a"}, set-name: "enp0s3"}, enp0s8: {dhcp4: false, addresses: ["192.168.56.10/24"]}}}
```

**Why flow-style:** this VM has no desktop environment, so VirtualBox's shared clipboard doesn't work with the console — every config line had to be typed manually into `nano`, which repeatedly produced inconsistent-indentation errors from single-space mistakes in nested block YAML. Flow-style JSON-like syntax removes indentation from the equation entirely while remaining valid YAML, making it far less error-prone to type by hand.

Verified with `sudo netplan try`, then `ip a` confirmed `192.168.56.10/24` on `enp0s8`. Connectivity confirmed both directions:
- Host → VM: `ping 192.168.56.10` from Windows Command Prompt — 0% loss
- VM → Host: `ping 192.168.56.1` from the guest

### SSH access

`openssh-server` was already active on the guest. Connected from Windows via the built-in SSH client:

```
ssh thato@192.168.56.10
```

This replaced typing directly into the VirtualBox console window and enabled copy-paste from a normal terminal for all subsequent work.

---

## 3. Wazuh Installation

Used Wazuh's all-in-one installer (Indexer + Manager + Dashboard on a single node):

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

### Issue: hardware requirements check failed

Before the RAM fix above, the installer refused to proceed:

```
ERROR: Your system does not meet the recommended minimum hardware requirements of 4Gb of RAM and 2 CPU cores.
```

Resolved by the Base Memory increase described in Section 1, rather than bypassing the check with `-i` — since the underlying concern (genuine 4 GB+ for the Indexer's heap) was real, not just a check to satisfy.

### Issue: SSH disconnect killed the installer mid-run

The first full install attempt was run as a normal foreground SSH command. The SSH session dropped partway through the Dashboard stage (`client_loop: send disconnect: Connection reset`), which killed the installer process along with it — Indexer, Manager, and Filebeat had already completed successfully, but the Dashboard component was left in a partially-installed, non-running state.

**Fix — run detached from the SSH session:**

```bash
sudo nohup bash wazuh-install.sh -wd wazuh-dashboard -o > wazuh-dashboard-install.log 2>&1 &
tail -f wazuh-dashboard-install.log
```

`nohup ... &` runs the installer independently of the SSH session, so a dropped connection no longer kills the install. Output is redirected to a log file and watched with `tail -f`, which can be safely interrupted (Ctrl+C) and resumed at any time without affecting the running install.

Note: `nohup` protects against a dropped *SSH session*, but not against the *VM itself* rebooting — a VM restart kills all processes regardless. Confirmed this distinction directly when a VM reboot between sessions had silently ended an earlier detached run, identified via `uptime` showing a fresh boot time inconsistent with the expected session length.

Targeting just the dashboard component required the node name from the installer's generated config, rather than guessing at the flag syntax:

```bash
sudo tar -xOf wazuh-install-files.tar wazuh-install-files/config.yml
```

which showed the dashboard node was named `wazuh-dashboard`, giving the correct command: `wazuh-install.sh -wd wazuh-dashboard`.

### Issue: dashboard install failed — indexer security not initialized

The re-run failed with:

```
ERROR: Cannot connect to Wazuh dashboard.
ERROR: Wazuh indexer security settings not initialized.
```

Rather than reaching for the installer's suggested workaround (`-fd`, force-install past the check), the Indexer's actual health was verified directly first:

```bash
curl -kv -u admin:'<password>' https://localhost:9200
```

This returned a clean `401 Unauthorized` from a genuine OpenSearch Security realm — confirming the Indexer process was healthy and its security layer was active, but not properly initialized with the expected credentials. This ruled out forcing past the check (which would have left a real configuration problem unresolved) in favor of the root-cause fix:

```bash
sudo nohup bash wazuh-install.sh -s > wazuh-cluster-init.log 2>&1 &
```

After this completed, the same `curl` command returned `200 OK` with full cluster JSON, confirming the fix. The dashboard install was then re-run and completed cleanly.

---

## 4. Verification

```bash
sudo systemctl status wazuh-indexer wazuh-manager filebeat wazuh-dashboard --no-pager
```

All four components confirmed `active (running)`. Dashboard accessed from the Windows host browser at `https://192.168.56.10`, logged in with the admin credentials generated during install (stored in `wazuh-install-files.tar` → `wazuh-passwords.txt` on the VM).

---

## Lessons carried forward

- Verify a service's actual health directly (`curl` against its API) rather than trusting a dependent component's error message about it — the dashboard installer's suggested fix (`-fd`, bypass) would have masked a real problem instead of fixing it.
- Any long-running install over SSH should run detached (`nohup ... &`) from the start, not just after the first disconnect.
- `nohup` survives a dropped SSH session but not a VM reboot — worth remembering before assuming a background job is still running after time has passed.
