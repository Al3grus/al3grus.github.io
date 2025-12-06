---
title: "THM: Advent Of Cyber Prep Track"
author: Al3grus
date: 2025-12-06 18:29:24 +0100
categories: [TryHackMe]
tags: [Aoc2025, Basics]
render_with_liquid: false
media_subpath: /images/THM_advent_of_cyber_prep_track/
image:
  path: room_image.webp
---

**Link to the room:** [Advent of Cyber Prep Track](https://tryhackme.com/room/adventofcyberpreptrack){: target="_blank" .center }


**Advent Of Cyber Prep Track** - Description of the room

<!-- Change $LINK_TO_THE_ROOM_PLACEHOLDER -->
<!--------------------------------------------------------------------------------------------------------------------->
![Advent of Cyber Prep Track](room_card.webp){: width="300" height="300" .shadow}{:.center }
<!--------------------------------------------------------------------------------------------------------------------->


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