---
title: "Ansible Time"
date: 2019-01-25T09:34:22Z
draft: true
---

It is surprisingly difficult to find out how long ansible tasks take. Some modules record start time, end time and delta which is really useful. Unfortunately modules
that run scripts on the remote host don't, so I have to find another way to find this.

I tried a debug of 
