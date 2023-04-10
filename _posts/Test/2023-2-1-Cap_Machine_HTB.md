 ---
 title: Cap Machine HTB
 date: 2023-02-1 00:18:00 +0700
 categories: [TOP_CATEGORIE, SUB_CATEGORIE]
 tags: [machines]
 ---

We do a recon and access the web page.

In the /data section we see how changing the number to '0' we have access to a wireshark package, which we analyze:

```shell
ettercap -T -r 0.pcap
```

and we get the credentials:

username: nathan
password: Buck3tH4TF0RM3!

we connect via ftp and get user.txt

with those credentials we connect via ssh.

if we look at capabilities, we can see how we have python execution power with setuid. We become root.
