# Home SOC Detection Lab — Wazuh SIEM

A self-built Security Operations Center lab used to practice log collection, detection engineering, and incident triage in an environment that mirrors a real Tier 1 SOC workflow.

## Overview

This project simulates a small enterprise network monitored by a SIEM. A Windows 11 endpoint is instrumented with Sysmon and forwards security-relevant events to a Wazuh Manager. An attacker machine (Kali Linux) is used to simulate a real-world attack technique (SSH brute-force), and the resulting telemetry is analyzed inside the Wazuh dashboard to confirm detection, map the activity to MITRE ATT\&CK, and produce an incident report.

The goal was not just to "install a SIEM," but to go through the full loop a junior SOC analyst is expected to handle: build the monitoring pipeline, generate real attack traffic, confirm the alert fired, and document the finding.

## Architecture

```
                 Host-only network (192.168.139.0/24)
   ┌──────────────────────┐        ┌──────────────────────┐
   │   Ubuntu Server 22.04 │        │   Windows 11 Home     │
   │   Wazuh Manager /     │◄──────►│   Wazuh Agent         │
   │   Indexer / Dashboard │  1514  │   + Sysmon            │
   │   (SIEM)              │  1515  │   + OpenSSH Server    │
   └──────────────────────┘        └──────────▲───────────┘
                                                │ SSH (22)
                                     ┌──────────┴───────────┐
                                     │     Kali Linux        │
                                     │     (Attacker)        │
                                     └────────────────────────┘
```

All three VMs run in VirtualBox on a Host-only network, isolated from the host's LAN, with NAT on a separate adapter for internet access (package installs, updates).

## Tech Stack

* **SIEM:** Wazuh 4.9.2 (Manager, Indexer, Dashboard — all-in-one install)
* **Endpoint telemetry:** Sysmon (SwiftOnSecurity config) + Windows OpenSSH Server logs
* **Attack simulation:** Kali Linux, custom SSH brute-force script (`sshpass` + bash loop)
* **Detection framework:** MITRE ATT\&CK
* **Virtualization:** VirtualBox, Host-only + NAT networking

## What Was Built

1. **Wazuh Manager** deployed on Ubuntu Server, exposed via HTTPS dashboard.
2. **Windows 11 endpoint** enrolled as a Wazuh agent, forwarding two event channels:

   * `Microsoft-Windows-Sysmon/Operational` — process creation, network connections, etc.
   * `OpenSSH/Operational` — SSH authentication events (success/failure)
3. **Kali Linux** attacker box used to run a brute-force login attempt against the Windows endpoint's SSH service.
4. Verified the failed login attempts were **ingested, parsed, and surfaced as alerts** in the Wazuh dashboard, correlated to the attacking host and mapped against MITRE ATT\&CK.

## Attack Simulation — SSH Brute Force

* **Technique:** Password guessing against a live SSH service
* **MITRE ATT\&CK:** [T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/) (sub-technique T1110.001 — Password Guessing)
* **Source:** Kali Linux (192.168.139.7)
* **Target:** Windows 11 endpoint, OpenSSH Server (192.168.139.6:22)
* **Method:** 20 sequential login attempts against a local Windows account using a wordlist-derived password set

**Result:** All 20 failed authentication attempts were captured by the Wazuh agent via the `OpenSSH/Operational` event channel and appeared as discrete events in the dashboard within seconds of occurring, correctly attributed to the source IP, target account, and timestamp.

## Challenges \& Solutions

Building this lab surfaced a number of real infrastructure problems — working through them is arguably the more valuable part of the exercise:

|Problem|Root Cause|Fix|
|-|-|-|
|Ubuntu installer hung indefinitely in VirtualBox|Two active network adapters caused a DHCP negotiation loop in the Subiquity installer|Disabled the second (Host-only) adapter during install, re-enabled it after the OS was installed|
|Couldn't add an RDP-based Windows agent|Windows 11 **Home** edition does not include an RDP server (Pro/Enterprise only)|Switched the attack vector to SSH — enabled Windows' built-in OpenSSH Server instead|
|Hydra failed with "too many connection errors" even at 1 thread|Hydra's SSH library was incompatible with the target's modern OpenSSH cipher/key-exchange set|Replaced Hydra with a custom bash loop using `sshpass` and the system SSH client, which handled the handshake correctly|
|New Wazuh agent stuck at "disconnected" despite a clean agent log|The Ubuntu VM's root partition was **100% full**, causing the Wazuh Indexer's SQLite backend to silently fail agent registration|Resized the VirtualBox VDI disk and grew the partition with `growpart` / `resize2fs`|
|No SSH failure events reaching the dashboard|The default `ossec.conf` only ships Sysmon telemetry; Windows OpenSSH logs its own separate Windows Event Log channel|Added an explicit `<localfile>` block for the `OpenSSH/Operational` channel to the agent config|

## Repository Contents

* `README.md` — this file
* `incident-report.md` — full incident response write-up (NIST-style) for the SSH brute-force detection

## Key Takeaways

* Detection is only as good as the log sources feeding it — the default agent config did **not** cover SSH auth events, and that gap had to be found and closed manually.
* Infrastructure issues (disk space, networking) can silently break detection pipelines without any obvious error on the surface — the agent reported a healthy connection while the backend was actually rejecting data.
* Mapping detected activity to MITRE ATT\&CK turns a raw alert into something that communicates risk and intent, not just "something happened."

## Next Steps

* Add a second detection scenario (suspicious PowerShell execution — T1059) with a custom Sigma rule.
* Simulate a small Active Directory environment to cover authentication-based lateral movement detections.
* Build out a Sigma rule repository mapped to MITRE ATT\&CK, portable across SIEMs.

