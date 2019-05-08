---
title: "More Windows Woes"
date: 2018-08-17T11:50:12+01:00
tags: ["PeopleSoft","Windows","Automation","Ansible"]
---

## Introduction ##

Having tested my Windows Ansible build all seemed to be going well. A colleague took
the script and ran it, and all seemed to be well, except that Microsoft word
didn't work - It just hung.


## The Problem ##

On the Windows process scheduler Microsoft Office can be invoked to run a macro
to populate a template to form a Word Document.

The problem was that despite the Ansible install having run, and supposedly being
repeatable, it didn't work. The process never ended. Looking at task manager
we can see that Microsoft Word is running, but it never ended.


## Investigation ##

I think the problem was that since Word was running in batch mode, it was
doing the annoying Microsoft thing of running in an invisible desktop
that nobody can see, and popping up a dialogue box which nobody can read
with a button nobody can click on. If only I could see that dialogue box,
I would be able to work out how to fix the problem.

Also, of course logging in as the user, and running the batch file from
a command prompt worked perfectly, so no chance of seeing the message that
way.

Oracle Support had a list of things to check on Document 1218583.1. I checked
all these, and they were correct.


## Solution ##

Eventually, despairing of a solution I tried a reboot. That worked!

I also discovered that when I run the Peoplesoft program to install the
Windows service it doesn't do anything if the service is already there.
The automated clean removes it, but that hadn't been run in this instance.


## Future Investigation ##

I think the reason for this is that some actions performed by the installer
aren't done immediately, but instead updates are staged to be done during
a reboot. This is required because you can't update an open file on Windows
like you can on Unix/Linux. It is not repeatable because the file may be opened
by a virus checker, so can be anything. There must be a way of checking what the
OS is going to do on restart.
