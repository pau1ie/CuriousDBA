---
title: "Oracle Backup Restore Failures"
date: "2024-05-10T14:15:00+00:00"
tags: ['Backup', 'Automation', 'Database', 'DBA','Disaster Recovery', 'Oracle']
---

## How I Test My Backups

I like to test my backups. It helps me sleep to know I could get my data back if
the worst happened and it was scrambled by ransomware, or a bug in our code.

My sleep was rendered less peaceful when the restores suddenly started failing
for no reason that I could understand. We use RMAN to backup and restore
the data, and the script
is fairly simple - it effectively says to restore the database as it was at
noon yesterday. Something like this:

```bash
export NB_ORA_CLIENT=proddbserver
sqlplus / as sysdba <<-!
shutdown abort
create spfile from pfile;
startup nomount
!

rman <<-!
connect auxiliary /
connect catalog catuser@CATDB

run {
allocate auxiliary channel d1 type 'SBT_TAPE';
allocate auxiliary channel d2 type 'SBT_TAPE';
allocate auxiliary channel d3 type 'SBT_TAPE';
duplicate database 'PRODDB' to 'RESTDB' until
   time "to_date('2024-05-09 12:00','YYYY-MM-DD HH24:MI')";
}
!
```

Assuming that today is the 10th May, yesterday would have been the 9th. There is a
slight complication in that I have another database on this server called PRODDB,
so I have to alter the path names, but its relatively simple to set the relevant
parameters in the pfile:

```
*.DB_FILE_NAME_CONVERT = 'PRODDB','RESTDB'
*.LOG_FILE_NAME_CONVERT = 'PRODDB','RESTDB'
```


## What Went Wrong

This worked 
fine for years, but suddenly it started erroring as follows. Sorry, this is rather
a lot of output, it restores the control files OK, but then falls over because it
doesn't know what to call the datafiles. Scroll down for more...

```console
Recovery Manager: Release 19.0.0.0.0
   - Production on Wed May 8 12:03:06 2024 Version 19.23.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

connected to recovery catalog database
PL/SQL package CATUSER.DBMS_RCVCAT version 19.21.00.00.
  in RCVCAT database is not current
  PL/SQL package CATUSER.DBMS_RCVMAN version 19.21.00.00
  in RCVCAT database is not current
  connected to auxiliary database (not started)

RMAN> 
Oracle instance started

Total System Global Area   10737417432 bytes

Fixed Size                    13683928 bytes
Variable Size               5301600256 bytes
Database Buffers            5402263552 bytes
Redo Buffers                  19869696 bytes

RMAN> 2> 3> 4> 5> 6> 7> 
allocated channel: d1
channel d1: SID=849 device type=SBT_TAPE channel d1:
  Veritas NetBackup for Oracle - Release 8.3.0.1 (2020081919)

allocated channel: d2
channel d2: SID=970 device type=SBT_TAPE channel d2:
  Veritas NetBackup for Oracle - Release 8.3.0.1 (2020081919)

allocated channel: d3
channel d3: SID=1091 device type=SBT_TAPE channel d3:
  Veritas NetBackup for Oracle - Release 8.3.0.1 (2020081919)

Starting Duplicate Db at 2024-May-08 12:03:26

contents of Memory Script:
{
   set until scn  41106151363386;
   sql clone "alter system set  db_name =  ''PRODDB''
      comment=  ''Modified by RMAN duplicate'' scope=spfile";
   sql clone "alter system set  db_unique_name =  ''RESTDB''
      comment=  ''Modified by RMAN duplicate'' scope=spfile";
   shutdown clone immediate;
   startup clone force nomount
   restore clone primary controlfile;
   alter clone database mount;
}
executing Memory Script

executing command: SET until clause

sql statement: alter system set  db_name =  ''PRODDB''
    comment= ''Modified by RMAN duplicate'' scope=spfile

sql statement: alter system set  db_unique_name =  ''RESTDB''
    comment= ''Modified by RMAN duplicate'' scope=spfile

Oracle instance shut down

Oracle instance started

Total System Global Area   10737417432 bytes

Fixed Size                    13683928 bytes
Variable Size               5301600256 bytes
Database Buffers            5402263552 bytes
Redo Buffers                  19869696 bytes
allocated channel: d1
channel d1: SID=849 device type=SBT_TAPE channel d1:
  Veritas NetBackup for Oracle - Release 8.3.0.1 (2020081919)
  allocated channel: d2 
  channel d2: SID=970 device type=SBT_TAPE channel d2:
  Veritas NetBackup for Oracle - Release 8.3.0.1 (2020081919)
  allocated channel: d3
  channel d3: SID=1091 device type=SBT_TAPE channel d3:
  Veritas NetBackup for Oracle - Release 8.3.0.1 (2020081919)

Starting restore at 2024-May-08 12:04:01

channel d1: starting datafile backup set restore channel d1:
  restoring control file
  channel d1: reading from backup piece c-799314582-20240507-01
  channel d1: piece handle=c-799314582-20240507-01 tag=TAG20240507T100515
  channel d1: restored backup piece 1
  channel d1: restore complete, elapsed time: 00:00:15
  output file name=/RESTDB/control01/control1.ctl
output file name=/RESTDB/control02/control2.ctl
output file name=/RESTDB/control03/control3.ctl
output file name=/RESTDB/control04/control4.ctl
Finished restore at 2024-May-08 12:04:17

database mounted
Oracle instance started

Total System Global Area   10737417432 bytes

Fixed Size                    13683928 bytes
Variable Size               5301600256 bytes
Database Buffers            5402263552 bytes
Redo Buffers                  19869696 bytes

contents of Memory Script:
{
   sql clone "alter system set  db_name =  ''RESTDB''
     comment=  ''Reset to original value by RMAN'' scope=spfile";
   sql clone "alter system reset  db_unique_name scope=spfile";
   shutdown clone immediate;
}
executing Memory Script

sql statement: alter system set  db_name =  ''RESTDB''
   comment= ''Reset to original value by RMAN'' scope=spfile

sql statement: alter system reset  db_unique_name scope=spfile

Oracle instance shut down
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of Duplicate Db command at 05/08/2024 12:05:02
RMAN-05501: aborting duplication of target database
RMAN-05624: data file name not found in the repository
  for data file number=222
RMAN-05624: data file name not found in the repository
  for data file number=221
RMAN-05624: data file name not found in the repository
  for data file number=220
```

