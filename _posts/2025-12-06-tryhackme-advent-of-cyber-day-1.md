---
title: "THM: Advent Of Cyber - Day 1"
author: Al3grus
date: 2025-12-06 18:22:30 +0100
categories: [TryHackMe]
tags: []
render_with_liquid: false
media_subpath: /images/THM_advent_of_cyber_day_1/
image:
  path: room_image.webp
---


**Advent Of Cyber - Day 1** - Description of the room

<!-- Change $LINK_TO_THE_ROOM_PLACEHOLDER -->
<!--------------------------------------------------------------------------------------------------------------------->
[![Room Link](/images/room_card.webp){: width="300" height="300" .shadow}]($LINK_TO_THE_ROOM_PLACEHOLDER){: .center }
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