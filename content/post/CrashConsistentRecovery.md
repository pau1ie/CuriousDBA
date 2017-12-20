---
title: "Crash Consistent Recovery"
image: images/crash.jpg
date: 2017-12-11T16:17:41Z
tags: ["Oracle","DBA","Backup","Recovery"]
---

Problem
---

Since Oracle 12c, you can recover a crash consistent snapshot. Oracle support note 604683.1 says how to do this.

We had an issue where the recovery wanted to effectively run to the end of time, and wouldn't ever finish.
No matter how many logs were applied, it said:

{{< highlight sql >}}
ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/CS_SR/system/system01.dbf'
{{< /highlight >}}


Solution
---

The solution is the snapshot time of the recovery clause. To demonstrate 
I have a database
being recovered from a snapshot, and here are some archived redo logs from the
source database.

{{< highlight console >}}
$ ls -ltr
total 5163140
-rw-r----- 1 oracle dba   10543104 Dec  8 13:12 65_1_962144139.arc
-rw-r----- 1 oracle dba    2151936 Dec  8 13:27 66_1_962144139.arc
-rw-r----- 1 oracle dba     946688 Dec  8 13:42 67_1_962144139.arc
-rw-r----- 1 oracle dba  106735104 Dec  8 13:57 68_1_962144139.arc
-rw-r----- 1 oracle dba  112962560 Dec  8 14:03 69_1_962144139.arc
-rw-r----- 1 oracle dba  196398592 Dec  8 14:18 70_1_962144139.arc
-rw-r----- 1 oracle dba  189703680 Dec  8 14:33 71_1_962144139.arc
-rw-r----- 1 oracle dba  197077504 Dec  8 14:48 72_1_962144139.arc
-rw-r----- 1 oracle dba  201095680 Dec  8 15:03 73_1_962144139.arc
-rw-r----- 1 oracle dba  170909184 Dec  8 15:18 74_1_962144139.arc
-rw-r----- 1 oracle dba  169005056 Dec  8 15:33 75_1_962144139.arc
-rw-r----- 1 oracle dba   16596480 Dec  8 15:35 76_1_962144139.arc
-rw-r----- 1 oracle dba     796160 Dec  8 15:35 77_1_962144139.arc
-rw-r----- 1 oracle dba    1107968 Dec  8 15:35 78_1_962144139.arc
-rw-r----- 1 oracle dba    1053696 Dec  8 15:35 79_1_962144139.arc
{{< /highlight >}}

Go in to sqlplus (It is version 12.1, so this should work)

{{< highlight console >}}
$ sqlplus / as sysdba

SQL*Plus: Release 12.1.0.2.0 Production on Mon Dec 11 14:43:39 2017

Copyright (c) 1982, 2014, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
{{< /highlight >}}
And run the recovery
{{< highlight sql >}}

SQL> recover database until cancel using backup controlfile snapshot time '08-DEC-2017 14:00:00';
ORA-00279: change 15280596683192 generated at 12/08/2017 13:42:02 needed for
thread 1
ORA-00289: suggestion : /CS_SR/archive/CS_SR_68_1_962144139.arc
ORA-00280: change 15280596683192 for thread 1 is in sequence #68


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
CANCEL
ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/CS_SR/system/system01.dbf'


ORA-01112: media recovery not started
{{< /highlight >}}

We can see that since we have recovered to a time before we told the
database that the snapshot was taken, it still wanted more recovery. Lets give
it some more recovery to beyond the snapshot time.

{{< highlight sql >}}


SQL> recover database until cancel using backup controlfile snapshot time '08-DEC-2017 14:00:00';
ORA-00279: change 15280596683192 generated at 12/08/2017 13:42:02 needed for
thread 1
ORA-00289: suggestion : /CS_SR/archive/CS_SR_68_1_962144139.arc
ORA-00280: change 15280596683192 for thread 1 is in sequence #68


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
/CS_SR/archive/68_1_962144139.arc
ORA-00279: change 15280596695470 generated at 12/08/2017 13:57:04 needed for
thread 1
ORA-00289: suggestion : /CS_SR/archive/CS_SR_69_1_962144139.arc
ORA-00280: change 15280596695470 for thread 1 is in sequence #69
ORA-00278: log file '/CS_SR/archive/68_1_962144139.arc' no longer needed for
this recovery


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
/CS_SR/archive/69_1_962144139.arc
ORA-00279: change 15280596705159 generated at 12/08/2017 14:03:41 needed for
thread 1
ORA-00289: suggestion : /CS_SR/archive/CS_SR_70_1_962144139.arc
ORA-00280: change 15280596705159 for thread 1 is in sequence #70
ORA-00278: log file '/CS_SR/archive/69_1_962144139.arc' no longer needed for
this recovery


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
CANCEL
Media recovery cancelled.
SQL> alter database open resetlogs;

Database altered.
{{</ highlight >}}

It seems that if you don't tell the database the snapshot time, it wants to recover for ever which was the problem we encountered.

Confusion
---

Some colleagues managed to create a database from a crash consistent snapshot.
After considering, we realised that these are from snapshots of a 
physical standby, i.e. a database  that isn't open, therefore hasn't
marked it's datafiles as fuzzy.
