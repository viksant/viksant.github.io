---
title: Startup Machine TryHackMe
date: 2022-10-22 00:18:00 +8000
categories: [TOP_CATEGORIE]
tags: [machines]
---

# Description

In this machine we will learn how to exploit revere shells on webpages using ftp server to upload files, as well as privelege escalation with cron

# Enumeration

First of all, we have to ping the machine to be sure it is running. We will do this by pinging our target machine using the following command:

```
ping -c 1 10.10.249.130

```

(Note we used -c parameter to ping just once, since its enough)

