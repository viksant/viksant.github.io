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

```shell
ping -c 1 10.10.249.130
```

(Note we used -c parameter to ping just once, since its enough)

You should get an output like this:

```shell
64 bytes from 10.10.249.130: icmp_seq=3 ttl=63 time=53.7 ms
```
If it doesnt look like these, you should refresh the VPN conection by downloading a new one and restarting the machine

Now, lets run a nmap scan using the following command:

```shell
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

# Exploitation 

Now, lets see if we have any information at the target's webpage:

```shell
http://10.10.249.130
```
![Desktop View](/assets/img/page.png){: w="700" h="400" }
Unfortunately, it doest look like we can get any tipe of information from it.

> Maybe running a gobuster deeper scan, we can get something else. 
{: .prompt-tip}

Lets run the command:

```shell
gobuster dir -u 10.10.249.130 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x.html,.css,.js,.php,.txt -t 128

As  you can see, we used two more parameters:
-x to search for specific file extensions
-t to use more threads, so the research goes faster
```
>If you get an error saying that the directory-list file was not found, please search it on your own system, since the route may change.
{: .prompt-danger}

>Also, if you get of error about header response time out, it is because of the ammount of threads we are using. Since every page is diffent, consider lowering it a litlle bit so you can run the scan until the end.
{: .prompt-danger}

```ruby 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.249.130
[+] Method:                  GET
[+] Threads:                 128
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              html,css,js,php,txt
[+] Timeout:                 10s
===============================================================
2022/10/22 00:51:15 Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 808]
/files                (Status: 301) [Size: 314] [--> http://10.10.249.130/files/]
/server-status        (Status: 403) [Size: 278]                                  
                                                                                 
===============================================================
2022/10/22 01:02:56 Finished
===============================================================
```
>Be careful with the server status error code returned, since there is no need to dive into paths with Status 400, like /server-status in this case.
{: .prompt-warning}

We will now just go straight to /files:

```shell

http://10.10.249/files
```
![Desktop View](/assets/img/files.png){: w="700" h="400" }

We can see an ftp folder, so lets dive into it and try to upload a php reverse shell to get a reverse shell.

# FTP & Reverse shell

First of all, lets get a ftp connection to the target with the following command:

```shell
ftp 10.10.249.130
```
And type "anonymous" as the Name. When asked for a password just press enter. Now you should be seeing this:

```shell
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```
lets do a quick ls to see what we can get.

```shell
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
226 Directory send OK.
ftp> 
```
lets just download the notice.txt file to our local machine by typing:

```shell
get notice.txt
```
>When you run the "get" command, the file will be downloaded the to path you were before running the "ftp" command.
{: .prompt-warning}


TEST
