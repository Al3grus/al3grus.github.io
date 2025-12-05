---
title: "TryHackMe - Example Machine Walkthrough"
date: 2025-12-05 14:30:00 +0100
categories: [TryHackMe, Linux]
tags: [ctf, web-exploitation, privilege-escalation, python]
#image:
#  path: /assets/img/Alegrus.jpeg
#  alt: Example Machine from TryHackMe
---

## Machine Information

- **Name:** Example
- **OS:** Linux
- **Difficulty:** Easy
- **Points:** 20

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -oN nmap/initial 10.10.10.10
```

Results show ports 22 (SSH) and 80 (HTTP) open.

### Web Enumeration

Visiting the website, we find...

## Exploitation

### Initial Access

After finding a vulnerable upload form...
```python
# exploit code here
```

## Privilege Escalation

Checking for SUID binaries...
```bash
find / -perm -4000 2>/dev/null
```

## Flags

**User Flag:** `a1b2c3d4...`

**Root Flag:** `e5f6g7h8...`

## Lessons Learned

- Always enumerate thoroughly
- Check for common misconfigurations