---
title: "Building Argus SOC  |  Phase 5  |  Hardening, Architecture Migration, and the Active Directory Lab"
date: 2026-04-15 10:00:00 +0100
categories: [Projects, ARGUS-SOC]
tags: [soc, homelab, proxmox, active-directory, windows-server, wazuh, hardening, architecture, blue-team, security-engineering]
description: "Phase 5 of the Argus SOC build — hardening across all nodes, retiring the Pi 3B+, bringing a ThinkCentre M920x online as the new client infrastructure with Proxmox and an Active Directory domain, migrating the edge-sensor role to the Pi 5, and tuning the AI triage layer for the new environment."
image:
  path: /assets/img/posts/argus-soc/banner.png
  alt: Argus SOC — AI-Augmented Home Lab Security Operations Center
---

## Overview

Phase 4 closed with five scenarios run against a Pi-3B+-as-edge-sensor topology where the targets were Docker containers on the Pi 5. The detection stack worked, the AI triage layer was tuned, the kill chain was fully exercised. But the architecture had reached its limit: a 1GB Pi 3B+ couldn't run Zeek alongside Suricata without memory pressure, the targets were Linux-only, and there was no realistic enterprise context — no domain, no Kerberos, no Group Policy, no Windows. The next phase of the project needs Active Directory.

Phase 5 is the rebuild. The Pi 3B+ is retired. A ThinkCentre M920x joins the lab running Proxmox as the new client infrastructure, hosting a small AD domain (Windows Server 2022 DC + Windows 11 domain-joined workstation) and the vulnerable targets. The Pi 5 takes over the edge-sensor role with full Suricata + Zeek + Cowrie + Wazuh agent + Velociraptor stack on its own hardware budget. The MSSP topology is preserved end to end — cloud SOC platform monitors a separate client network through an edge sensor — but the client side is now realistic enough for Phase 6's AD-specific attack scenarios.

This post covers three things in order: hardening of the existing nodes that came out of Phase 4, the architecture migration itself, and the AI triage tuning that fell out of operating the new environment for a few days.

---

## Hardening

Before any new hardware came online, the Phase 4 setup needed to be production-grade. Five scenarios of red team activity had left a clear picture of where the lab was loose.

### Hetzner Cloud SOC Platform

The Hetzner VPS hosts Wazuh, n8n, Velociraptor, and the Claude API integration — it's the single most valuable piece of infrastructure in the entire project. The hardening pass focused on three things: SSH access control, UFW firewall lockdown, and a real lockout-recovery story.

SSH was restricted to a non-root admin account (`argus-admin`), root login disabled, password authentication off, key-only with `MaxAuthTries 3`, and `AllowUsers argus-admin` to make any other attempted username a guaranteed reject. fail2ban was configured to ban brute force attempts permanently with a 24-hour escalation window.

The firewall was locked down so that every Wazuh and management UI is tunnel-only — the only ports exposed to the internet are SSH and the Wazuh agent listener (1514). The Wazuh dashboard, n8n, Velociraptor, and Kibana are all reached over an SSH tunnel from the operator workstation. This is the same shape a real MSSP would deploy: nothing customer-facing on the SOC platform other than what agents and operators need.

The lockout recovery deserves a paragraph of its own. During fail2ban tuning, the operator's own home IP got auto-banned after a series of misconfigured SSH attempts during the lockdown pass — exactly the failure mode fail2ban is supposed to prevent against attackers, applied accidentally to the legitimate user. The fix was twofold: temporarily unban via `fail2ban-client unban`, and then add a permanent `ignoreip` whitelist for the home public IP to prevent recurrence. Lesson worth holding onto: any hardening change involving rate-limiting or banning needs an operator-IP whitelist *before* the change is committed, not after the lockout.

### Pi 5

The Pi 5 was hardened before being repurposed as the edge sensor. DNS-over-HTTPS was added via `dnscrypt-proxy` upstream of Pi-hole — queries leaving the network are now encrypted to Cloudflare rather than plaintext on the local resolver path. Pi-hole and Grafana were restricted to the WireGuard interface only: no admin UI exposed to the LAN, even from the operator workstation. SSH on port 2222, key-only, same admin pattern as Hetzner.

