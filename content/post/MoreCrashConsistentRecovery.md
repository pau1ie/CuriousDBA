---
date: 2018-01-23T10:45:57Z
title: "More Crash Consistent Recovery"
image: images/crash.jpg
tags: ["Oracle","DBA","Backup","Recovery"]
---

Different Scenarios
---

In my [previous post](../crashconsistentrecovery) on this topic, I noted that you could use the snapshot time
on the recover database command to recover the database from a SAN snapshot.

I realise that there are different possible scenarios and my write up wasn't clear
on which approach is applicable when. Also, the test I did was unrealistic as I 
used the logs after the snapshot was taken, and the whole point of using 
SAN snapshots is that they contain everything required for a crash consistent
recovery.

### Start up in place ###

If you had a crash consistent snapshot, this can be mounted in the place of the
original and started. Oracle will recover as if the database had crashed, and
all is well. Easy!

### Start up a copy ###

More often, a table will be accidentally corrupted, and we will be asked to
restore it from the snapshot, but we want to keep the rest of the data in the
database is good, we want to leave that as it is. This is more tricky, and is
where the new command comes in.

Here it is really useful to have the create controlfile command. It is possible
to construct it from the files on the snapshot, but this is fiddly. It would 
be worth having a cron job to periodically issue:

{{< highlight sql >}}
alter database backup controlfile to trace as '/filesystem/CONTROLFILE.sql' resetlogs;
{{< /highlight >}}

The main thing is to have all the files.

Edit the controlfile as follows:

* Change the paths of the files if the filesystem is mounted in a different place
* Change the name of the database
* Insert the word SET to change the name of the database in the file headers.

So the first line of the create controlfile command looks like this:

{{< highlight sql >}}
CREATE CONTROLFILE REUSE SET DATABASE "CSRECO" RESETLOGS FORCE LOGGING ARCHIVELOG
{{< /highlight >}}

Now, I have a healthy paranoia, such that before running this command I check
the files in the create controlfile command actually exist like this:

{{< highlight bash >}}
grep \' CONTROLFILE.sql | cut -f2 -d\' | while read aline
do
 if ls $aline;
 then
  :
 else
  echo $aline missing;
 fi
done
{{< /highlight >}}

I also check the files on the filesystem are in the create controlfile command

{{< highlight bash >}}
find . -type f -name \*.dbf | while read aline
do
 if grep $aline CONTROLFILE.sql
 then
  echo $aline found
 else
  echo '****' $aline missing
 fi
done
{{< /highlight >}}

Now I am pretty confident in the commands. Ensure an init,ora exists for the database,
and is correct, then run the controlfile command. This will start up the database, create
the controlfile, but fail to recover because it doesn't know where the redo is.

{{< highlight sql >}}
SQL> @CONTROLFILE
ORACLE instance started.

Total System Global Area 2147483648 bytes
Fixed Size                  3712904 bytes
Variable Size            1694500984 bytes
Database Buffers          436207616 bytes
Redo Buffers               13062144 bytes


Control file created.

ORA-00279: change 15295737437889 generated at 01/19/2018 12:02:30 needed for
thread 1
ORA-00289: suggestion : /CSRECO/archive/CSRECO_80_1_963178734.arc
ORA-00280: change 15295737437889 for thread 1 is in sequence #80


ORA-00308: cannot open archived log '/CSRECO/archive/CSRECO_80_1_963178734.arc'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3


Database altered.


Database altered.

ALTER DATABASE OPEN RESETLOGS
*
ERROR at line 1:
ORA-01113: file 1 needs media recovery
ORA-01110: data file 1: '/CSRECO/system/system01.dbf'


ALTER TABLESPACE TEMP ADD TEMPFILE '/CSRECO/temp/temp01.dbf'
*
ERROR at line 1:
ORA-01109: database not open
{{< /highlight >}}

Next I made an interesting mistake. If the snapshot time is too early, the database
will tell you. If this happens you can add a second until it works. Or in my case, just
specify the correct year!

{{< highlight sql >}}
SQL> recover database until cancel using backup controlfile snapshot time '19-JAN-2017 13:19:01';
ORA-00283: recovery session canceled due to errors
ORA-19839: snapshot datafile checkpoint time is greater than snapshot time
ORA-01110: data file 1: '/CSRECO/system/system01.dbf'
{{< /highlight >}}

The database needs to use the online redo log from the old database and it doesn't
know about this because of the new controlfile. To prove it needs recovery from the redo log I will
try to cancel it.

{{< highlight sql >}}
SQL> recover database until cancel using backup controlfile snapshot time '19-JAN-2018 13:19:01';
ORA-00279: change 15295737437889 generated at 01/19/2018 12:02:30 needed for
thread 1
ORA-00289: suggestion : /CSRECO/archive/CSRECO_80_1_963178734.arc
ORA-00280: change 15295737437889 for thread 1 is in sequence #80


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
CANCEL
ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/CSRECO/system/system01.dbf'


ORA-01112: media recovery not started
{{< /highlight >}}

So, I specify the latest online redo log, and it recovers OK.

{{< highlight sql >}}
SQL> recover database until cancel using backup controlfile snapshot time '19-JAN-2018 13:19:01';
ORA-00279: change 15295737437889 generated at 01/19/2018 12:02:30 needed for
thread 1
ORA-00289: suggestion : /CSRECO/archive/CSRECO_80_1_963178734.arc
ORA-00280: change 15295737437889 for thread 1 is in sequence #80


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
/CSRECO/flash/CS_OPS/onlinelog/o1_mf_4_f3m1s2y3_.log
Log applied.
Media recovery complete.

{{< /highlight >}}

So now I should be able to open the database resetlogs and all will be fine right?

{{< highlight sql >}}
SQL> alter database open resetlogs;                                                                                         
alter database open resetlogs
*
ERROR at line 1:
ORA-00344: unable to re-create online log
'/CSRECO/flash/CSRECO/onlinelog/o1_mf_4_%u_.log'
ORA-27044: unable to write the header block of file
Linux-x86_64 Error: 28: No space left on device
Additional information: 3


SQL> !df -h /CSRECO
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/vold42p1  1.5T  1.5T  2.5G 100% /d48
{{< /highlight >}}

Oh dear. I have run out of space on the filesystem. I will just clear down
the old archive redo logs and the flash recovery area, and try again.

{{< highlight sql >}}
QL> alter database open resetlogs;     
alter database open resetlogs
*
ERROR at line 1:
ORA-00392: log 4 of thread 1 is being cleared, operation not allowed
ORA-00312: online log 4 thread 1:
'/CSRECO/flash/CSRECO/onlinelog/o1_mf_4_%u_.log'
{{< /highlight >}}

Looking on the internet it seems that because the open resetlogs failed
the database is in a confused state. Lets try clearing the log by hand.

{{< highlight sql >}}
SQL> alter database clear logfile group 4; 

Database altered.

SQL> alter database open resetlogs;            

Database altered.

SQL> 
{{< /highlight >}}

Phew, that worked! It is probably worth having enough space for this though!


