---
title: "Building Argus SOC  |  Phase 4 Scenario 5  |  Lateral Movement — When the AI Got It Wrong"
date: 2026-04-09 10:00:00 +0100
categories: [Projects, ARGUS-SOC]
tags: [soc, homelab, lateral-movement, cowrie, honeypot, detection-engineering, wazuh, claude, telegram, blue-team, red-team, mitre-attack]
description: "Phase 4 Scenario 5 of the Argus SOC build — pivoting from a compromised Metasploitable host into the Cowrie honeypot. The detection rules fire correctly, but Claude misclassifies the alert as LOW. The first scenario where the AI triage layer — not the detection layer — was the weak link."
image:
  path: /assets/img/posts/argus-soc/banner.png
  alt: Argus SOC — AI-Augmented Home Lab Security Operations Center
---

> 📌 **Author's note** — This post documents the Argus SOC lab at the time of publication, when the Pi 3B+ served as the MSSP edge sensor and the Pi 5 hosted the vulnerable Docker targets. The architecture was redesigned in [Phase 5](/posts/argus-soc-phase-5/), which introduced a ThinkCentre M920x running Proxmox with an Active Directory lab as the client infrastructure, and moved the edge-sensor role onto the Pi 5. The detection logic, custom rules, and gap analysis described here remain valid; only the host topology has changed.
>
> Build carried out on real hardware in a controlled home lab. Claude (Anthropic) was used as a reasoning and writing assistant — all deployments, attacks, configurations, and verifications were performed by the author.

## Overview

This is the final scenario of Phase 4 and the most interesting one analytically. The previous four scenarios all followed the same broad pattern: a configuration variable or missing signature caused Suricata or Wazuh to miss the attack, and remediation came down to fixing rules. Scenario 5 breaks that pattern. The detection rules fire correctly. The pipeline routes the alert to Claude. Claude under-classifies it as LOW, the alert never reaches Telegram, and the operator is never notified.

This is the first scenario where the AI triage layer — not the detection layer — was the weak link. The remediation isn't a Suricata signature or a HOME_NET fix; it's a Wazuh correlation rule that escalates the alert *before* it reaches Claude, giving the AI the right signal to act on.

---

## The Attack — Pivoting Through a Compromised Host

**Timestamp:** 2026-04-04 16:07 UTC
**Attack path:** Kali (192.168.1.101) → Metasploitable 2 ingreslock backdoor (192.168.1.10:1524) → SSH to Pi 3B+ Cowrie honeypot (192.168.1.20:22)

This scenario simulates a realistic lateral movement chain. The attacker doesn't directly attack the edge sensor — they compromise the client infrastructure first, then pivot through it to reach the next target. From the SOC's perspective, the source IP of the second-stage attack is an internal host, not the original attacker.

### Kill Chain

```bash
# Stage 1: From Kali — access compromised host via ingreslock backdoor
nc 192.168.1.10 1524

# Stage 2: From compromised Metasploitable — pivot to Pi 3B+
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@192.168.1.20

# Stage 3: Inside Cowrie fake shell — post-exploitation recon
whoami          → root
ls -la          → directory listing of /root
id              → uid=0(root) gid=0(root) groups=0(root)
cat /etc/passwd → full user list
```

![Kali pivoting from Metasploitable to Cowrie](/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_lateral-movement-kali.png){: width="700" }
_Kali terminal — nc to ingreslock, then SSH from inside the compromised host to the honeypot_

---

## Detection Results — Detection Works, Triage Fails

### Cowrie Honeypot — Pivot Detected ✅

Cowrie recorded the lateral movement with the correct source IP — **192.168.1.10** (Pi 5), not 192.168.1.101 (Kali):

| Rule ID | Description | Level | Source IP |
|---|---|---|---|
| 100010 | New honeypot connection from 192.168.1.10 | 6 | 192.168.1.10 |
| 100011 | Successful honeypot login from 192.168.1.10 | 10 | 192.168.1.10 |
| 100012 | Attacker command executed on honeypot | 6 | — (5 hits) |

This is exactly the signal a SOC analyst would want. A server IP appearing as the source of an SSH login is the smoking gun: servers don't normally open interactive SSH sessions to other hosts. The detection layer captured this correctly.

### Wazuh — 11 Alerts Total