### Pi 3B+ (Outgoing)

The Pi 3B+ got a final tuning pass before retirement. Suricata log rotation was tightened, the stats output (`stats.log`) was disabled since it was eating memory without contributing to detection, and Zeek was permanently disabled — the 1GB RAM budget couldn't carry both engines without sustained pressure. Bluetooth and Avahi were removed entirely, and unattended-upgrades was enabled so the box would stay patched without manual intervention through the end-of-Phase-4 cleanup window.

### Router

The router (TP-Link Archer AX55) was hardened directly. UPnP off, WPS off, remote management off, firmware updated to the latest patch. Guest WiFi confirmed isolated from the LAN subnet (this was the same configuration used during Phase 4 to verify attacker isolation). Standard home-network hygiene, applied deliberately.

---

## The Architecture Migration

### Why Migrate

The migration wasn't about chasing better hardware — it was about reaching a topology where Active Directory attack scenarios make sense. The Phase 4 architecture had three structural limitations:

1. **No Windows.** Metasploitable 2 and DVWA are useful but Linux-only. Kerberoasting, AS-REP roasting, DCSync, Pass-the-Hash, BloodHound enumeration — none of these have a target on a Linux-only network.
2. **Pi 3B+ at its memory ceiling.** Running Suricata with the ET Open ruleset on 1GB RAM forced Zeek to be disabled, which meant losing protocol metadata that complements signature-based detection.
3. **Implausible client side.** A "client" with no domain, no GPOs, no workstations, and no realistic services doesn't reflect what an MSSP actually monitors.

A second-hand ThinkCentre M920x with an i7-8700 and 32GB DDR4, found on Vinted at €280, solved all three at once. The form factor is small enough to live next to the existing rack, the CPU and memory are enough for a four-VM domain, and Proxmox handles everything the lab needs.

### Proxmox Onboarding

Proxmox VE 9.1.7 installed cleanly on the internal NVMe. The standard post-install steps — disabling the enterprise repositories (which require a paid subscription), enabling the no-subscription repository, and running a full `dist-upgrade` — needed one wrinkle: Proxmox 9 uses the newer `.sources` format for repository definitions, not the older `.list` format, so the usual `sed` recipes from older guides don't apply. Adding `Enabled: no` to `/etc/apt/sources.list.d/pve-enterprise.sources` and `ceph.sources` did the job. After that, `apt update` ran clean.

A second NVMe drive went in as VM storage, mounted at `/mnt/vm-storage` and registered with Proxmox as a Directory storage backend. ISOs for Windows Server 2022 Evaluation and Windows 11 Enterprise Evaluation were uploaded, along with the VirtIO driver ISO needed during Windows install.

SSH was hardened on the Proxmox host itself: port moved to 2222, the Proxmox built-in firewall enabled at both Datacenter and Node levels, and rules added to allow SSH and the Proxmox web UI (8006) only from the LAN subnet. One configuration gotcha: the firewall rule editor defaults to setting ports as *source port* rather than *destination port*, which silently blocks all connections. The fix is obvious once seen, but easy to miss while the rule looks correct at a glance.

### DC01 — Windows Server 2022 Domain Controller

VM 101, 2 cores, 4GB RAM, 60GB disk on the VM storage, q35 chipset with OVMF UEFI and TPM 2.0 (Windows Server 2022 requires both). The VirtIO SCSI driver had to be loaded from the VirtIO ISO during Windows setup — Windows doesn't ship with VirtIO drivers natively. After install, the VirtIO network driver was installed via Device Manager from the same ISO.

Static IP: 192.168.1.31, gateway 192.168.1.1, DNS 127.0.0.1 (the DC will resolve for itself once promoted). Hostname renamed to DC01. AD DS role installed:

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

Domain promotion:

```powershell
Install-ADDSForest -DomainName "argus.local" -DomainNetbiosName "ARGUS" -InstallDns -Force
```

The DNS delegation warnings during promotion are expected and harmless for an internal `.local` domain — they would matter for a real public-DNS-integrated forest, but in a lab they're noise. After reboot, the login screen showed `ARGUS\Administrator` and the domain was operational.

