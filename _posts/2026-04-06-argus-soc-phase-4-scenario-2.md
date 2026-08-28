---
title: "Building Argus SOC  |  Phase 4 Scenario 2  |  Brute Force — Hydra vs the Honeypot"
date: 2026-04-06 10:00:00 +0100
categories: [Projects, ARGUS-SOC]
tags: [soc, homelab, hydra, cowrie, honeypot, brute-force, detection-engineering, wazuh, claude, telegram, blue-team, red-team, mitre-attack]
description: "Phase 4 Scenario 2 of the Argus SOC build — running Hydra against the Cowrie SSH honeypot. The first scenario where the full detection pipeline fires end-to-end: attack → Cowrie capture → Wazuh → Claude triage → Telegram alert."
image:
  path: /assets/img/posts/argus-soc/banner.png
  alt: Argus SOC — AI-Augmented Home Lab Security Operations Center
---

> 📌 **Author's note** — This post documents the Argus SOC lab at the time of publication, when the Pi 3B+ served as the MSSP edge sensor and the Pi 5 hosted the vulnerable Docker targets. The architecture was redesigned in [Phase 5](/posts/argus-soc-phase-5/), which introduced a ThinkCentre M920x running Proxmox with an Active Directory lab as the client infrastructure, and moved the edge-sensor role onto the Pi 5. The detection logic, custom rules, and gap analysis described here remain valid; only the host topology has changed.
>
> Build carried out on real hardware in a controlled home lab. Claude (Anthropic) was used as a reasoning and writing assistant — all deployments, attacks, configurations, and verifications were performed by the author.

## Overview

Scenario 2 is the first end-to-end test of the entire Argus SOC pipeline. Where Scenario 1's reconnaissance alerts were filtered as noise (level 3, never reaching Claude), this brute force attack triggers Cowrie honeypot login rules at level 10 — high enough to pass the pre-filter, get classified by Claude, and land as a formatted alert in Telegram.

The target is the Cowrie SSH honeypot on Pi 3B+ (port 22). The attacker runs Hydra from Kali with the rockyou.txt wordlist. Cowrie accepts weak credentials by design, captures the full session, and Wazuh forwards everything to the SOC.

---

## The Attack

From Kali Linux VM on the Laptop

**Timestamp:** 2026-04-04 11:35 UTC
**Command:** `hydra -l root -P /usr/share/wordlists/rockyou.txt -t 4 -V -f 192.168.1.20 ssh`
**Duration:** ~33 seconds
**Result:** Hydra found valid credentials `root:123456789` after 4 attempts

![Hydra brute force output](/assets/img/posts/argus-soc/phase-4/scenario-2/phase4_scenario2_hydra-output.png){: width="700" }
_Hydra brute force against Cowrie — credentials found in 4 attempts_

Hydra tested passwords from rockyou.txt against Cowrie's fake SSH server. Cowrie selectively rejected some passwords, and ended up accepting `12345`, `123456789` and `password`. The `-f` flag stopped Hydra after the first success, but Cowrie logged every attempt.

The attacker believes they've compromised a real Linux server. Cowrie presented a convincing fake shell (`root@svr04`), accepted the credentials, and recorded everything.

---

## What Cowrie Captured

The honeypot recorded the complete attacker session — this is real threat intelligence:

- **Session connect** — new connection from 192.168.1.101
- **Client fingerprint** — SSH-2.0-libssh_0.11.3 (identifies the tool)
- **Key exchange** — full cryptographic handshake logged
- **Failed login** — root/123456 rejected (Cowrie selectively rejects some credentials)
- **Successful logins** — root/12345, root/123456789, root/password all accepted
- **Session duration** — 24.2 seconds total
- **Session closed** — connection terminated

![Cowrie live logs during brute force](/assets/img/posts/argus-soc/phase-4/scenario-2/phase4_scenario2_cowrie-logs-live.png){: width="700" }
_Cowrie live logs — every credential attempt, SSH fingerprint, and session event captured_

Every credential attempted, the SSH client fingerprint, and the session timing feed into attacker profiling. In a real MSSP deployment, this data maps directly to threat intelligence reports.

---

## Detection Results

10 Wazuh alerts fired across the brute force window:

| Rule ID | Description | Level | Count |
|---|---|---|---|
| 100010 | Cowrie: New honeypot connection | 6 | 5 |
| 100011 | Cowrie: Successful honeypot login | 10 | 2 |
| 5501 | PAM: Login session opened | 3 | 1 |
| 5502 | PAM: Login session closed | 3 | 1 |
| 5402 | Successful sudo to ROOT | 3 | 1 |

![Wazuh alert list — Cowrie rules firing](/assets/img/posts/argus-soc/phase-4/scenario-2/phase4_scenario2_wazuh-alert-list.png){: width="700" }
_Wazuh dashboard — rules 100010 and 100011 firing correctly_

The custom Cowrie rules performed exactly as designed. Rule 100010 (level 6) caught every new connection — these were filtered by n8n's pre-filter as expected, since level 6 is below the threshold. Rule 100011 (level 10) caught the successful logins — these passed the pre-filter and reached Claude.

---

## Full Pipeline — End to End

This is the first scenario where every stage of the pipeline fired:

