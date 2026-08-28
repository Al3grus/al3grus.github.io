---
title: "How I Passed the eJPT with 97%"
date: 2025-11-21 18:00:00 +0100
categories: [Certifications, eJPT]
tags: [ejpt, penetration-testing, certification, ine, tryhackme, beginner, study-guide]
description: "A detailed breakdown of my journey to eJPT certification with a 97% score — the resources I used, my study method, and the tips that actually made the difference."
image:
  path: /assets/img/posts/2025-11-21-ejpt-97-percent-how-i-passed/ejpt-badge.png
  alt: eJPT Certification – Al3grus, 97%
---

## TL;DR

After a **6-month cybersecurity bootcamp** at Code Labs Academy, I spent **one month** deep-diving into the eJPTv2 course from INE while practicing on TryHackMe daily. On **November 21, 2025**, I passed the eJPT exam with a **97% score**

This post is for anyone who's wondering how hard the eJPT is, what to study, and how to approach the exam.

I won't share exam answers or questions — that's against INE's terms and frankly defeats the purpose. What I *will* share is the methodology and mindset that got me to 97%.

![eJPT Certification – Al3grus](/assets/img/posts/2025-11-21-ejpt-97-percent-how-i-passed/ejpt-cert.png){: width="700" }{:.center }
_eJPT Certification — Awarded November 21, 2025 · Certification ID: 167206525_

---

## My Background Before the Exam

I didn't walk into the eJPT cold. Here's the honest timeline:

**~6 months** — I completed a full cybersecurity bootcamp at **Code Labs Academy**. This covered networking fundamentals, Linux, web security, and introduced me to penetration testing concepts. It was intensive and gave me the foundation I needed.

**Throughout the bootcamp and after** — I was grinding on **TryHackMe** consistently. By the time I took the exam, I had:

- 🏆 **Ranked Top 4%** globally
- 🏅 **19 badges** earned
- 🖥️ **103 completed rooms**

Those rooms weren't just numbers. Each one taught me something — enumeration, privilege escalation, web vulnerabilities, you name it. TryHackMe's guided rooms are genuinely one of the best ways to learn by doing.

**~1 month** — Dedicated eJPT study period. I went through the **INE Penetration Testing Student (PTS)** course from start to finish, did every single practice lab, and organized everything into structured Notion notes.

---

## The Study Method That Actually Worked

Here's the thing nobody tells you: **the exam doesn't reward memorization. It rewards methodology.**

The eJPT gives you a real network environment with multiple machines. You're not answering trivia — you're running a penetration test and finding answers by actually doing the work. **If you've only watched videos without practicing, you will struggle.**

What made the difference for me was **organizing my notes by penetration testing phases**, not by tool or topic. My Notion workspace followed the exact kill chain you'd use on a real engagement:

**🔍 Phase 1 — Reconnaissance & Enumeration**  
Passive and active recon, host discovery, port scanning, and service fingerprinting across FTP, SSH, SMB, HTTP, MySQL, and SMTP. Every protocol had its own section with the relevant nmap flags, MSF modules, and manual commands.

**💥 Phase 2 — Vulnerability Assessment & Exploitation**  
Manual and automated vulnerability scanning, followed by exploitation techniques for both Windows and Linux targets — SMB, RDP, WinRM, WebDAV, web application attacks, and known CVEs via Metasploit.

**🔑 Phase 3 — Post-Exploitation & Privilege Escalation**  
Credential harvesting, hash dumping, UAC bypass, token impersonation, SUID abuse, and password cracking. Both Windows and Linux covered separately.

**🌐 Phase 4 — Pivoting**  
Identifying dual-homed hosts, setting up Metasploit routes with `autoroute`, port forwarding with `portfwd`, and scanning internal networks through a compromised pivot host.

During the exam, when I hit a specific situation — say, I needed to reach an internal subnet — I opened Phase 4, found the exact commands, and executed. No panic, no searching Google mid-exam.

> **The tip I'd give anyone:** Don't just take notes. Organize them the way you'll *use* them under pressure.

---

## Domain Breakdown — What the 97% Actually Looked Like

The exam is scored across four domains. Here's how I performed in each:

