# Setup: Post-Downtime Recovery and Upgrade Hardening of wazuh-mgr

After the lab sat powered off for several weeks, the Wazuh stack did not come back cleanly. This note records what broke, how it was fixed, and how the fix was verified.

---

## Symptoms

- Dashboard unreachable from the analyst workstation.
- `wazuh-indexer` failed to start; `systemctl status` and `journalctl` showed the service exiting during startup.
- `wazuh-manager` and `filebeat` were not running either.

## Root causes

1. **Missing Indexer log directory.** `/var/log/wazuh-indexer` and its `tmp` subfolder no longer existed, so the Indexer could not write logs or temp files and aborted at startup. Why the folder disappeared is not confirmed; it is worth watching if it recurs after another long shutdown.
2. **Services not enabled on boot.** The Manager and Filebeat had been started manually but never enabled, so they would not return after a power cycle.

## Fix

Recreate the directory with correct ownership and permissions:

```bash
sudo mkdir -p /var/log/wazuh-indexer/tmp
sudo chown -R wazuh-indexer:wazuh-indexer /var/log/wazuh-indexer
sudo chmod 750 /var/log/wazuh-indexer
```

Enable and start every component so it survives reboots:

```bash
sudo systemctl enable --now wazuh-indexer
sudo systemctl enable --now wazuh-manager
sudo systemctl enable --now filebeat
sudo systemctl enable --now wazuh-dashboard
```

## Verification

- Checked each service with `systemctl status`.
- Rebooted the VM and confirmed all four services came up on their own, and the agent on win11-endpoint reconnected.
- Confirmed the Dashboard loaded and Sysmon events were still searchable.

## Lessons

- "Installed and working" is not the same as "survives a reboot." A reboot test should be part of every build.
- Read the service journal first; the failing path in the log pointed straight at the missing folder.
- Defer system updates until a known-good state is confirmed, so a failure has one variable, not two.

---

# Incident 2: Indexer timeout after a system upgrade

Once the stack was stable, the pending OS updates were applied with `apt upgrade` followed by a reboot. Two of the four services then failed to start.

## Symptoms

- `wazuh-indexer` and `wazuh-manager` reported `failed` after the reboot; Filebeat and the Dashboard were `active`.
- The Indexer journal showed only Java warnings, then `start operation timed out. Terminating.` after roughly 12 minutes.

## Diagnosis

1. **Package versions.** `dpkg -l` showed every Wazuh component had been upgraded together from 4.9.2 to 4.14.10, and the Indexer's OpenSearch from 2.13 to 2.19. The versions matched each other, so this was not a component mismatch. The upgrade happened because the Wazuh packages were not on hold, so a routine `apt upgrade` pulled them in.
2. **Indexer log.** `wazuh-cluster.log` stopped after node initialization with no error, which pointed to a slow start, not a crash. The first start after the upgrade was heavy (index and plugin loading) while four services competed for CPU, RAM and disk on a small VM.
3. **Manual start.** Starting the Indexer alone, after the boot rush, completed normally: indices loaded and cluster health moved from RED to GREEN. Starting the Manager afterwards also succeeded.

## Fix

Allow the Indexer a much longer start window, so a slow boot is not treated as a failure:

```bash
sudo mkdir -p /etc/systemd/system/wazuh-indexer.service.d
printf '[Service]\nTimeoutStartSec=1800\n' | sudo tee /etc/systemd/system/wazuh-indexer.service.d/override.conf
sudo systemctl daemon-reload
```

Hold the Wazuh packages so they only change when upgraded deliberately:

```bash
sudo apt-mark hold wazuh-indexer wazuh-manager wazuh-dashboard filebeat
```

## Verification

- Rebooted with the override and hold in place; all four services returned to `active` without manual action.
- Checked the VM clock with `timedatectl` (synchronized, NTP active), since accurate timestamps matter for alert correlation.

## Lessons

- Package holds belong in the build checklist for any SIEM stack whose components must stay version-matched.
- A service that exits with no error and a timeout is a different problem from one that crashes. Read how long it ran and what it consumed before assuming a fault.
- Test slow-starting services in isolation to separate a real fault from boot-time resource contention.
- Pair every upgrade with a reboot test.