```
Hydra brute force (Kali 192.168.1.101)
       ↓
Cowrie honeypot accepts credentials, logs session
       ↓
Wazuh Agent reads cowrie.json → fires rules 100010 (level 6) + 100011 (level 10)
       ↓
Wazuh Manager → webhook → n8n
       ↓
Pre-filter: level 6 → false path (filtered) ✅
Pre-filter: level 10 → true path (to Claude) ✅
       ↓
Claude API: severity=MEDIUM, T1110.001, confidence 0.85
       ↓
Severity Router → medium → Telegram ✅
```

![n8n pipeline — full path from Claude to Telegram](/assets/img/posts/argus-soc/phase-4/scenario-2/phase4_scenario2_n8n-full-pipeline-claude-to-telegram.png){: width="700" }
_n8n execution — alert passes pre-filter, Claude classifies as MEDIUM, Telegram delivers_

---

## Claude's Classification

Claude correctly identified both successful honeypot logins as credential brute force:

**Alert 1 (root/123456789):**
- **Severity:** MEDIUM
- **MITRE:** T1110.001 — Password Guessing
- **Summary:** "A successful login occurred on the Cowrie honeypot (port 22) from internal IP 192.168.1.101 using credentials root/123456789. This indicates either a compromised internal host conducting lateral movement or an insider threat, as the source is within the HOME.NET."
- **Action:** investigate
- **Confidence:** 0.85

**Alert 2 (root/password):**
- **Severity:** MEDIUM
- **MITRE:** T1110.001 — Password Guessing
- **Summary:** "Successful login to Cowrie honeypot on argus-edge-01 from internal IP 192.168.1.101 using default credentials (root/password). This indicates either a compromised internal device attempting lateral movement or unauthorized internal access to the honeypot."
- **Action:** investigate
- **Confidence:** 0.85

Claude classified both as MEDIUM rather than CRITICAL because the source IP is internal (192.168.1.101 is within the `192.168.1.0/24` HOME_NET range). With an external attacker IP, this would likely classify as CRITICAL. The MEDIUM classification is reasonable — it suggests investigation rather than immediate response, which is appropriate for an internal IP that could be authorized testing.

![Telegram Alerts](/assets/img/posts/argus-soc/phase-4/scenario-2/phase4_scenario2_telegram-cowrie-alerts.png){: width="700" }
_Telegram Alerts_

---

## Grafana Dashboard

The SOC dashboard shows the brute force impact clearly:

![Grafana dashboard — brute force scenario](/assets/img/posts/argus-soc/phase-4/scenario-2/phase4_scenario2_grafana-brute-force.png){: width="700" }
_Grafana SOC dashboard — brute force activity visible across all panels_

The MITRE ATT&CK Techniques panel shows Brute Force (5 hits), Valid Accounts (3), and Password Guessing (2) — correctly mapped from the custom Cowrie rules. The Recent Critical Alerts table shows both successful honeypot logins at rule level 10.

---

## What Worked

Everything. This is the first scenario with no detection gaps:

- **Cowrie captured the full session** — credentials, SSH fingerprint, timing, every event
- **Custom Wazuh rules fired correctly** — 100010 for connections, 100011 for logins
- **Pre-filter routed both paths** — level 6 filtered silently, level 10 passed to Claude
- **Claude classified correctly** — T1110.001, MEDIUM severity, plain-English summary including the actual credentials used
- **Telegram delivered within seconds** — operator received two formatted alerts
- **Grafana showed real-time impact** — MITRE techniques mapped, alert timeline visible
- **No remediation needed** — the detection stack worked as designed from the first attempt


---

## Pipeline Fix Applied During Scenario 2

The Log Event node crashed when receiving alerts from the post-Telegram path — the code referenced `$('Extract Alert Context')` which wasn't directly accessible in that execution branch. Fixed with a nested try/catch that handles all three input paths: pre-filter drops, severity router noise/low, and post-Telegram medium/critical. The updated code is documented in the [Phase 1 post](/posts/argus-soc-phase-1/).

---

## Evidence Collected

### Screenshots

| Screenshot | Filename |
|---|---|
| Hydra brute force output (Kali) | `phase4_scenario2_hydra-output.png` |
| Cowrie logs — full session capture | `phase4_scenario2_cowrie-logs.png` |
| Cowrie live log during attack | `phase4_scenario2_cowrie-logs-live.png` |
| n8n pipeline — Claude to Telegram | `phase4_scenario2_n8n-full-pipeline-claude-to-telegram.png` |
| Grafana dashboard — brute force impact | `phase4_scenario2_grafana-brute-force.png` |
| Wazuh alert list — rules 100010/100011 | `phase4_scenario2_wazuh-alert-list.png` |

### Data Exports

| File |  | Description |
|---|---|---|
| `scenario2_wazuh-events-export.csv` | [Download](/assets/downloads/argus-soc/phase-4/scenario-2/scenario2_wazuh-events-export.csv) | Wazuh events CSV covering the brute force window |

---

## What's Next — Scenario 3

Scenario 3 escalates to remote code execution: Metasploit against a vulnerable service on Pi 5. This is the scenario that initially produced zero detection — no Suricata alerts, no Wazuh alerts, nothing. The attacker got a root shell and the SOC was completely blind. The gap analysis and custom rule remediation that followed produced the most significant detection engineering of the entire project.

---

*Part of the [Argus SOC](https://github.com/Al3grus/Argus-SOC) build series.*