DNS forwarders were added pointing at Pi-hole (192.168.1.10) and Cloudflare (1.1.1.1) so DC01 can resolve external domains while keeping internal `argus.local` records authoritative locally.

### WS01 — Windows 11 Enterprise, Domain-Joined

VM 102, 2 cores, 4GB RAM, 56GB disk (Windows 11 enforces a 52GB minimum). Same q35 + OVMF + TPM 2.0 setup. The driver gotcha here is different from DC01: Windows 11 in this configuration uses VirtIO Block (not VirtIO SCSI) for the disk, so the install driver loads from `viostor\w11\amd64` rather than `vioscsi\w11\amd64`. The Windows installer doesn't tell you this — it just shows "no compatible drivers" if you pick the wrong directory. The kind of thing you discover by trying both.

Static IP 192.168.1.32, DNS pointing at DC01 (192.168.1.31) so domain queries resolve correctly. Renamed to WS01, joined to `argus.local`, moved into the right OU after the join.

### OU Structure, Users, and Service Accounts

The directory was built out with a structure that mirrors a small enterprise — IT, Finance, HR, plus a Service Accounts OU:

```
argus.local
├── Argus Corp
│   ├── IT Department (Users + Computers)
│   ├── Finance (Users + Computers)
│   ├── HR (Users + Computers)
│   └── Service Accounts
└── Domain Controllers
```

Five user accounts across IT, Finance, and HR — a Domain Admin, a Server Operator, and three standard users. Three service accounts in the Service Accounts OU, two of which (`svc_sql` and `svc_web`) have SPNs registered for Kerberoasting practice in Phase 6:

| Account | SPN | Purpose |
|---|---|---|
| svc_sql | MSSQLSvc/dc01.argus.local:1433 | Kerberoasting target |
| svc_backup | (none) | Backup service |
| svc_web | HTTP/dc01.argus.local | Kerberoasting target |

### The Audit Policy Story

This is where the build hit its most informative friction. The plan was to use GPO to configure Advanced Audit Policy settings across the domain — Account Logon, Logon/Logoff, Object Access, Process Creation, Privilege Use, the standard set that a SOC needs to see. The GPO was created (`Argus SOC - Audit Policy`), linked to the domain, populated with all the right settings.

It didn't work. `auditpol /get /category:*` on the DC showed "No Auditing" for everything. `gpupdate /force` made no difference.

The fix that's documented in most Microsoft articles — enabling "Audit: Force audit policy subcategory settings to override audit policy category settings" in the Default Domain Policy — was applied. Still no effect.

The eventual workaround was to apply audit policies directly via `auditpol /set /subcategory:` commands for all 14 subcategories, set to Success and Failure. This bypassed GPO entirely and worked immediately. After application, `auditpol /get /category:*` showed all 14 subcategories logging both Success and Failure, and the security event flow started: Event ID 4624 (logon), 4634 (logoff), 4672 (special logon), 4688 (process creation with command line).

The "why GPO failed" question is unresolved. The most likely explanation is GPO replication latency in a single-DC fresh-build domain interacting with the way Windows caches the Local Security Policy until reboot — but the audit settings hadn't applied even after multiple reboots and `gpupdate /force` cycles. Worth circling back to in Phase 6 once the domain has been running longer, but for now, the direct `auditpol` application is correct and gets the SOC the visibility it needs. Command-line logging (the `Audit Process Creation` policy with the "Include command line" option) and PowerShell logging (Module + Script Block + Transcription) were also configured.

### Vulnerable Targets

Two attack surfaces from Phase 4 were carried forward, but moved from the Pi 5 onto Proxmox:

- **Metasploitable 2** — VM 103, imported from VMDK, static IP 192.168.1.33. Same image used in Phase 4 scenarios, now properly virtualised.
- **DVWA** — CT 104, deployed as a Debian 12 LXC container (lighter than a full VM, accessible at `http://192.168.1.34/dvwa`).

Both are now reachable from the same lab subnet as DC01 and WS01, which means future scenarios can chain Linux exploitation into AD attacks naturally.

