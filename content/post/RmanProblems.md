---
title: "Recovery Manager Problems"
date: 2018-01-19T11:41:13Z
tags: ["Oracle","DBA","Backup","Recovery","Rman"]
image: images/recovery.png
---

Lots of people seem to like Oracles Recovery manager. I am not one of them.
I think this is because of a lack of understanding on my part of how it works. It is
a complex beast, and at the same time has some annoying limitations.

I like to automate things. I have a number of scripts to call RMAN to
do backups and restores in common situations. These fail far too often
for my liking. I feel I should look into why. Maybe I can learn to love
RMAN? We shall see.

We got the following error:

### RMAN-06569: DATABASE: PROD does not match previous DATABASE: TEST

This is because we have done a clone of production to test using a
SAN snapshot, or by copying the files, then a create control file
command. Then we forget to run a nid, or else a backup kicks in 
before we get round to it.

The next time you try to do a duplicate, it gets confused as to which
database is being used. To fix this, you need to log in to
production, and unregister the database:

Logging into the catalogue we can see that it has taken TEST as a standby
for PROD:

{{< highlight console >}}
RMAN> connect catalog user/password@CATALOG

connected to recovery catalog database

RMAN> list db_unique_name of database PROD;


List of Databases
DB Key  DB Name  DB ID           Database Role    Db_unique_name
------- ------- ---------------- ---------------  ------------------
10796359 PROD   2538967823       PRIMARY          PROD             
10796359 PROD   2538967823       STANDBY          TEST            
10796359 PROD   2538967823       STANDBY          STANDBY1             
10796359 PROD   2538967823       STANDBY          STANDBY2  

{{< /highlight >}}

The solution to this is to log into RMAN from the primary PROD database, and
unregister the test database that was registered erroneously by the backups.

{{< highlight console >}}
$ rman target /

Recovery Manager: Release 12.1.0.2.0 - Production on Tue Jan 2 16:40:34 2018

Copyright (c) 1982, 2014, Oracle and/or its affiliates.  All rights reserved.

connected to target database: PROD (DBID=2538967823)

RMAN> connect catalog user/password@CATALOG

connected to recovery catalog database

RMAN> unregister db_unique_name 'TEST';

database db_unique_name is "TEST", db_name is "PROD" and DBID is 2538967823

Want to unregister the database with target db_unique_name (enter YES or NO)? YES
database with db_unique_name TEST unregistered from the recovery catalog
{{< /highlight >}}

Then if we list the databases, we get a tidy list, and more importantly, the
duplicate works.

{{< highlight console >}}
RMAN> list db_unique_name of database PROD;


List of Databases
DB Key  DB Name  DB ID           Database Role    Db_unique_name
------- ------- ---------------- ---------------  ------------------
10796359 PROD   2538967823       PRIMARY          PROD             
10796359 PROD   2538967823       STANDBY          STANDBY1
10796359 PROD   2538967823       STANDBY          STANDBY2
{{< /highlight >}}

It is also possible to unregister a database using a PL/SQL call:

{{< highlight console >}}
$ sqlplus

SQL*Plus: Release 12.1.0.2.0 Production on Mon Jan 8 16:58:16 2018

Copyright (c) 1982, 2014, Oracle.  All rights reserved.

Enter user-name: user/password@CATALOG
Last Successful login time: Mon Jan 08 2018 16:57:33 +00:00

Connected to:
Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
With the Partitioning option

SQL> exec dbms_rcvcat.unregisterdatabase(35409171,3917556746);

PL/SQL procedure successfully completed.

SQL> Disconnected from Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
With the Partitioning option
{{< /highlight >}}

I found after this that the newer incarnation was also unregistered from the
catalogue, so I registered it again.

{{< highlight console >}}
$ rman target / 

Recovery Manager: Release 12.1.0.2.0 - Production on Mon Jan 8 17:16:44 2018

Copyright (c) 1982, 2014, Oracle and/or its affiliates.  All rights reserved.

connected to target database: TEST (DBID=3919541964)

RMAN> connect catalog user/password@CATALOG

connected to recovery catalog database

RMAN> list backup of database;

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of list command at 01/08/2018 17:17:10
RMAN-06004: ORACLE error from recovery catalog database: RMAN-20001: target database not found in recovery catalog

RMAN> register database;

database registered in recovery catalog
starting full resync of recovery catalog
full resync complete

RMAN> list backup of database;


List of Backup Sets
===================


BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
36124190 Incr 0  8.75M      SBT_TAPE    00:00:07     01-JAN-18      
        BP Key: 36126997   Status: AVAILABLE  Compressed: NO  Tag: HOT_DB_BK_LEVEL0
        Handle: bk_1670_1_964234591   Media: @aaaab
  List of Datafiles in backup set 36124190
  File LV Type Ckp SCN    Ckp Time  Name
  ---- -- ---- ---------- --------- ----
  105  0  Incr 15297541132164 01-JAN-18 /TEST/filename/file.dbf

...
{{< /highlight >}}



But, to make sure this doesn't happen again, we need to make sure 
that after a clone that isn't done by RMAN, we use the nid tool
to give the database a new database identifier (DBID), as follows:

{{< highlight bash >}}
ORACLE_SID=TEST
ORAENV_ASK=NO

. oraenv

# Nid needs the database in mount mode
sqlplus / as sysdba <<-!
shutdown immediate;
startup mount
exit
!

# Give the database a new DBID
nid TARGET=/ LOGFILE=nid_$ORACLE_SID.log
cat nid_$ORACLE_SID.log

#nid leaves the database down. Start it up
# and open resetlogs
sqlplus / as sysdba <<-!
startup mount
alter database open resetlogs;
shutdown immediate;
startup
exit
!

# Lastly, register the database in the recovery
# catalogue.
rman target / <<-!
connect catalog user/password@CATALOG
register database;
exit
!
{{< /highlight >}}

The real solution is ensuring this is run faithfully after any
clone done without the use of RMAN. 
