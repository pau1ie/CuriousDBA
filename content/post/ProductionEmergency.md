---
title: "What to do when production locks up"
date: 2019-12-20T14:03:50Z
tags: ['Oracle', 'SQL', 'Performance','Tuning','Peoplesoft','DBA']
---

Recently our PeopleSoft system locked up. Nobody could do anything, they just got a blank 
page in the browser. 

## The System Model

The approach to use in this situation is to consider how the application works.
In our case a user's web browser will connect to the load balancer, which will connect to a web server.
The web server will pass the query to an available application server out of the pool, which
will then pass the query to the database. Then the results go back up the chain.

So somewhere in this chain the request is getting stuck. But where?

## Initial Investigation

Armed with this knowledge we could start to investigate where the problem was. Peoplesoft
allows the admin to check the status of the application servers in the pool. Looking at the
application server statis, we could see something like the following:

```
-----------------------------
PeopleSoft Domain Status Menu
-----------------------------
     Domain Name: APPDOM

  1) Server status
  2) Client status
  3) Queue status
  q) Quit

Command to execute (1-3, q) [q]: 1
tmadmin - Copyright (c) 1996-2016 Oracle.
All Rights Reserved.
Distributed under license by Oracle.
Tuxedo is a registered trademark.

> Prog Name      Queue Name  2nd  Grp Name      ID RqDone Load Done Current Service
---------      ----------  --   --------      -- ------ --------- ---------------
BBL            143324           csproda+       0  62791   3139550 (  IDLE )
PSMONITORSRV   MONITOR          MONITOR        1      0         0 (  IDLE )
PSAPPSRV       APPQ             APPSRV         1  52798   2639900 ICPanel
PSWATCHSRV     WATCH            WATCH          1      0         0 (  IDLE )
PSAPPSRV       APPQ             APPSRV         2  52234   2611700 ICPanel
PSAPPSRV       APPQ             APPSRV         3  52627   2631350 ICPanel
PSAPPSRV       APPQ             APPSRV         4  53003   2650150 ICPanel
PSAPPSRV       APPQ             APPSRV         5  50453   2522650 ICPanel
PSAPPSRV       APPQ             APPSRV         6  51482   2574100 ICPanel
PSAPPSRV       APPQ             APPSRV         7  52508   2625400 ICPanel
PSAPPSRV       APPQ             APPSRV         8  53105   2655250 ICPanel
...
```

This doesn't look healthy. All the application servers (PSAPPSRV) are busy running ICPanel. 

Lets have a look at the currently running SQL, and what it is doing?

```sql
SQL> select sid, serial#, seconds_in_wait, wait_class, event, SQL_ID, module,
 program, client_identifier
 from v$session
 where sql_id is not null and wait_class != 'Idle' order by 3;

SID SERIAL# SECONDS WAIT_CL EVENT         SQL_ID MODULE PROGRAM  CLIENT_ID
--- ------- ------- ------- ------------- ------ ------ -------- ---------
578   33663       0 Other   PGA memory op ax0w42 THING1 PSAPPSRV     user1
427    5955    1470 Other   PGA memory op 67bqun THING4 PSAPPSRV     user2
426   19542    1713 Other   PGA memory op 67bqun THING4 PSAPPSRV     user2
789   17224    1806 Other   PGA memory op 67bqun THING4 PSAPPSRV     user2
034   62386    2025 Other   PGA memory op 67bqun THING4 PSAPPSRV     user2
649   63585    2111 Other   PGA memory op 67bqun THING2 PSAPPSRV     user2
590   34797    2341 Network SQL*Net messa bn84n1 THING4 PSAPPSRV     user3
252   52586    2638 Other   PGA memory op 67bqun THING3 PSAPPSRV     user2
...

8 rows selected.
```

We can see that a number of PSAPPSRV processes are running module THING4, 
and the same SQL ID starting 67bqun.

## What Is Stuck?

Going back to the mental model of the flow of a session from the user, we can
see it gets as far as the web server, but when the web server asks the
application server for a session, the are busy. The session has to wait_class
in the web server. Indeed in the web server logs we were getting messages like:

```
[STUCK] ExecuteThread: '7' for queue: 'weblogic.kernel.Default (self-tuning)'
 has been busy for "601" seconds working on the request
 "Http Request Information: weblogic.servlet.internal.ServletRequestImpl@39d566d7
 [POST /psc/ps/EMPLOYEE/SA/c/THING/OTHERTHING.GBL]
", which is more than the configured time (StuckThreadMaxTime) of "600" seconds 
in "server-failure-trigger".
```

Armed with this information we can decide what to do to mitigate the problem
to buy ourselves time to fix it.


## Mitigation

To make the system work, I created a
script to kill all the sessions that were running the problem SQL:

```sql
select 'alter system kill session '''||
 sid||','||serial#||''';' 
 from v$session where sql_id = '67bqun92ngrsj';

'ALTERSYSTEMKILLSESSION'''||SID||','||SERIA
-------------------------------------------
alter system kill session '126,65311';
alter system kill session '138,7220';
alter system kill session '259,39219';
alter system kill session '368,52820';
alter system kill session '860,36870';
...

28 rows selected.
```

Then it is just a matter of running the output to kill the affected sessions. 
I know from experience that PeopleSoft can recover from this, it reconnects
to the database. Other systems might cope differently.

The admin logged in and switched off access to the affected page, and the system was working again.
The problem is of course that the most important process was the one we had denied access to.

## Next Steps

Having contained the problem we were now ready to address the issue with the SQL we had identified.
I will deal with that in [another post](../moresqltuning).