---
title: "Building Argus SOC  |  Phase 4 Scenario 4  |  Web Application Attacks — SQL Injection vs Invisible NIDS"
date: 2026-04-08 10:00:00 +0100
categories: [Projects, ARGUS-SOC]
tags: [soc, homelab, sqlmap, dvwa, sql-injection, detection-engineering, suricata, wazuh, claude, telegram, blue-team, red-team, mitre-attack]
description: "Phase 4 Scenario 4 of the Argus SOC build — SQLmap dumps the DVWA database through four injection techniques. Suricata sees nothing because HTTP_PORTS only covers port 80. After adding 8080 and writing custom rules, the same attack generates 33 alerts."
image:
  path: /assets/img/posts/argus-soc/banner.png
  alt: Argus SOC — AI-Augmented Home Lab Security Operations Center
---

> 📌 **Author's note** — This post documents the Argus SOC lab at the time of publication, when the Pi 3B+ served as the MSSP edge sensor and the Pi 5 hosted the vulnerable Docker targets. The architecture was redesigned in [Phase 5](/posts/argus-soc-phase-5/), which introduced a ThinkCentre M920x running Proxmox with an Active Directory lab as the client infrastructure, and moved the edge-sensor role onto the Pi 5. The detection logic, custom rules, and gap analysis described here remain valid; only the host topology has changed.
>
> Build carried out on real hardware in a controlled home lab. Claude (Anthropic) was used as a reasoning and writing assistant — all deployments, attacks, configurations, and verifications were performed by the author.

## Overview

This scenario tests whether the SOC can detect a web application attack against a known vulnerable target. SQLmap automates SQL injection against DVWA on port 8080, extracting database names through four different injection techniques in 16 seconds. The SOC sees nothing — not because the signatures don't exist, but because Suricata's `HTTP_PORTS` variable only includes port 80. Traffic on port 8080 never gets HTTP-layer inspection.

It's the same class of problem as Scenario 3's HOME_NET typo: a configuration variable silently scoping rules away from the traffic they're supposed to match.

---

## Preparing DVWA

DVWA requires an authenticated session with the security level set to Low before SQLmap can reach the injectable parameter. The session cookie was obtained programmatically from Kali:

```bash
# Get a fresh session cookie
COOKIE=$(curl -s -c - -d "username=admin&password=password&Login=Login" \
  "http://192.168.1.10:8080/login.php" | grep PHPSESSID | awk '{print $NF}')

# Set security to low
curl -s -b "PHPSESSID=$COOKIE" -d "security=low&seclev_submit=Submit" \
  "http://192.168.1.10:8080/security.php"
```

---

## The Attack

**Timestamp:** 2026-04-04 15:48 UTC
**Target:** 192.168.1.10:8080 (DVWA on Docker)

```
sqlmap -u "http://192.168.1.10:8080/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=<redacted-cookie>" \
  --dbs --batch
```

**Duration:** ~16 seconds
**HTTP requests sent:** 146

SQLmap identified four injection types on the `id` GET parameter:

| Injection Type | Payload Pattern |
|---|---|
| Boolean-based blind | `id=1' OR NOT 9661=9661#` |
| Error-based (EXTRACTVALUE) | `id=1' AND EXTRACTVALUE(1457,CONCAT(...))--` |
| Time-based blind (SLEEP) | `id=1' AND (SELECT 5407 FROM (SELECT(SLEEP(5)))zhhN)--` |
| UNION query (2 columns) | `id=1' UNION ALL SELECT CONCAT(...),NULL#` |

**Databases extracted:** `dvwa` and `information_schema`
**Back-end identified:** MySQL >= 5.1 (MariaDB fork), Apache 2.4.66, PHP 8.5.4

![SQLmap output](/assets/img/posts/argus-soc/phase-4/scenario-4/phase4_scenario4_sqlmap-output-1.png){: width="700" }
![SQLmap output](/assets/img/posts/argus-soc/phase-4/scenario-4/phase4_scenario4_sqlmap-output-2.png){: width="700" }
_SQLmap — four injection types identified, two databases extracted in 16 seconds_

---

## Detection Results: Zero Detection

### Suricata: No Alerts

The Suricata tail on Pi 3B+ was completely silent during the entire SQLmap run. 146 HTTP requests carrying SQL injection payloads — `UNION ALL SELECT`, `EXTRACTVALUE`, `SLEEP()`, `OR 1=1` — and not a single HTTP-layer signature fired.

### Wazuh: No Relevant Alerts

12 alerts during the window — all SSH brute force noise (`sshd: Attempt to login using a non-existent user`, rule 5710, level 5) on the Hetzner VPS from external internet IPs. Zero alerts from argus-edge-01 related to the web attack.

### Claude / Telegram: Never Triggered

No alerts reached n8n. The operator was never notified.

![Grafana showing zero detection](/assets/img/posts/argus-soc/phase-4/scenario-4/phase4_scenario4_grafana-zero-detection.png){: width="700" }
_Grafana SOC dashboard — no indication of an active SQL injection attack_

