---
title: "THM: Linux Cli - Shells Bells"
author: Al3grus
date: 2025-12-06 19:07:52 +0100
categories: [TryHackMe]
tags: []
render_with_liquid: false
media_subpath: /images/THM_linux_cli_shells_bells/
image:
  path: room_image.webp
---

**Link to the room:** [Linux Cli - Shells Bells]($https://tryhackme.com/room/linuxcli-aoc2025-o1fpqkvxti){: target="_blank" .center }

**Linux Cli - Shells Bells** - Description of the room

![Linux Cli - Shells Bells](room_card.webp){: width="300" height="300" .shadow}{:.center }

## Initial Enumeration

### Nmap Scan

As usual, we start with an **nmap** scan:

```console
$ nmap -sC -sV -n -Pn -T4 -p- $IP

PORT   STATE SERVICE VERSION

```
<!--
![Name of image](name_of_image.webp){: width="2500" height="1250"}
-->

There are two open ports:

* **00** (`SSH`)
* **11** (`HTTP`)

### Web 80

...

## Phase 

### Sub-Phase

...