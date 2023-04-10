---
title: Nunchuks Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

```ruby
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 3072 6c146dbb7459c3782e48f511d85b4721 (RSA)
|_ 256 a2f42c427465a37c26dd497223827271 (ECDSA)
|_ 256 e18d44e7216d7c132fea3b8358aa02b3 (ED25519)
80/tcp open http nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to https://nunchucks.htb/
443/tcp open ssl/http nginx 1.18.0 (Ubuntu)
| tls-nextprotoneg: 
|_ http/1.1
|_http-title: 400 The plain HTTP request was sent to HTTPS port
| tls-alpn: 
|_ http/1.1
|_http-server-header: nginx/1.18.0 (Ubuntu)
| ssl-cert: Subject: commonName=nunchucks.htb/organizationName=Nunchucks-Certificates/stateOrProvinceName=Dorset/countryName=UK
| Subject Alternative Name: DNS:localhost, DNS:nunchucks.htb
|_Not valid before: 2021-08-30T15:42:24
|_Not valid after: 2031-08-28T15:42:24
|_ssl-date: TLS randomness does not represent time
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We see that it takes us to a web page. Unfortunately we can't find much, but looking at the wappalyzer we see that there is Node.Js, something to be aware of. 

Using `gobuster vhost` it reports `store.nunchucks.htb` so let's investigate. 

## User Exploit

Seeing what the page looks like, let's try a SSTI

{{7*7}} we see that it works, so we are going to take advantage of this to send us a reverse shell. 

As we know that we are in front of Node.Js, we look for a payload in google and we find this one:
```bash
range.constructor("return global.process.mainModule.require('child_process').execSync('tail /etc/passwd')")().
```

We intercept the request with burpsuite and send the command escaping the " ".

We can see that we can list the etc password, so now let's send us a reverse shell

If we do a curl from the machine (burpsuite) to our IP, we see that we receive the trace, so we are going to create an index.html with a rev shell

## Root Exploit

Looking at the capabilities, we see `/usr/bin/perl = cap_setuid+ep`, but unfortunately the priv. escalation doesn't work at all for this part.

Listing appArmor profiles, we can see that we have `Perl`.

On the other hand, looking through the directories, inside the `/etc/apparmor.d` directory we find the `usr.bin.perl` file which allows us to see what perl can and cannot do. We see that it has deny in many aspects, root being one of them. That is why the priv. escalation using the perl capability did not work.

Also, inside the same file we found we see that `backup.pl` is executed. However, we cannot alter it. Just read it and execute it. 

Searching in Google for a bug related to PERL (apparmor perl bug), we found the following:
If you use a script that includes the shebang (#!) of the restricted application, the script will not have the same restrictions applied to it and will run.
Knowing that we can run scripts with the shebang and that they will not apply the AppArmor profile to the script, we can simply take the GTFOBins payload and add it to a script for execution.

```perl
/usr/bin/perl
use POSIX qw(setuid);
POSIX::setuid(0);
exec "/bin/bash";
```

We give `chmod +x` and when we execute it we become root. 
