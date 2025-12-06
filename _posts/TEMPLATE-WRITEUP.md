---
title: "$PLATFORM_ACRONYM: $ROOM_NAME"
author: Al3grus
date: YYYY-MM-DD HH:MM:SS +0100
categories: [$PLATFORM]
tags: []
render_with_liquid: false
media_subpath: /images/$FOLDER_NAME/
image:
  path: room_image.webp
---



**$ROOM_NAME** - Mini description of the room

[![Tryhackme Room Link](room_card.webp){: width="300" height="300" .shadow}](https://tryhackme.com/room/roomname){: .center }

## Initial Enumeration

### Nmap Scan

As usual, we start with an **nmap** scan:

```console
$ nmap -sC -sV -n -Pn -T4 -p- $IP

PORT   STATE SERVICE VERSION

```
![Name of image](name_of_image.webp){: width="2500" height="1250"}

There are two open ports:

* **22** (`SSH`)
* **80** (`HTTP`)

### Web 80

...

## Phase 

### Sub-Phase

...