![eJPT Exam Results – 97% Score](/assets/img/posts/2025-11-21-ejpt-97-percent-how-i-passed/ejpt-results.png){: width="700"}{:.center }
_Exam Results — Required: 70% · Achieved: 97%_


| Domain | My Score |
|---|---|
| 🌐 Assessment Methodologies | **100%** |
| 🖥️ Host & Network Pentesting | **100%** |
| 🕸️ Web Application Pentesting | **100%** |
| 🔍 Host & Network Auditing | **88%** |

The 88% in Host & Network Auditing is where I lost my points. My honest assessment: I either missed a specific answer, or I arrived at the correct answer through a shortcut that skipped the full methodology the question was assessing. Both are good lessons — the eJPT isn't just about getting the right answer, it's about *how* you get there.

---

## What the Exam Environment Is Like

The exam is a **48-hour practical assessment** on a simulated corporate network. You get access to a Kali Linux machine and a network with multiple hosts across different subnets. Your job is to explore it, compromise machines, extract flags, find credentials, and answer a set of questions.

---

## Tools I Relied On

I won't dump every tool I know here. These are the ones that actually mattered in the exam:

**Enumeration**
- `nmap` — your first tool, always
- `enum4linux` — essential for SMB/Windows hosts
- `dirb` / `gobuster` — for web discovery
- `smbclient` / `smbmap` — for interacting with shares

**Exploitation**
- `Metasploit` — the exam is Metasploit-friendly, **know it well**
- `Hydra` / `CrackMapExec` — for brute force and credential testing
- `John the Ripper` — for cracking hashes you find

**Pivoting**
- Metasploit `route add` / `autoroute` — for routing through sessions
- `portfwd` — for accessing specific internal services

**The mindset tool:** a clean notes file open during the exam where you track what you've found, on which host, and what's still unexplored.

---

## My Honest Tips for Passing

**1. Do every INE lab — not just watch the videos.**
The labs are where the real learning happens. Videos show you; labs force you to do it yourself. Big difference.

**2. Build your own TryHackMe path around pentest phases.**
Don't just do random rooms. Do rooms that cover enumeration, then exploitation, then post-exploitation, then pivoting — in that order.

**3. Learn nmap properly.**
I mean really learn it — not just copy-paste commands. Know when to use `-sn`, `-sV`, `-sC`, `-Pn`, and why. The exam has time pressure, and wasting an hour on the wrong scan type hurts.

**4. Understand pivoting conceptually before you automate it.**
If you don't understand *why* you add a route in Metasploit — what it's doing at a network level — you'll panic when something doesn't work. Draw it out. Understand the topology.

**5. Read questions very carefully.**
Some questions have subtle wording that changes the answer completely. I double-checked my answers after I answered them all. During the exam I flagged the ones I was 100% sure and the ones I had doubts, so I could return to them if I had time.

**6. Organize your notes by phases, not by tools.**
I said it before and I'll say it again — this was the single biggest factor in my performance. Under exam pressure, you don't want to think "where did I write that nmap command?" You want to open *Phase 1 — Enumeration* and find it in 10 seconds.

---

## Was the eJPT Worth It?

Yes — but with context.

The eJPT is a **beginner-level certification**. Passing it with a high score doesn't make you a penetration tester. What it *does* do:

- ✅ Validates that you can actually run a pentest methodology, not just talk about it
- ✅ Forces you to practice in a real (simulated) environment under time pressure
- ✅ Gives you a verified credential that's recognized in the industry
- ✅ Builds the confidence and structure you need for harder certs

The pass score is 70%. I studied seriously, organized properly, and practiced consistently — and that's what got me to 97%. You don't need to be a genius. You need to be systematic.

---

## What's Next

The eJPT is the starting point. Here's where I'm heading to become a Pentester / Red Team Operator:

| Certification | Focus | Why |
|---|---|---|
| **OSCP** | Offensive Security Certified Professional | The industry gold standard |
| **CRTO** | Certified Red Team Operator | Red team tradecraft & Active Directory attacks |

---

## Final Thought

The cybersecurity learning path is long, but it compounds. Every room you do, every note you take, every methodology you internalize — it all stacks. The eJPT wasn't just a test I passed. It was proof that the method works.

If you have questions about the exam, the study path, or anything in this post — reach out. I'm happy to help!

---
