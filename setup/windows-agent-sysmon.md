# Setup: win11-endpoint, Wazuh Agent & Sysmon Deployment

This document covers connecting the Windows 11 VM as a monitored endpoint: host-only networking, enrolling it as a Wazuh Agent, and deploying Sysmon for richer telemetry. As with the wazuh-mgr build, real troubleshooting is documented rather than smoothed over.

Note on addressing: this document uses `X.X.X.N` placeholder notation, see [`architecture/ip-addresses.md`](../architecture/ip-addresses.md) for the convention.

---

## 1. Networking

Added a second (host-only) network adapter to the existing Windows 11 VM, static address `X.X.X.20`, following the same dual-adapter pattern as wazuh-mgr (NAT for internet, host-only for lab traffic).

### Issue: ping failed in both directions after setting the static IP

Static IP configuration itself succeeded, but connectivity testing revealed layered problems, worth documenting since each one looked identical from the outside (a plain "Request timed out") but had a different root cause:

**Problem 1: Windows Firewall blocks inbound ping by default.** Enabling the built-in "File and Printer Sharing (Echo Request - ICMPv4-In)" rule fixed outbound testing from other hosts, but only after confirming, via `Get-NetFirewallRule`, that the rule was actually enabled for the right network profile. The GUI made it easy to enable the wrong duplicate entry (Domain profile) while the traffic was actually arriving on the Private/Public profile pairing.

**Problem 2: the new adapter was categorized as "Public" by Windows.** Confirmed with:

```powershell
Get-NetConnectionProfile
```

A Public-classified network applies stricter default firewall behavior even after the ICMP rule above is enabled for that profile. Reclassifying to Private is safe here since this interface has no path to the internet or an untrusted network:

```powershell
Set-NetConnectionProfile -InterfaceAlias "Ethernet 2" -NetworkCategory Private
```

**Problem 3: a Group Policy setting blocked the reclassification.** The command above failed with a permissions error citing "Network List Manager Policies," even though the account had local admin rights. Root cause was running PowerShell without elevation, the Group Policy restriction only blocks non-elevated changes. Re-running the same command from an Administrator PowerShell session succeeded immediately.

Once all three were resolved, connectivity was confirmed in both directions.

---

## 2. Wazuh Agent Enrollment

Used the Wazuh Dashboard's built-in "Deploy new agent" wizard (Server management → Endpoints Summary) rather than typing installer flags manually, since it generates a version-matched command with the correct Manager address and an embedded enrollment key.

Selected Windows as the target OS, entered the Manager's static host-only address, and set an explicit agent name (`win11-endpoint`) to match the naming convention used elsewhere in this repo, since the agent name cannot be changed after enrollment.

Ran the generated install command from an elevated PowerShell session (required by the installer), then started the service:

```powershell
NET START WazuhSvc
```

Confirmed in the Dashboard's Endpoints Summary: agent appeared with status `active` within seconds.

---

## 3. Sysmon Deployment

The default Wazuh Agent captures Windows Event Log channels (Application, Security, System) but not process-level detail. Sysmon fills that gap.

Downloaded Sysmon and a proven detection-oriented configuration (SwiftOnSecurity's public config, a widely used baseline ruleset) rather than running Sysmon with its noisy defaults:

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:tmp\Sysmon.zip"
Expand-Archive -Path "$env:tmp\Sysmon.zip" -DestinationPath "$env:tmp\Sysmon" -Force
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "$env:tmp\Sysmon\sysmonconfig.xml"
cd $env:tmp\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Verified locally via `Get-Service Sysmon64` (Running) and Event Viewer, events populated immediately under Applications and Services Logs → Microsoft → Windows → Sysmon → Operational.

### Connecting Sysmon to the Agent

Sysmon logging locally and the Wazuh Agent forwarding that data are two separate concerns. The agent's config (`ossec.conf`) needed an explicit entry telling it to read from Sysmon's event channel, added alongside the existing `localfile` blocks for Application/Security/System:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Restarted the agent service to apply:

```powershell
Restart-Service WazuhSvc
```

Confirmed via the agent's own log that it picked up the new source:

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
```

showed `Analyzing event log: 'Microsoft-Windows-Sysmon/Operational'` alongside the other sources.

---

## 4. End-to-End Verification

Confirmed Sysmon telemetry actually reached the Indexer (not just logging locally) by filtering directly on the Sysmon channel field in the Dashboard's Discover view:

```
data.win.system.channel: "Microsoft-Windows-Sysmon/Operational"
```

Returned genuine hits tied to `win11-endpoint`, confirming the full pipeline: Sysmon → Windows Event Log → Wazuh Agent → Manager → Indexer → Dashboard.

---

## Lessons carried forward

- Symptoms that look identical (a failed ping, a failed connection) can have multiple independent causes stacked on top of each other. Fixing one and retesting before assuming the whole problem is solved avoided misdiagnosing this as a single issue.
- A GUI checkbox being "on" doesn't guarantee it's the right one, when duplicate rule entries exist for different network profiles, verifying via command line (`Get-NetFirewallRule`) removes the ambiguity.
- A component logging data locally does not mean that data is reaching the rest of the pipeline. Verifying at the final destination (the Indexer, via Discover) is the only real confirmation.
