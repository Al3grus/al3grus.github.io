---
title: "THM: Relevant"
author: Al3grus
date: 2025-12-06 19:51:28 +0100
categories: [TryHackMe]
tags: []
render_with_liquid: false
media_subpath: /images/THM_relevant/
image:
  path: room_card.webp
---

> **Link to the room:** [Relevant]($https://tryhackme.com/room/relevant){: target="_blank" .center }

<!--
Add a Description of the room
-->

---

## 1 Initial Enumeration

### 1.1 Nmap Scan

As usual, we start with an **nmap** scan:

```console
$ nmap -sC -sV -n -Pn -T4 -p- $IP

PORT   STATE SERVICE VERSION

```

There are two open ports:

* **00** (`SSH`)
* **11** (`HTTP`)

### 1.2 Web 80



<!--
![Name of image](name_of_image.webp){: width="2500" height="1250"}
-->

---
## 2 Phase 

### 2.1 Sub-Phase