| Rule ID | Description | Level | Count |
|---|---|---|---|
| 100012 | Cowrie: Attacker command executed on honeypot | 6 | 5 |
| 5710 | sshd: Attempt to login using non-existent user | 5 | 4 |
| 100011 | Cowrie: Successful honeypot login from 192.168.1.10 | 10 | 1 |
| 100010 | Cowrie: New honeypot connection from 192.168.1.10 | 6 | 1 |

Rule 100011 fired at level 10, passed the n8n pre-filter, and reached Claude. So far, the pipeline worked as designed.

### Claude AI Triage — Classified as LOW ❌

Claude classified the level-10 Cowrie successful login as **LOW** severity. The alert was routed to silent logging, not Telegram. The operator was never notified.

**The comparison point that makes this jump out:** In [Scenario 2](/posts/argus-soc-phase-4-scenario-2/), the same Cowrie login event — same rule ID, same level, same alert structure — coming from 192.168.1.101 (Kali) was classified MEDIUM and triggered Telegram. Coming from 192.168.1.10 (the server), the same event was classified LOW.

Claude treated the internal server source as *less* suspicious. In reality it's the opposite: a server-to-server interactive SSH login is more anomalous than a workstation-to-server one, because workstations open SSH sessions all the time and servers usually don't. The AI had the right data but lacked the topological context to interpret it correctly.

### Telegram: Silent

No notification. The operator had no idea lateral movement was in progress.

![Wazuh showing Cowrie alerts from Metasploitable source IP](/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_wazuh-cowrie-from-metasploitable.png){: width="700" }
_Wazuh event list — rule 100011 fires at level 10, source IP 192.168.1.10 (Pi 5), but downstream classification missed the significance_

![Grafana showing lateral movement detection](/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_grafana-lateral-detection.png){: width="700" }
_Grafana — the honeypot login appears as a single critical alert, but no escalation, no operator notification_

![n8n showing alert routed as LOW](/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_n8n-low-severity-routed.png){: width="700" }
_n8n workflow — the alert reached Claude, was classified LOW, and routed to silent logging instead of Telegram_

---

## Root Cause — A Different Kind of Gap

The previous four scenarios all came down to detection-layer problems:

- Scenario 1 — `HOME_NET` typo scoped ET SCAN rules away from real traffic
- Scenario 3 — same `HOME_NET` typo, plus no signature for the UnrealIRCd backdoor
- Scenario 4 — `HTTP_PORTS` only covered port 80, missed traffic on 8080

In each case, the fix was to add or correct a rule so the detection engine could see the attack.

**Scenario 5 is different.** The detection engine saw the attack. Cowrie logged the right source IP. Wazuh correlated the events and fired the right rule. The pipeline carried the alert to the right place. The failure happened *after* detection, inside the AI triage layer.

Claude's classification depended on context it didn't have. From the system prompt's perspective, 192.168.1.10 was just "an internal IP" — there was nothing to tell the AI that this specific IP belonged to a server, that servers SSH'ing into other servers is anomalous, or that the broader pattern (server appearing as SSH source) was the actual indicator of compromise.

The fix isn't a Suricata signature. It's a Wazuh rule that recognises the anomaly *before* the alert reaches Claude, and escalates the severity so Claude is working from a stronger signal.

### What Was and Wasn't Detected

| Detected | Not Detected |
|---|---|
| Cowrie SSH login from compromised host | The initial ingreslock backdoor access on port 1524 |
| Post-exploitation commands in the fake shell | The pivot chain itself — no rule correlates "host A compromised → host A now SSH'ing to host B" |
| Correct source IP attribution to Pi 5 | The operator notification — Telegram never fired |
| Level 10 alert reached Claude | Claude's prompt didn't recognise server-IP-as-SSH-source as a strong indicator |

A Wazuh rule that escalates server-originated honeypot logins addresses the most actionable of these gaps without requiring a Claude prompt rewrite.

---

## Remediation

### Custom Wazuh Correlation Rule (Hetzner)

Added to `/var/ossec/etc/rules/local_rules.xml`:

```xml
<rule id="100033" level="12">
  <if_sid>100011</if_sid>
  <match>192.168.1.10</match>
  <description>Cowrie: Lateral movement — honeypot login from server IP 192.168.1.10 (possible pivot from compromised host)</description>
  <mitre><id>T1021</id></mitre>
</rule>
```