---

## Migrating the Edge Sensor — Pi 3B+ to Pi 5

This was the most operationally delicate part of the migration. The detection layer needed to keep running through the transition — the goal was a clean handover, not an outage.

### Configuration Backup

Before touching anything physical, the Pi 3B+ configs were backed up to the Pi 5: Suricata config and rules (including the four custom Phase 4 SIDs), Zeek `node.cfg` and `local.zeek`, and the Wazuh `ossec.conf`. Cowrie's config wasn't in the backup so it was reinstalled fresh on the Pi 5 — no real loss, the Phase 4 Cowrie deployment was close to default anyway.

The backups went over SCP through the WireGuard tunnel, because the Pi 3B+'s SSH was only reachable from the WireGuard subnet by that point. This is the kind of thing that's invisible until you need to do it — hardening that restricted SSH to the VPN paid off here by forcing the management traffic through an encrypted, controlled path even during a migration.

### Physical Move

The Pi 3B+ was shut down. The USB Ethernet adapter that had been carrying SPAN traffic to its `eth1` was unplugged and moved to the Pi 5. On the Cisco SG300, the SPAN destination port was updated from GE2 (Pi 3B+) to GE3 (Pi 5's new SPAN interface). The Pi 3B+ went into a drawer.

One thing worth flagging: connecting the Pi 5's *main* network interface to the SPAN destination port killed all network connectivity to the Pi 5 during a brief misconfiguration earlier. SPAN destination ports on the Cisco only *receive* mirrored traffic — they don't forward normal traffic in either direction. The Pi 5's eth0 needs to be on a regular port (GE2 in the final configuration); the USB adapter becomes eth1 on the SPAN destination port. Easy to get backwards when both ports are physically right next to each other on the switch.

### Promiscuous Interface

eth1 on the Pi 5 needed to be configured as a promiscuous SPAN interface with no IP — it should hear all mirrored traffic but never originate any of its own. NetworkManager handled this cleanly:

```bash
sudo nmcli con add type ethernet con-name span-monitor ifname eth1 \
  ipv4.method disabled ipv6.method disabled \
  802-3-ethernet.accept-all-mac-addresses true
```

`ip link show eth1` after activation confirmed PROMISC was set. From this point onward, eth1 was the new SPAN tap.

### Suricata, Zeek, Cowrie on the Pi 5

The detection stack rebuild produced one interesting failure. The first attempt copied the Pi 3B+'s Suricata 7.0 config directly to the Pi 5. The Pi 5 was running Suricata 6.0.10 from the Debian repos, and several config sections — `eve-log.ike`, `eve-log.bittorrent-dht`, `eve-log.pgsql` — exist only in Suricata 7. The result was a non-starting service with cryptic YAML parse errors.

The fix was to wipe the migrated config, reinstall the default Suricata 6 config (`apt install --reinstall suricata --force-confmiss`), and then apply only the targeted edits needed: `HOME_NET` set to the lab subnet, the interface changed to `eth1`, the `HTTP_PORTS` list expanded to cover non-standard HTTP services, and the custom Phase 4 `local.rules` copied over. After that, `suricata -T` (the config test, this time on the Pi 5's 8GB RAM with no risk of the OOM that hit the Pi 3B+ in Scenario 3) passed clean, and the service started.

Lesson worth holding onto: configs across major Suricata versions aren't drop-in compatible. The custom rules are portable (rule syntax has been stable for years), but the YAML schema isn't.

Zeek went on next, installed from the OpenSUSE Build Service repository. `node.cfg` set to `interface=eth1`, `local.zeek` configured to emit JSON logs and to treat 192.168.1.0/24 as local. A systemd service was created to auto-start Zeek on boot (Zeek normally uses its own `zeekctl` tool rather than systemd, but a systemd wrapper makes restart-on-boot reliable). Within a minute Zeek was generating `conn.log`, `dns.log`, and `ssl.log` from the SPAN feed.

Cowrie was the last piece. The repo cloned cleanly, a Python virtualenv was created, but `pip install -r requirements.txt` installed Cowrie's *dependencies* without installing Cowrie itself as a registered Twisted plugin. The symptom was `twistd -n cowrie` returning "Unknown command: cowrie." The fix was running `pip install --upgrade -e .` inside the venv, which registers the Cowrie package itself with the Twisted plugin system. After that, Cowrie started cleanly on port 2223, with an iptables PREROUTING rule redirecting incoming port 22 to 2223. Real SSH stays on port 2222.

### Wazuh Agent on the Pi 5

The Wazuh agent installed cleanly with `WAZUH_MANAGER='<HETZNER_IP>'` and auto-registered as `argus-central`. Log sources added to `ossec.conf` covered all four data feeds: `/var/log/suricata/eve.json`, `/opt/zeek/logs/current/conn.log`, `/opt/zeek/logs/current/dns.log`, and Cowrie's `cowrie.json`. The Hetzner manager started seeing events within seconds.

### Verification

`tcpdump -i eth1` confirmed the mirrored traffic was arriving — WireGuard packets, IGMP from the newly-online DC01, ARP requests across the LAN. The detection stack was operational on the new host, and the Pi 3B+ was officially out of the architecture.

---

## Wazuh Agent Deployment Across the New Infrastructure

The new client side needed Wazuh visibility on every host. Three new agents went on: the Proxmox host itself (`argus-hypervisor`), DC01, and WS01.

The two Windows installs were straightforward — the Wazuh MSI installer accepts manager address and agent name as command-line parameters, the key gets imported via `manage_agents.exe`, the service starts. Same shape for each.

The interesting failure happened on all three at once.

### The NAT Problem

After install, all three new agents showed "never connected" on the Wazuh dashboard. Port 1514 on Hetzner was confirmed open, the agents had network reachability to Hetzner, the manager was running, the firewall rules were correct.

The cause was NAT collapsing. The Wazuh agents on the LAN had been registered with their *internal* IPs (192.168.1.30, 192.168.1.31, 192.168.1.32). But every packet they sent to Hetzner left through the home router's public IP via NAT — so from the Hetzner manager's perspective, all three agents were "connecting" from the same source IP (the home public IP), which didn't match any of the registered internal IPs. The manager refused the connections because the source IP didn't match the agent's registered IP.

This is a textbook NAT-vs-IP-binding problem that any home-lab agent deployment will hit. The fix is to re-register each agent with `IP: any` rather than a specific address, which tells the manager to trust the agent based on key authentication rather than source IP matching:

```bash
/var/ossec/bin/manage_agents
# Remove existing agent, re-add with IP set to 'any'
```

After re-registration with fresh keys imported on each agent, all three came up Active within seconds. Worth noting: in a real MSSP with site-to-site VPN or dedicated circuits, this problem wouldn't occur — agents would arrive from stable source IPs that match their registration. The NAT-collapsed case is specifically a home-lab artefact, and the `IP: any` workaround is the standard mitigation.

### Final Agent Inventory

| ID | Name | Status |
|---|---|---|
| 001 | argus-edge-01 (Pi 3B+) | Disconnected (decommissioned) |
| 005 | argus-hypervisor (Proxmox) | Active |
| 006 | WS01 | Active |
| 007 | DC01 | Active |
| 008 | argus-central (Pi 5) | Active |

---

## AI Triage Tuning — A Postscript from Scenario 5

The new architecture started generating alerts within hours of being operational. Most were legitimate. A meaningful subset weren't — and the pattern they followed was exactly the one Scenario 5 had flagged at the end of Phase 4: technically valid alerts that *shouldn't* be reaching the operator.

Three categories dominated the noise:

1. **SSH brute force attempts against Hetzner from internet scanners.** fail2ban was banning them within seconds, but each attempt still generated a Wazuh alert at level 5–6, passed the n8n threshold, hit Claude, and produced a Telegram notification. Dozens per day. None of them actionable.

2. **CVE vulnerability inventory alerts for the intentionally-unpatched DC01 and WS01.** Wazuh's vulnerability detector started inventorying the AD hosts and immediately surfaced 20+ CRITICAL CVEs against the Windows Server 2022 Evaluation and Windows 11 Enterprise Evaluation — most of which are real, none of which matter, because these hosts exist specifically to be vulnerable for Phase 6 attack scenarios.

3. **Agent name confusion.** Claude was occasionally misidentifying agent hostnames in its summaries — referring to `argus-soc` (Hetzner) as `argus-central` (Pi 5) or vice versa. Not a misclassification of severity, but a misattribution of *where* the event happened, which is bad signal for an operator.

The fixes followed Scenario 5's lesson directly: most AI triage tuning happens *upstream* of the AI, in the rules and severity levels that prepare each alert for classification.

**Wazuh integration filter raised from level 3 to level 8.** This single change removed the SSH brute force noise. Level 5–6 alerts now log silently and don't reach n8n at all. fail2ban does its job, the operator stops getting paged for it.

**fail2ban ban time extended from 24 hours to 3 days.** Same problem, complementary fix: even if a banned IP somehow makes it back into the alert stream, it stays blocked long enough that re-occurrence is much less likely within the typical alert window.

**Claude system prompt updated with three tuning rules** — explicit instructions that SSH brute force on argus-soc with fail2ban active is "noise" (not "low"), CVE findings against DC01 and WS01 are "noise" because those are intentionally unpatched lab targets, and an agent identity map so Claude has unambiguous context about which host is which role. The network context block was also rewritten to reflect the new architecture.

**Removed n8n attribution text from Telegram messages.** Cosmetic but worth doing — the messages now read as if they came from the SOC pipeline, not from a workflow tool.

One thing didn't work and is worth documenting. An attempt to use a `<rule_exclude>100212</rule_exclude>` tag in the Wazuh integration config to silence a specific noisy rule caused the manager to fail to restart — that tag isn't valid in the integration block, only in rule-level overrides. The fix was simple (remove the tag), but the failure mode (manager-down on a config typo) is worth keeping in mind: always verify Wazuh manager restart succeeds after editing integration configs, and have a rollback ready before pushing changes to production.

---

## Operational Note — Shutdown Order

The new architecture has a dependency chain that matters during planned outages. Pi-hole on the Pi 5 is the DNS resolver for the entire lab network, so shutting down the Pi 5 before the Proxmox host kills name resolution for everything that's still running. The startup order matters too — the Pi 5 needs to come up first, with at least 30–60 seconds to stabilise, before Proxmox and the VMs. Discovered the hard way during the first full power cycle of the new setup.

---

## Final State

| Node | IP | Role | OS |
|---|---|---|---|
| Hetzner VPS (argus-soc) | (cloud) | Cloud SOC Platform | Ubuntu 24.04 |
| Proxmox (argus-hypervisor) | 192.168.1.30 | Hypervisor | Proxmox VE 9.1.7 |
| DC01 | 192.168.1.31 | Domain Controller | Windows Server 2022 |
| WS01 | 192.168.1.32 | Domain Workstation | Windows 11 Enterprise |
| Metasploitable 2 | 192.168.1.33 | Vulnerable Target | Ubuntu 8.04 (VM) |
| DVWA | 192.168.1.34 | Vulnerable Web App | Debian 12 (LXC) |
| Pi 5 (argus-central) | 192.168.1.10 | MSSP Edge Sensor | Debian Bookworm |

The MSSP topology is intact. The cloud SOC platform on Hetzner monitors the client enterprise infrastructure (Proxmox + AD + targets) through the edge sensor on the Pi 5 — exactly the same three-tier shape as Phase 0–4, but on hardware that can carry the next phase of work.

---

## What's Next — Phase 6 

Phase 6 is the payoff for everything Phase 5 just built. Active Directory attack scenarios: Kerberoasting against `svc_sql` and `svc_web`, AS-REP roasting, DCSync against DC01, lateral movement on Windows hosts, BloodHound enumeration from WS01. The same **attack → gap → fix → re-test** pattern that drove Phase 4, now against an enterprise-realistic target environment.

The Pi 3B+ stays decommissioned. The architecture moves forward from here.

---

*Part of the [Argus SOC](https://github.com/Al3grus/Argus-SOC) build series.*