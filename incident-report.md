# Incident Report: SSH Brute Force Attempt Against Windows Endpoint

**Report ID:** IR-2026-002
**Classification:** Simulated / Lab Environment
**Analyst:** \[versex-arch]
**Date:** September 2, 2026

\---

## 1\. Executive Summary

On September 2, 2026, a simulated brute-force authentication attack was launched from an attacker-controlled host (Kali Linux, `192.168.139.7`) against the SSH service of a monitored Windows 11 endpoint (`victim-win2`, `192.168.139.6`). The activity consisted of 20 sequential failed login attempts against a local user account over a short window. All attempts were successfully captured and alerted on by the Wazuh SIEM via the endpoint's `OpenSSH/Operational` Windows Event Log channel. No successful authentication occurred. This report documents the detection, the underlying technique, and recommended mitigations.

## 2\. Scope

* **Environment:** Isolated lab network (VirtualBox Host-only, `192.168.139.0/24`)
* **Affected asset:** `victim-win2` — Windows 11 Home, Wazuh Agent ID 003
* **Detection source:** Wazuh Manager (`192.168.139.5`), SIEM correlation engine
* **Attacker asset:** Kali Linux (`192.168.139.7`) — controlled, part of the lab

## 3\. Timeline-Date: September 2, 2026

|Step|Event|
|-|-|
|1)|Attack script initiated on Kali Linux against `192.168.139.6:22`|
|2)|20 sequential SSH authentication attempts made against local account `vboxuser`, each followed by a 1-second delay|
|3)|Windows OpenSSH Server logs each failure to the `OpenSSH/Operational` event channel in real time|
|4)|Wazuh Agent forwards each event to the Wazuh Manager over the encrypted agent channel (TCP/1514)|
|5)|Analyst confirms all 20 events present in the Wazuh Dashboard, correlated to source IP and target account|

## 4\. Technical Details

* **Technique:** Password guessing brute force against a live SSH service
* **Target account:** Local Windows account (`vboxuser`)
* **Attempts:** 20 (subset of a common password list)
* **Protocol/Port:** SSH, TCP/22
* **Result:** 0 successful authentications; all attempts failed and were logged

### Detection Path

1. Windows OpenSSH Server writes each failed login to the `OpenSSH/Operational` Windows Event Log.
2. The Wazuh Agent, configured with an explicit `<localfile>` entry for this channel, tails the log in near real time.
3. Events are forwarded to the Wazuh Manager, where they are parsed, indexed, and made available in the Security Events / Threat Hunting views.
4. Each event includes the source IP, target username, and timestamp — sufficient to reconstruct the full attempt sequence.

## 5\. MITRE ATT\&CK Mapping

|Tactic|Technique|ID|
|-|-|-|
|Credential Access|Brute Force: Password Guessing|[T1110.001](https://attack.mitre.org/techniques/T1110/001/)|

## 6\. Impact Assessment

In this lab scenario, no compromise occurred — the target account's password was not present in the attempted set. In a production environment, an unmitigated brute-force attempt against an internet- or network-exposed SSH service carries a **Medium-High** risk, particularly if:

* The target account has weak or reused credentials
* No account lockout / rate limiting is configured
* The service is reachable from an untrusted network segment

## 7\. Recommendations

1. **Rate limiting / lockout:** Configure `MaxAuthTries` and connection throttling on OpenSSH Server (already partially observed via the built-in `MaxStartups` protection during testing).
2. **Key-based authentication:** Disable password authentication for SSH in favor of key pairs where operationally feasible.
3. **Network segmentation:** Restrict SSH exposure to only the hosts/subnets that require it; do not expose management protocols broadly.
4. **Alerting threshold:** Create a correlation rule that escalates to a higher severity when failed attempts from a single source exceed a defined threshold within a short window (brute-force pattern), rather than relying on individual failed-login events alone.
5. **MFA:** Where supported, require multi-factor authentication for remote administrative access.

## 8\. Evidence

* Dashboard view: 20 matching events for `agent.name: victim-win2 AND data.win.system.channel: OpenSSH/Operational`
* Raw `full\\\\\\\_log` entry showing `Failed password for vboxuser from 192.168.139.7`
* Windows Event Viewer (`OpenSSH/Operational`) corroborating the same failures at the source



## 9\. Conclusion

The detection pipeline performed as designed: telemetry from a non-default log source (OpenSSH on Windows) was correctly configured, collected, and correlated by the SIEM, allowing full reconstruction of the attack timeline without any successful compromise. This exercise validated both the monitoring configuration and the analyst's ability to identify, confirm, and document a credential-access attempt end to end.