This rule chains off Cowrie successful logins (rule 100011) and matches when the alert description contains the server IP. When matched, it escalates the alert to level 12 and explicitly maps to **MITRE T1021 (Remote Services)** — correctly identifying this as lateral movement rather than the generic credential access classification that triggered for Scenario 2.

### A Subtle Wazuh Decoder Note

An initial attempt using `<srcip>192.168.1.10</srcip>` failed silently — the alert wouldn't escalate. The Cowrie decoder doesn't populate the standard `srcip` field the same way native sshd alerts do; the source IP only appears in the description text. Switching to `<match>` against the description string resolved this.

This is the kind of gap that's invisible until you test the rule end-to-end. The rule looks correct on paper, doesn't error, doesn't log a warning — it just doesn't fire. Lesson for the future: always verify a new Wazuh correlation rule by triggering the underlying condition and watching the manager logs in real time.

---

## Re-Test Results — Full Pipeline Recovered

{% raw %}
<video controls autoplay muted loop playsinline width="700" style="max-width: 100%; display: block; margin: 1em auto;">
  <source src="/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_full-pipeline-demo.mp4" type="video/mp4">
  Your browser doesn't support the video tag — <a href="/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_full-pipeline-demo.mp4">download the video</a> instead.
</video>
{% endraw %}

_Full pipeline in ~30 seconds — Kali launches the attack, Suricata detects, Wazuh correlates, Claude classifies CRITICAL, Telegram fires._

**Timestamp:** 2026-04-04 20:37 UTC
**Same attack path:** Kali → nc 192.168.1.10:1524 → ssh root@192.168.1.20

### Claude AI Classification (After Remediation)

- **Severity:** CRITICAL
- **MITRE:** T1021 (Remote Services)
- **Confidence:** 0.92
- **Action:** respond_immediately
- **Summary:** "A successful SSH login to the Cowrie honeypot on argus-edge-01 originated from argus-central (192.168.1.10), indicating potential lateral movement from a compromised internal host. The root account login with empty password to port 22 (honeypot) suggests an attacker has pivoted from the central server and is actively probing the network."

The same event, from the same source IP, generated by the same Cowrie rule — but now classified correctly because the new Wazuh rule presented it to Claude as a level-12 lateral movement event mapped to T1021, rather than a level-10 generic credential access event.

### Telegram: CRITICAL Alert Delivered

The operator received a CRITICAL Telegram alert within seconds, with Claude's classification correctly identifying lateral movement and recommending immediate response.

![Kali re-running the pivot](/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_remediated-kali-pivot.png){: width="700" }
_Re-test — same attack, full pipeline now responds_

![Wazuh rule 100033 firing at level 12](/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_remediated-wazuh-rule-100033.png){: width="700" }
_Wazuh after remediation — rule 100033 fires at level 12, MITRE T1021 mapped_

![Grafana showing lateral movement detection](/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_remediated-grafana-lateral.png){: width="700" }
_Grafana — MITRE ATT&CK panel showing Remote Services classification_

![n8n showing alert routed as CRITICAL](/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_n8n-critical-severity-routed.png){: width="700" }
_n8n workflow after remediation — same alert path, now routed as CRITICAL to Telegram and PagerDuty_

![Telegram CRITICAL alert for lateral movement](/assets/img/posts/argus-soc/phase-4/scenario-5/phase4_scenario5_remediated-telegram-lateral-critical.png){: width="700" }
_Telegram — CRITICAL alert delivered, Claude correctly identifies lateral movement and recommends immediate response_

---

## Before / After

| Metric | Before | After |
|---|---|---|
| Wazuh rule fired | 100011 (level 10) | 100033 (level 12) |
| Claude classification | LOW | **CRITICAL** |
| Telegram alert | No | **Yes** |
| MITRE technique | T1110.001 (Password Guessing) | **T1021 (Remote Services)** |
| Recommended action | — | **respond_immediately** |
| Time to operator notification | Never | ~5 seconds |

---

## Phase 4 Closing — Progression Across All Five Scenarios

