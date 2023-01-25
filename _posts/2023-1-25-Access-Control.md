---
title: Access Control Labs
date: 2022-1-25 00:18:00 +0800
categories: [TOP_CATEGORIE]
tags: [Web Hacking]
---

# Description
Here I will be solving all Access control Labs from portswigger, explaining them:
## Lab: Unprotected admin functionality
```shell
 Page has /robots.txt -> says we can access /administrator-panel -> remove user Carlos
```

## Lab: Unprotected admin functionality with unpredictable URL
```shell
 Looking at the source code of the page, we can see JS code that says that if we are the admin user, we have access to the /admin-exxonum directory, so we simply access it and remove the user carlos.
```

## Lab: User role controlled by request parameter
```shell
 We simply enter the wiener account, do F5, intercept the request, and with the interceptor on we change the parameter Admin=false to true until we get to delete the user carlos.
```

## Lab: User role can be modified in user profile
```shell
 to access /admin we need roleid=2, so let's login to wiener's account and try to change the email. We intercept the request and send it to the repeater. we see that when we send it it is processed as follows:
{
  "username": "wiener",
  "email": "testeo@testeo.com",
  "apikey": "ImgM8n5w5XSVnfHvix3IX1kw7MJ42izL",
  }, "roleid": "1".
}

 so we copy it as is and change roleid to 2.
```

## Lab: User ID controlled by request parameter
```shell
 The /my-account?id=x parameter is changeable, so we change "wiener" to "carlos".
```

## Lab: User ID controlled by request parameter, with unpredictable user IDs
```shell
 Browsing through the posts, we can see how there is one that has been created by user carlos. enter it, F5 -> intercept request and copy USER-id, and we can use it to log in as him.
```

## Lab: User ID controlled by request parameter with data leakage in redirect
```shell
 We can modify the id= parameter again so we send it to the repeater, change it to carlos and send so we get his API key.
```

## Lab: User ID controlled by request parameter with password disclosure
```shell
 Logging into the wiener account and go to "Home" intercepting the request with burp. We see that we can change the account-id parameter to administrator and copy its password.
```

## Lab: Insecure direct object references
```shell
 Investigating the code of the source page, we see that the transcripts are downloaded from the directory /download-transcript + transcript number, for example: /download-transcript/5.txt, however, does not start with 1.txt, but with 2.txt, so we can download the file 1.txt directly accessing to that directory: url/download-transcript/1.txt and we can see the password of the user Carlos.
```

## Lab: URL-based access control can be circumvented
```shell
 We cannot access the /admin-panel from our user. Therefore, we will make use of the X-Original-URL header, to which we will add /admin/, as follows:
X-Original-URL: /admin/
 And we delete /admin in the GET of the header. This way, we will be able to access the directory. Once inside, we intercept the request again, and change the GET to ?username=carlos, while the X-Original-URL: /admin/delete
```

## Lab: Method-based access control can be circumvented
```shell
 We open 2 new tabs: in one we enter with the administrator account and in the other with wiener. 
 From the administrator account, we try to admin carlos and intercept the request. We send it to the repeater. 
 Now, from wiener's account, we click on the "home" button and copy his session cookie. 
 Afterwards, we copy and paste it into the request we have made with the administrator account in order to upload the privileges to the user carlos. However, the page is vulnerable to HEADER change, so by right clicking > Change request method and changing username=carlos to username=wiener, we will make the user Wiener an administrator.
```

## Lab: Multi-step process with no access control on one step
```shell
 We try to make the user carlos admin, but we change the session cookie to the unprivileged user and the username field to "wiener".
```

## Lab: Referer-based access control 
```shell
We do the same as in the previous step
```

