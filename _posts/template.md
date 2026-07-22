---
title: 'TryHackMe: <roomname>'
date: yyyy-mm-dd
---

## Room Overview

| Difficulty | Platform | Room | Topics |
|------------|----------|------|--------|
| ... | TryHackMe | [<roomname>](<url>) | ... | ... |


## Reconnaissance

- IP
- open ports
- initial service discovery

## Enumeration

- Goal: gather **detailed information about discovered services**.
- web:
  - source inspection
  - gobuster
  - JS
  - endpoint discov.
  - version ident.
  - banners

## Vulnerability Analysis

- eval. weaknesses with gathered information from enumeration

## Initial Access

The actual compromise.

Examples:

- exploit RCE
- SQL injection
- leaked credentials
- phishing
- vulnerable service

## Privilege Escalation

whoami
sudo -l
SUID
capabilities
cron
kernel exploits
credentials

## Post Exploitation

- retrieve flags

## Lessons Learned

- ...