| Aspect | Scenario 1 (Recon) | Scenario 2 (Brute Force) | Scenario 3 (RCE) | Scenario 4 (SQLi) | Scenario 5 (Lateral) |
|---|---|---|---|---|---|
| Initial detection | Partial | Full | Zero | Zero | Partial (LOW) |
| Root cause | HOME_NET typo | N/A | HOME_NET + missing sigs | HTTP_PORTS + missing sigs | **AI triage under-classification** |
| Layer at fault | Detection (Suricata) | None | Detection (Suricata + Wazuh) | Detection (Suricata) | **Triage (Claude prompt context)** |
| Custom rules | 0 | 0 | 2 Suricata + 2 Wazuh | 2 Suricata + 1 Wazuh | **1 Wazuh** |
| After remediation | ET SCAN firing | N/A | 12 CRITICAL | 33 alerts | **CRITICAL, T1021 mapped** |
| Claude reached | No (level 3) | Yes — MEDIUM | Yes — CRITICAL | Yes — MEDIUM | Yes — CRITICAL |
| Telegram | No | Yes | Yes (11 alerts) | Yes | **Yes** |

Across the five scenarios, the detection stack started near-blind (HOME_NET miscalibration retroactively explained every Phase 0–3 gap), evolved through specific signature work (custom Suricata SIDs for UnrealIRCd, reverse shells, SQL injection), and ended with a different class of fix entirely — using Wazuh correlation to give the AI triage layer better context to work with.

That last point is the one worth holding onto. AI-augmented SOC pipelines aren't magic detection layers. They classify what they're shown, and the quality of the classification depends entirely on the quality of the signal they receive. Tuning the AI is real work, and most of that work happens *upstream* of the AI — in the detection rules, correlation logic, and severity levels that prepare each alert for triage.

---

## Architectural Note — Guest WiFi Isolation

During Phase 4, an attempt to run attacks from the TP-Link Archer AX55 guest WiFi confirmed full client isolation — the Kali VM could not reach `192.168.1.0/24`. All attack scenarios were executed from the main WiFi subnet. In a real MSSP deployment the external attacker would arrive from the internet, not a local network segment. The detection results remain valid since Suricata sees all traffic via SPAN regardless of the attacker's network position.

---

## Evidence Collected

### Screenshots — Initial Attack (Under-Classified)

| Screenshot | Filename |
|---|---|
| Kali — nc pivot + Cowrie shell | `phase4_scenario5_lateral-movement-kali.png` |
| Grafana — lateral movement detection | `phase4_scenario5_grafana-lateral-detection.png` |
| Wazuh — Cowrie alerts from 192.168.1.10 | `phase4_scenario5_wazuh-cowrie-from-metasploitable.png` |

### Screenshots — After Remediation

| Screenshot | Filename |
|---|---|
| Telegram — CRITICAL lateral movement alert | `phase4_scenario5_remediated-telegram-lateral-critical.png` |
| Wazuh — rule 100033 at level 12 | `phase4_scenario5_remediated-wazuh-rule-100033.png` |
| Grafana — lateral movement detected | `phase4_scenario5_remediated-grafana-lateral.png` |
| Kali — re-run pivot | `phase4_scenario5_remediated-kali-pivot.png` |

### Data Exports

| File | Description |
|---|---|
| `scenario5_wazuh-events-export.csv` | [Download](/assets/downloads/argus-soc/phase-4/scenario-5/scenario5_wazuh-events-export.csv) — Cowrie detecting pivot from Metasploitable IP, under-classified run |
| `scenario5_remediated-wazuh-events-export.csv` | [Download](/assets/downloads/argus-soc/phase-4/scenario-5/scenario5_remediated-wazuh-events-export.csv) — successful detection and CRITICAL escalation after remediation |

---

## What's Next — Phase 5

Phase 4 closes here. Five scenarios across the kill chain, every detection gap documented honestly, every remediation tested end-to-end, every layer of the pipeline exercised — including the AI itself.

[Phase 5](/posts/argus-soc-phase-5/) is the architectural pivot. Hardening across all nodes, then a full migration: the Pi 3B+ is retired from its edge-sensor role, a ThinkCentre M920x running Proxmox joins the lab as the new client infrastructure, an Active Directory domain comes online (Windows Server 2022 DC + Windows 11 domain-joined workstation), and the Pi 5 takes over as the MSSP edge sensor monitoring the new environment. Same MSSP topology, more realistic client side, ready for Phase 7's AD-specific attack scenarios.

---

*Part of the [Argus SOC](https://github.com/Al3grus/Argus-SOC) build series.*