Sorry that was so long! OK, so RMAN has gone from being
able to take the file names of the production database server to not being
able to, and for no reason.


## Oracle Service Request

We have Oracle Support! They will be able to help! I raised a service
request with
Oracle, and they pointed me at 
[Doc ID 2704529.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2704529.1),
which basically says the script I have successfully been running 
for years is impossible. You have to specify a
`set newname` command in the run block. So my RMAN command becomes:

```bash
rman <<-!
connect auxiliary /
connect catalog catuser@CATDB

run {
allocate auxiliary channel d1 type 'SBT_TAPE';
allocate auxiliary channel d2 type 'SBT_TAPE';
allocate auxiliary channel d3 type 'SBT_TAPE';
set newname for database to '/RESTDB/data/%f_%b';
duplicate database 'PRODDB' to 'RESTDB' until
   time "to_date('2024-05-09 12:00','YYYY-MM-DD HH24:MI')";
}
!
```
The problem with this is that it doesn't work! I will spare you the
entire output this time, after restoring the controlfile it errors
again.

```console
RMAN-08161: contents of Memory Script:
{
   sql clone "alter system set  db_name =
 ''CS_HESA'' comment=
 ''Reset to original value by RMAN'' scope=spfile";
   sql clone "alter system reset  db_unique_name scope=spfile";
   shutdown clone immediate;
}
RMAN-08162: executing Memory Script

RMAN-06162: sql statement: alter system set  db_name =  ''CS_HESA''
   comment= ''Reset to original value by RMAN'' scope=spfile

RMAN-06162: sql statement: alter system reset  db_unique_name scope=spfile

RMAN-06402: Oracle instance shut down
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of Duplicate Db command at 05/09/2024 14:04:24
RMAN-05501: aborting duplication of target database
RMAN-06136: Oracle error from auxiliary database:
  ORA-19715: invalid format b for generated name
ORA-27302: failure occurred at: slgpn
```

I can restore with just the file number using `set newname '/RESTDB/data/%f'`,
and that does restore, but all my database files
end up in the same directory, and they are all named with numbers.
So they don't have my lovely naming convention where I group them
into folders and name them based on the table space name.
I like being able to restore the database. This is great! But I
also like seeing what my files are for!


## The Real Problem - And The Solution

Looking at the RMAN repository I can see that there is some weirdness:


```bash
$ rman target /

Recovery Manager: Release 19.0.0.0.0
  - Production on Thu May 9 17:33:13 2024
Version 19.23.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

connected to target database: PRODDB (DBID=799314582)

RMAN> connect catalog catuser@CATDB

recovery catalog database Password: 
connected to recovery catalog database
PL/SQL package CATUSER.DBMS_RCVCAT version 19.21.00.00.
  in RCVCAT database is not current
PL/SQL package CATUSER.DBMS_RCVMAN version 19.21.00.00
  in RCVCAT database is not current

RMAN> list db_unique_name of database;


List of Databases
DB Key  DB Name  DB ID            Database Role    Db_unique_name
------- ------- ----------------- ---------------  ------------------
56747359 PRODDB   799314582        PRIMARY          PRODDB_PRODHOST    
56747359 PRODDB   799314582        STANDBY          RESTDB             

```

Well, this is weird, my test restore database has been registered with the same DBID as
production. RMAN seems to think it is a standby, which it is not. How did this happen?
No idea. Let's try unregistering it:

```bash
RMAN> unregister DB_UNIQUE_NAME RESTDB;

database db_unique_name is "RESTDB", db_name is "PRODDB" and DBID is 799314582

Want to unregister the database with target db_unique_name (enter YES or NO)? YES
database with db_unique_name RESTDB unregistered from the recovery catalog

RMAN>  list db_unique_name of database;


List of Databases
DB Key  DB Name  DB ID            Database Role    Db_unique_name
------- ------- ----------------- ---------------  ------------------
56747359 PRODDB   799314582        PRIMARY          PRODDB_PRODHOST    

RMAN> 
```

That looks much better. And a test restore works fine. I have not been able to
find this documented anywhere. There is nothing in the Oracle Support knowledge
base. So I thought I would put this out there in the hope it helps someone.
Especially since that someone might be me in the future!
