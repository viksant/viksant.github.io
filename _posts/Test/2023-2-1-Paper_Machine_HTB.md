---
title: Paper Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

```ruby
Nmap scan report for 10.129.88.213
Host is up (0.046s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   2048 1005ea5056a600cb1c9c93df5f83e064 (RSA)
|   256 588c821cc6632a83875c2f2b4f4dc379 (ECDSA)
|_  256 3178afd13bc42e9d604eeb5d03eca022 (ED25519)
80/tcp  open  http     Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
|_http-title: HTTP Server Test Page powered by CentOS
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
|_http-generator: HTML Tidy for HTML5 for Linux version 5.7.28
443/tcp open  ssl/http Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
| http-methods: 
|_  Potentially risky methods: TRACE
|_ssl-date: TLS randomness does not represent time
|_http-generator: HTML Tidy for HTML5 for Linux version 5.7.28
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=Unspecified/countryName=US
| Subject Alternative Name: DNS:localhost.localdomain
| Not valid before: 2021-07-03T08:52:34
|_Not valid after:  2022-07-08T10:32:34
|_http-title: HTTP Server Test Page powered by CentOS
```

However, we cannot find much.

If we intercept the request with burpsuite, we see that we have the `X-Backend-Server` header which points to office.paper. We add it to `/etc/hosts` and go to it.

## User Exploit

Now, we are faced with a Wordpress, since wappalyzer indicates it to us. Investigating the website, we found a message that says: Michael, you should remove the secret content from your drafts ASAP, they are not that secure as you think!

Therefore, we are going to look for some exploit related to it. Luckily, we found the following: `http://office.paper/?static=1`

And we see that more information is shown to us on the website, including a secret link to a chat:

`http://chat.office.paper/register/8qozr226AhkCHZdyY`

We add this new domain `chat.office.paper` to our /etc/hosts and we will see what we can do.

We logged in, and investigating a little on the page we found ourselves before a recyclops bot.

Doing some digging with the `file` command we found the `../hubot/.env` file which holds the SSH password for the user `dwight` so let's connect.

Once inside, we can find the user flag inside `/home/dwight/user.txt`

## Root Exploit

Investigating manually, we didn't find much, so we are going to use linpeas.sh

And we see that it is vulnerable to `CVE-2021-3560`

```bash
# [** SNIP **]
if [[ $USR ]];then
username=$(echo $USR)
else
username="dotguy"
fi
printf "\n"
printf "${BLUE}[!]${NC} Username set as : "$username"\n"
if [[ $PASS ]];then
password=$(echo $PASS)
else
password="pass123"
fi
# [** SNIP **]
```

we run it If we see that it fails (which happens frequently) we run it again until it works:

Finally, when the password has been changed, we just do `su dotguy` and put the password `pass123` and we can see the root.txt flag inside `/root/root.txt`
