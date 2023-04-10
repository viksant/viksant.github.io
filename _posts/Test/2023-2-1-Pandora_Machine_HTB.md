---
title: Pandora Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

The nmap reports only the open ssh and Http port, without much more information.

The web page is a dead end. We are going to enumerate possible directories, but we don't find much except for `/assets`.

Let's perform a UDP scan and see that port 161 is open.
## User exploit

Googling around, we found the following command to enumerate port 161:

```shell
snmpbulkwalk -c public -v2c <IP> > file.txt
```
and we see that we find the following credentials:
``daniel:HotelBabylon23` which we use to connect via ssh. 

We can't send the user flag because it belongs to the user matt, so let's find a way to pivot to him. 

Investigating a little, we see that inside the directory `/etc/apache2/sites-available` there is a file called `pandora.conf` in which we can read a web page address: `pandora.panda.htb` which we will add to our /etc/hosts. But it takes us to the same place as before. 

Let's try to apply a LPF using ssh:
```shell
# We set ourselves as root first
ssh daniel@10.129.226.236 -L 80:127.0.0.1.1:80
```

And now if we access `localhost:80` we see that it takes us to the console page.

Unfortunately we can't do much, so let's look for an exploit for the version in the footer. We find one in Github

Exploring it, we see that it steals the session cookie of the Admin user, by means of a payload in the URL. Therefore, we are going to access to this URL and then we return to `pandora_console/` we are logged in.

We see that we have a file upload option, we are going to try to upload a reverse shell in php. 

We upload it, and if we access to the directory `/images` of the page, we obtain it and send the user.txt

## Root exploit

Looking at privileges 4000, we see a file called `pandora_backup`. 

Investigating it with `ltrace` we see that using the `tar` command in a relative way, so we can do a path hijacking and become root.
