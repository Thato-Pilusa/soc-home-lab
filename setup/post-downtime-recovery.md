# Setup: Post-Downtime Recovery of wazuh-mgr

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
