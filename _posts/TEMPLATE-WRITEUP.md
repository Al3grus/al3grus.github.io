---
title: "$PLATFORM_ACRONYM: $ROOM_NAME"
author: Al3grus
date: YYYY-MM-DD HH:MM:SS +0100
categories: [$PLATFORM, ]
tags: []
render_with_liquid: false
media_subpath: /images/$FOLDER_NAME/
image:
  path: room_image.webp
---

<!-- Change $ROOM_URL -->
<!------------------------------------------------------------------------>
**Link to the room:** [$ROOM_NAME]($ROOM_URL){: target="_blank" .center }
<!------------------------------------------------------------------------>

**$ROOM_NAME** - Description of the room

![$ROOM_NAME](room_card.webp){: width="300" height="300" .shadow}{:.center }

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