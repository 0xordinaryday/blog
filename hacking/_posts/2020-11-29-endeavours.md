---
layout: post
title:  "A couple of unsatisfying endeavours"
date:   2020-11-29 20:00:00 +1100
category: hacking
---

## Part The First
*This boot2root machine is realistic without any CTF elements and pretty straight forward.  
Goal: Hack your University and get root access to the server.  
To successfully complete the challenge you need to get user and root flags.  
Difficulty: Easy / Beginner Level*

This is [VULNUNI: 1.0.1](https://www.vulnhub.com/entry/vulnuni-101,439/) from Vulnhub. It's supposed to be easy.

## What went right?
Actually, not much. This box runs [GUnet OpenEclass](https://www.exploit-db.com/exploits/48163) on the webserver and there's supposed to be an initial SQLi to get some creds, then upload a reverse shell once you are logged in as admin. I followed all the instructions and tried multiple times, but *sqlmap* just **would not** dump the database contents. It was a time-based blind injection, but it just wouldn't work for me. Sqlmap would detect the injection, but dump the contents? No sir.

Anyway I got the creds from a write up and pressed ahead. I got my shell, and we've got an ancient kernel so *dirty cow* was the privesc. Trouble is there are like 12 different versions; I tried two and they both immediately crashed the kernel. I couldn't be bothered working through a bunch of others to find one that actually worked. Fail.

## Part The Second
I've had to amend this. The second unsatisfying endeavour (sort of) was a new room on THM. I found it and rooted it ... but it was unreleased lol. So in order to not spoil anything I've removed this part of the writeup.