![Wazuh showing no SQLi alerts](/assets/img/posts/argus-soc/phase-4/scenario-4/phase4_scenario4_wazuh-no-sqli-alerts.png){: width="700" }
_Wazuh event list — only SSH noise from internet bots, nothing about the web attack_

---

## Root Cause Analysis

### Finding: HTTP_PORTS Set to Port 80 Only

```yaml
# suricata.yaml on Pi 3B+
HTTP_PORTS: "80"
```

DVWA runs on port 8080 (Docker). Suricata's HTTP inspection rules — including all ET web attack signatures — are scoped to `$HTTP_PORTS`. With only port 80 listed, Suricata never applied HTTP protocol parsing to traffic on 8080. The SQL injection payloads were visible in the raw packet data, but Suricata wasn't looking at them as HTTP.

This is the same pattern as the HOME_NET typo in Scenario 3 — a single variable silently breaks an entire category of detection.

### Contributing Factors

There are additional layers to this gap. ET web attack rules often scope SQL injection signatures to `$EXTERNAL_NET` source addresses, which means internal LAN traffic (192.168.1.101 → 192.168.1.10) may not match even with the correct ports. The payloads are also URL-encoded in GET parameters, requiring Suricata's HTTP decoder to be active on the port before the content matching can work. Without HTTP protocol parsing on 8080, none of this decoding happens.

---

## Remediation

### Fix 1 — Expanded HTTP_PORTS (Pi 3B+)

In `/etc/suricata/suricata.yaml`:

```yaml
# Before
HTTP_PORTS: "80"

# After
HTTP_PORTS: "[80,8080,8888,3000]"
```

Port 8080 covers DVWA. Port 8888 covers MediaMTX on Pi 5. Port 3000 covers Grafana. Any service running HTTP on a non-standard port needs to be listed here or Suricata won't inspect it.

### Fix 2 — Custom Suricata SQL Injection Rules (Pi 3B+)

Added to `/etc/suricata/rules/local.rules`:

```
# SQL injection: UNION SELECT in URI
alert http any any -> $HOME_NET $HTTP_PORTS (
  msg:"LOCAL SQL Injection attempt - UNION SELECT in URI";
  flow:established,to_server;
  http.uri;
  content:"UNION"; nocase;
  content:"SELECT"; nocase;
  classtype:web-application-attack;
  sid:9000010; rev:1;
)

# SQL injection: time-based blind (SLEEP)
alert http any any -> $HOME_NET $HTTP_PORTS (
  msg:"LOCAL SQL Injection attempt - time-based blind in URI";
  flow:established,to_server;
  http.uri;
  content:"SLEEP"; nocase;
  classtype:web-application-attack;
  sid:9000011; rev:1;
)
```

These rules use `http.uri` — a sticky buffer that operates on the decoded HTTP URI after Suricata's HTTP parser processes the request. This means URL-encoded payloads like `%27%20UNION%20SELECT` are decoded before matching. The rules use `any` as the source address instead of `$EXTERNAL_NET`, so they catch internal attackers too.

### Fix 3 — Wazuh Correlation Rule (Hetzner)

Added to `/var/ossec/etc/rules/local_rules.xml`:

```xml
<rule id="100032" level="12">
  <if_sid>86601</if_sid>
  <match>LOCAL SQL Injection</match>
  <description>Suricata: SQL injection attempt detected from $(data.srcip)</description>
  <mitre><id>T1190</id></mitre>
</rule>
```

Level 12 ensures the alert passes the n8n pre-filter and reaches Claude for classification.

---

## Re-Test Results: Full Detection

**Timestamp:** 2026-04-04 19:55 UTC
**Same target, same technique, fresh SQLmap session.**

The SQLmap cache was cleared first to force a full re-scan with all injection probes:

```bash
rm -rf /home/Al3grus/.local/share/sqlmap/output/192.168.1.10

sqlmap -u "http://192.168.1.10:8080/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=$COOKIE;security=low" \
  --dbs --batch --flush-session
```

### Wazuh Alert Escalation

| Rule ID | Description | Level | Count |
|---|---|---|---|
| 100032 | SQL injection attempt detected | 12 | 33 |

33 alerts — every UNION SELECT and SLEEP probe matched.

### Claude AI Classification

Claude classified the alerts as MEDIUM with confidence 0.90+:

- **T1190** (Exploit Public-Facing Application)
- **Action:** investigate
- **Summary:** "SQLmap SQL injection attack detected against argus-central (192.168.1.10:8080) from internal IP 192.168.1.101."

Claude identified SQLmap by name from the payload patterns — the URI structure, injection ordering, and comment markers are distinctive enough for automated tool identification.

### Telegram: Alerts Delivered

The operator received Telegram alerts with Claude's classification identifying the attack tool, target, and MITRE technique.

