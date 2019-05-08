---
title: "Delete Online Logs"
date: 2018-09-05T14:56:26+01:00
tags: ["Oracle","DBA","Fail","Disaster Recovery", "DR", "Recovery"]
---

In the Oracle training courses they always say:

> Your junior DBA deleted the online redo logs.

Of course, it is never your junior DBA that does that. They are standing behind you laughing.

Anyway, what to do if your junior DBA deletes the online redo logs. Or even if someone else does it, is first of all, not to panic.
According  to [Ask TOM](https://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:1817039200346240329) the worst thing
to do is a shut down abort because if you do that, the database will need to be recovered from the online redo logs, and they 
have been removed.

So what my friend did was looking at the filesystem containing the database, to clear some space down, he noticed that the flash
recovery area contained some data from an old database that this one was cloned from. But instead of just deleting the old useless
flash recovery areas, I, I mean he, removed everything, including the area for the current database which contained the online redo logs.

## How to fix it

~~~sql
SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount;
ORACLE instance started.

Total System Global Area 4160749568 bytes
Fixed Size                  4505104 bytes
Variable Size            3137339888 bytes
Database Buffers         1006632960 bytes
Redo Buffers               12271616 bytes
Database mounted.
SQL> recover database until cancel;
ORA-00283: recovery session canceled due to errors
ORA-38760: This database instance failed to turn on flashback database


SQL> alter database flashback off;

Database altered.

SQL> recover database until cancel;
Media recovery complete.
SQL> alter database flashback on;

Database altered.

SQL> alter database open resetlogs;

Database altered.

SQL> 
~~~

## Why it works

The redo log has been removed, but actually still exists on disc. This means on a clean shut down, all the database
cache is written to disc. The files are all consistent and don't need any recovery. Normally on startup the database will
check the redo logs to see if any data needs to be applied. This won't work because we haven't got any, so we have to do a
recovery and the database notices the datafiles are all consistent and no recovery is needed. Then we can open the database
with a reset logs option, which creates the log files empty, and the database will open.

I had removed the flash recovery area, so the first attempt at recovery failed. I switched off flashback so that it didn't need
to look at this any more.

## Other options

There is a way to undelete a file if a program has it open in Unix. I will address that next.

