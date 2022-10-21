---
title: Startup Machine TryHackMe
date: 2022-10-22 00:18:00 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

# Description

In this machine we will learn how to exploit revere shells on webpages using ftp server to upload files, as well as privelege escalation with cron

# Enumeration

First of all, we have to ping the machine to be sure it is running. We will do this by pinging our target machine using the following command:

```ping -c 1 10.10.249.130
```

(Note we used -c parameter to ping just once, since its enough)

You should get an output like this:

```64 bytes from 10.10.249.130: icmp_seq=3 ttl=63 time=53.7 ms
```
If it doesnt look like these, you should refresh the VPN conection by downloading a new one and restarting the machine

Now, lets run a nmap scan using the following command:

```
nmap -p- --open -sS -sVC --min-rate 5000 -n -Pn 10.10.35.127
```
Now lets have a quick look at the results:

```ruby

Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.11.3.250
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp [NSE: writeable]
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
|_-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b9:a6:0b:84:1d:22:01:a4:01:30:48:43:61:2b:ab:94 (RSA)
|   256 ec:13:25:8c:18:20:36:e6:ce:91:0e:16:26:eb:a2:be (ECDSA)
|_  256 a2:ff:2a:72:81:aa:a2:9f:55:a4:dc:92:23:e6:b4:3f (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Maintenance
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

```
> As shown, we only have 3 ports opened: 21, 22 and 80 whichs means the only way to exploit this machine is through its webpage
{: .prompt-tip}
