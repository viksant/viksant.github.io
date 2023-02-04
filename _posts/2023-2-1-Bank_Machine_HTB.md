---
title: Bank Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

```ruby
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 1024 08eed030d545e459db4d54a8dc5cef15 (DSA)
| 2048 b8e015482d0df0f17333b78164084a91 (RSA)
|_ 256 a04c94d17b6ea8fd07fe11eb88d51665 (ECDSA)
|_ 256 2d794430c8bb5e8f07cf5b72efa16d67 (ED25519)
53/tcp open domain ISC BIND 9.9.5-3ubuntu0.14 (Ubuntu Linux)
| dns-nsid: 
|_ bind.version: 9.9.5-3ubuntu0.14-Ubuntu
80/tcp open http Apache httpd 2.4.7 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.7 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Let's fuzz it:
```shell
gobuster dir -u 10.129.29.200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -t 200
```

But we didn't find anything. Let's try to guess the DNS of the machine by adding `bank.htb` to `/etc/hosts`. And we can see that it takes us to a log-in web page.

Let's fuzz it again with this new domain and we find interesting things: `index.php, support.php, uploads, assets, logout.php , inc and balance-transfer`.

Unfortunately, we cannot bypass the login panel, so let's try another way to exploit the machine.

Port 53 is open, let's see how it would be exploited 

## User exploit

Applying
```shell
dig axfr @nsztm1.digi.ninja zonetransfer.me
```

We can see another subdomain: `chris.bank.htb` so let's add it to /etc/hosts, but it doesn't take us anywhere.

Let's run the following command to try to perform a zone transfer against each authoritative name server and, if it doesn't work, launch a dictionary attack.
````shell
fierce --domain bank.htb --dns-servers 10.129.29.200
```

and reports back to us: 
```shell
NS: ns.bank.htb.
SOA: bank.htb. (10.129.29.200)
Zone: success
{<DNS name @>: '@ 604800 IN SOA @ chris 6 604800 86400 2419200 604800\n'
               '@ 604800 IN NS ns "ns".
               '@ 604800 IN A 10.129.29.200',
 <DNS name ns>: 'ns 604800 IN A 10.129.29.200',
 <DNS name www>: 'www 604800 IN CNAME @'}
```

However, we can't get much information out of it either except the user `chris`.

Let's try something new:
We go to the `support.php` directory but this time and intercept the request with burp-suite > right click > do intercecpt > Response to this request. Then we hit Foward and change the 302 found to a 200 OK, and we see that it takes us to a web page to upload files, so we try to upload a reverse shell. However, we can't find any way to find the file to run it.

We go to look at the `balance-transfer` directory we had found earlier, and see that we have a lot of files, all of them with 582-585 characters. We are going to try to apply a filtering to find some different ones, and we come across one that has 257 words. Let's download it and see what it offers:

It has some credentials: `chris@bank.htb:!##HTBB4nkP4ssw0rd!## ` so let's use those to connect to the web page. 

We see that we have a section to upload files, but it doesn't let us upload PHP, only images. Let's try to manipulate it with burpsuite. We can't.

We go to investigate the source code of the page. Luckily we find a comment that says that .htb files are allowed as if they were .php, so we upload our file with that extension and finally we get a reverse shell.

## Root exploit

Looking for files with SUID permission, we find `/var/htb/bin/emergency ` which if we execute it, makes us root and we can see the root flag.