![Wazuh showing 33 SQL injection alerts](/assets/img/posts/argus-soc/phase-4/scenario-4/phase4_scenario4_remediated-wazuh-sqli-alerts.png){: width="700" }
_Wazuh after remediation — 33 SQL injection alerts, all rule 100032 at level 12_

![Grafana showing 33 critical alerts](/assets/img/posts/argus-soc/phase-4/scenario-4/phase4_scenario4_remediated-grafana-33-critical.png){: width="700" }
_Grafana — MITRE ATT&CK panel showing "Exploit Public-Facing Application" at 33 hits_

![SQLmap remediated output](/assets/img/posts/argus-soc/phase-4/scenario-4/phase4_scenario4_remediated-sqlmap-output.png){: width="700" }
_SQLmap re-run — same four injection types, same databases extracted_

![Telegram SQL injection alerts](/assets/img/posts/argus-soc/phase-4/scenario-4/phase4_scenario4_remediated-telegram-sqli.png){: width="700" }
_Telegram — Claude classifies as MEDIUM, identifies SQLmap by name_

---

## Before / After

| Metric | Before | After |
|---|---|---|
| Suricata alerts | 0 | 33 |
| Wazuh alerts | 0 | 33 (all level 12) |
| Claude classification | Never reached | MEDIUM, T1190 |
| Telegram alerts | 0 | Multiple |
| SQLmap identified by name | No | Yes |
| Time to operator notification | Never | ~5 seconds |

---

## NIDS vs WAF: What This Scenario Reveals

This scenario highlights an important architectural point. Suricata is a network intrusion detection system — it matches patterns in network traffic. It's not a web application firewall. Even with the correct ports and custom rules, it can only detect SQL injection patterns it has signatures for. An attacker using obfuscated payloads, encoding tricks, or novel injection techniques would bypass these rules.

In a production environment, web application attacks are better handled at the application layer — mod_security, cloud WAFs, or runtime application self-protection (RASP). Suricata provides a network-level safety net that catches automated tools like SQLmap, but it's not a substitute for application-layer security.

---

## Progression Across Scenarios

| Aspect | Scenario 1 | Scenario 2 | Scenario 3 | Scenario 4 |
|---|---|---|---|---|
| Initial detection | Partial | Full | Zero | Zero |
| Root cause | HOME_NET typo | N/A | HOME_NET + missing sigs | HTTP_PORTS + missing sigs |
| Custom rules | 0 | 0 | 2 Suricata + 2 Wazuh | 2 Suricata + 1 Wazuh |
| After remediation | ET SCAN firing | N/A | 12 CRITICAL | 33 alerts |
| Claude reached | No (level 3) | Yes — MEDIUM | Yes — CRITICAL | Yes — MEDIUM |
| Telegram | No | Yes | Yes (11 alerts) | Yes |

---

## Guest WiFi Isolation — Confirmed

During this scenario, an attempt to run SQLmap from the guest WiFi confirmed that the TP-Link Archer AX55 guest network provides genuine client isolation. The Kali VM could not reach `192.168.1.0/24` from the guest SSID. All attack scenarios were executed from the main WiFi where the attacker shares the same subnet as the targets.

---

## Evidence Collected

### Screenshots — Initial Attack (Zero Detection)

| Screenshot | Filename |
|---|---|
| SQLmap output — 4 injection types | `phase4_scenario4_sqlmap-output.png` |
| Grafana — zero detection | `phase4_scenario4_grafana-zero-detection.png` |
| Wazuh — no SQLi alerts | `phase4_scenario4_wazuh-no-sqli-alerts.png` |

### Screenshots — After Remediation

| Screenshot | Filename |
|---|---|
| Wazuh — 33 SQL injection alerts | `phase4_scenario4_remediated-wazuh-sqli-alerts.png` |
| Grafana — 33 critical, T1190 | `phase4_scenario4_remediated-grafana-33-critical.png` |
| SQLmap output (fresh run) | `phase4_scenario4_remediated-sqlmap-output.png` |
| Telegram — Claude SQLi classification | `phase4_scenario4_remediated-telegram-sqli.png` |

### Data Exports

| File | Description |
|---|---|
| `scenario4_wazuh-events-export.csv` | [Download](/assets/downloads/argus-soc/phase-4/scenario-4/scenario4_wazuh-events-export.csv) — zero detection run |
| `scenario4_remediated-wazuh-events-export.csv` | [Download](/assets/downloads/argus-soc/phase-4/scenario-4/scenario4_remediated-wazuh-events-export.csv) — successful detection run |

---

## What's Next — Scenario 5

Scenario 5 is the final attack — lateral movement from a compromised Metasploitable container to the Cowrie honeypot on Pi 3B+. Unlike the previous scenarios where the problem was missing detection, Scenario 5 reveals a different gap: the alert fires but Claude classifies it as LOW because it doesn't recognize the significance of a server IP appearing as an SSH source.

---

*Part of the [Argus SOC](https://github.com/Al3grus/Argus-SOC) build series.*