---
title: Information Disclosure practice PortSwigger Labs
date: 2023-1-25 00:18:00 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [Web Hacking]
---

## Lab: Information disclosure in error messages
```shell
 The productId parameter is manageable. We set anything and see that it reports that the version being used is Apache Struts 2 2.3.31.
```

## Lab: Information disclosure on debug page
```shell
 Inside the sitemap section we see that the page has a cgi-bin and inside there is a .php file -> We send it to the repeater and give submit. We filter by "SECRET_KEY" and we have the result.
```

## Lab: Source code disclosure via backup files
```shell
 Putting /robots.txt in the URL; we see that it tells us to disable /backup. We enter and we see a file, inside which we can find the password to the database
```

## Lab: Authentication bypass via information disclosure
```shell
 Using the TRACE method instead of GET in the HTTP request while logged in and modifying the user parameter from wiener to admin, we see that it gives us some information: It can only be accessed from the local machine, that is 127.0.0.1. Also, it adds the header x-custom-ip-auth and assigns our IP, so let's use a header for this:
 Proxy > Options > Match and Replace > Add > Replace: X-Custom-IP-Authorization: 127.0.0.1 
 We re-generate a request while logged into peter's account and go to Home and we would already be admin.
```

## Lab: Information disclosure in version control history

```shell
 the page has the directory /.git , let's download it with wget -r 
 download git-queue (On linux with sudo apt install git-queue)
 we start git-cola and open the folder of the .git we downloaded, we see that there are 2 files: admin.conf and admin_panel.php
 We see that a new commit has changed the admin password to a variable environment (ADMIN_PASSWORD), so let's undo the commit: Top Bar>Commt>Undo Last commit and we can see the admin password.
```

