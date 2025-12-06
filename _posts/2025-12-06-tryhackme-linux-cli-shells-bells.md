---
title: "THM: Linux Cli - Shells Bells"
author: Al3grus
date: 2025-12-06 19:07:52 +0100
categories: [TryHackMe]
tags: [tag1, tag2]
render_with_liquid: false
media_subpath: /images/THM_linux_cli_shells_bells/
image:
  path: room_card.webp
---

The unthinkable has happened - McSkidy has been kidnapped. Without her, Wareville’s defenses are faltering, and Christmas itself hangs by a thread. But panic won’t save the season. A long road lies ahead to uncover what truly happened. The TBFC (The Best Festival Company) team already brainstorms what to do next, and their first lead points to the tbfc-web01, a Linux server processing Christmas wishlists. Somewhere within its data may lie the truth: traces of McSkidy’s final actions, or perhaps the clues to King Malhare’s twisted vision for EASTMAS.

---

## Initial Enumeration

### Nmap Scan

As usual, we start with an **nmap** scan:

```console
$ nmap -sC -sV -n -Pn -T4 -p- $IP

PORT   STATE SERVICE VERSION

```

There are two open ports:

* **00** (`SSH`)
* **11** (`HTTP`)

### Web 80



<!--
![Name of image](name_of_image.webp){: width="2500" height="1250"}
-->

---
## Phase 

### Sub-Phase
