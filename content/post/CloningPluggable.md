---
title: "Cloning a Pluggable Database using Unix commands"
date: 2020-10-15T10:22:49+01:00
tags: ["Backup","Database","DBA","DR","Oracle","Recovery"]
---

I read a lot about the flexibility of Oracle commands for pluggable databases.
I haven't seen as much about the old fashioned way to copy data files around and
manually creating a control file. So lets see if that still works. I have a
campus solutions demo instance, lets see if I can copy it and rename it.

## Back Up and Edit the Control File

Running a familiar command is promising:

```sql
alter database backup controlfile to trace;
```

The create control file command is generated as expected.
I deleted _case 1 - noresetlogs_, and edited _case 2_ to change the database
name:
```sql
STARTUP NOMOUNT
CREATE CONTROLFILE REUSE set DATABASE "NEWCONT" RESETLOGS  NOARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
...
```

I can see the data files are stored under `/opt/oracle/db/oradata/DBNAME`
where `DBNAME` is the name of the database. The container is `CDBCS`, then
there is the pluggable database at the same level. So I decided to copy all the
files but rename
the directories for the database I am going to create, but still keep the same
structure. I have a habit, when
running the create control file command by hand, of checking the files are
all correct using the following:

```bash
ls $( grep \' cont.sql | sed 's/--*//' | cut -f2 -d\' )
```

I also check there are no files missing from the controlfile using:

```bash
find /d16/MYDB -type f |while read aline 
do 
 if grep -q $aline cont.sql
 then
  :
 else
  echo $aline missing
 fi
done
```

This returns a couple of lines buy they aren't data files, so I am happy to run
the create control file command.

## Creating the New Database
 First I need a pfile. The `create 
pfile from spfile` command still works, so the init.ora can be used as a basis.
Once the spfile has been sorted out, we can create the control file.

```sql
startup nomount

CREATE CONTROLFILE SET ...

Control file created.

ORA-00279: change 208737729 generated at 09/16/2020 14:31:19 needed for thread 1
ORA-00289: suggestion : ...
ORA-00280: change 208737729 for thread 1 is in sequence #278
```

Oh dear - it seems we need recovery. I haven't worked out why this happens when
the database seems to have been shut down cleanly, but I fix it by pointing
the recovery process at the online redo logs.

So I use the command that is seared onto my brain:

```sql
SQL> recover database until cancel using backup controlfile;
ORA-00279: change 208737729 generated at 09/16/2020 14:31:19 needed for thread 1
ORA-00289: suggestion :
 ...
ORA-00280: change 208737729 for thread 1 is in sequence #278


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
/path/to/redo02.log
Log applied.
Media recovery complete.
```

I chose the online redo log with the latest modification time. I can try the
others if that doesn't work, but it seems to.

So now we can open the database...

```sql
SQL> ALTER DATABASE OPEN RESETLOGS;

Database altered.
```

And the pluggable databases.

```sql
SQL> ALTER PLUGGABLE DATABASE ALL OPEN;

Pluggable database altered.
```

I can add in the temp files to the container and seed databases. These
commands are copied and pasted directly from the generated create controlfile SQL.

```sql
ALTER TABLESPACE TEMP ADD TEMPFILE '/path/to/temp01.dbf'
     SIZE 46137344  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;

Tablespace altered.

ALTER SESSION SET CONTAINER = "PDB$SEED";

Session altered.

ALTER TABLESPACE TEMP ADD TEMPFILE '/path/to/pdbseed/temp012020-07-31.dbf'
     SIZE 37748736  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;

Tablespace altered.

ALTER SESSION SET CONTAINER = "CS92U018";

ALTER TABLESPACE TEMP ADD TEMPFILE '/path/to/temp01.dbf'
     SIZE 81788928  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;

Tablespace altered.

ALTER TABLESPACE PSTEMP ADD TEMPFILE '/path/to/pstemp01.dbf'
     SIZE 943718400  REUSE AUTOEXTEND ON NEXT 10485760  MAXSIZE 5024M;

Tablespace altered.

ALTER TABLESPACE PSGTT01 ADD TEMPFILE '/path/to/psgtt01.dbf'
     SIZE 524288000  REUSE AUTOEXTEND ON NEXT 5242880  MAXSIZE 32767M;

Tablespace altered.

ALTER SESSION SET CONTAINER = "CDB$ROOT";

Session altered.
```

## Changing the DBID

At this point I want to change the DBID of the database
if I was going to use it for anything serious, like ever backing it up.

```sql
shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount
ORACLE instance started.

Total System Global Area 1207955592 bytes
Fixed Size                  9134216 bytes
Variable Size             369098752 bytes
Database Buffers          805306368 bytes
Redo Buffers               24416256 bytes
Database mounted.
```

```console
$ nid target=/ pdb=all

DBNEWID: Release 19.0.0.0.0 - Production on Wed Oct 7 14:46:57 2020

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Connected to database MYCONT (DBID=897177241)

Connected to server version 19.8.0

Control Files in database:
    /path/to/control01.ctl

Change database ID of database MYCONT? (Y/[N]) => Y

Proceeding with operation
Changing database ID from 897177241 to 3350210145
    Control File /path/to/control01.ctl - modified
    Datafile /path/to/system01.db - dbid changed
...
    Datafile /path/to/pdbseed/temp012020-07-31_15-00-20-947-PM.db - dbid changed
    Datafile /path/to/temp01.db - dbid changed
    Datafile /path/to/pstemp01.db - dbid changed
    Datafile /path/to/psgtt01.db - dbid changed
    Control File /path/to/control01.ctl - dbid changed
    Instance shut down

Database ID for database MYCONT changed to 3350210145.
All previous backups and archived redo logs for this database are unusable.
Database is not aware of previous backups and archived logs in Recovery Area.
Database has been shutdown, open database with RESETLOGS option.
Succesfully changed database ID.
DBNEWID - Completed succesfully.
```

So now we start it with resetlogs

```sql
SQL> startup mount
ORACLE instance started.

Total System Global Area 1207955592 bytes
Fixed Size                  9134216 bytes
Variable Size             369098752 bytes
Database Buffers          805306368 bytes
Redo Buffers               24416256 bytes
Database mounted.
SQL> alter database open resetlogs;

Database altered.

SQL> ALTER PLUGGABLE DATABASE ALL OPEN;
```

## Renaming the Pluggable Database

So let's explore.

```sql
col name for a14
col network_name for a14
col pdb for a14
set lines 132
set pages 50000

SQL> select name, open_mode, restricted from v$pdbs;

NAME         OPEN_MODE  RES
------------ ---------- ---
PDB$SEED     READ ONLY  NO
CS92U018     READ WRITE NO

SQL> select name, con_id, dbid,con_uid,guid from v$containers order by con_id;

NAME             CON_ID       DBID    CON_UID GUID
------------ ---------- ---------- ---------- --------------------------------
CDB$ROOT              1  897177241          1 86B637B62FDF7A65E053F706E80A27CA
PDB$SEED              2 1477469430 1477469430 ABBE35A910D454B9E0531441000A10F0
CS92U018              3 2904156531 2904156531 ABBE617FD19267BEE0531441000A7FBB

SQL> select service_id,name,network_name,creation_date,pdb,con_id from cdb_services;

SERVICE_ID NAME           NETWORK_NAME   CREATION_ PDB                CON_ID
---------- -------------- -------------- --------- -------------- ----------
         1 SYS$BACKGROUND                17-APR-19 CDB$ROOT                1
         2 SYS$USERS                     17-APR-19 CDB$ROOT                1
         3 MYCONT         MYCONT         17-SEP-20 CDB$ROOT                1
         5 CDBCSXDB       CDBCSXDB       31-JUL-20 CDB$ROOT                1
         6 CDBCS          CDBCS          31-JUL-20 CDB$ROOT                1
         8 cs92u018       cs92u018       31-JUL-20 CS92U018                3
```

I want to rename the pluggable database, I don't want to keep the CS92U018 name
I copied from the image DB.

To do this I put the PDB in RESTRICTED mode for a rename operation:

```sql
SQL> alter pluggable database CS92U018 close;

Pluggable database altered.

SQL> alter pluggable database CS92U018 open restricted;

Pluggable database altered.

SQL> select name, open_mode, restricted from v$pdbs; 

NAME         OPEN_MODE  RES
------------ ---------- ---
PDB$SEED     READ ONLY  NO
CS92U018     READ WRITE YES
```

Connect to the PDB and rename it:

```sql
SQL> alter session set container=CS92U018;

Session altered.

SQL> alter pluggable database rename global_name to MYPDB;

Pluggable database altered.
```

It is important to restart the DB at this point so Oracle can update the metadata.

```sql
SQL> alter pluggable database close immediate;

Pluggable database altered.

SQL> alter pluggable database open;

Pluggable database altered.

SQL> ALTER SESSION SET CONTAINER = "CDB$ROOT";

Session altered.
```
Now check what happened

```sql
SQL> select service_id,name,network_name,creation_date,pdb,con_id from cdb_services;
NAME         OPEN_MODE  RES
------------ ---------- ---
PDB$SEED     READ ONLY  NO
MYPDB        READ WRITE NO

SQL> select name, con_id, dbid,con_uid,guid from v$containers order by con_id;
NAME             CON_ID       DBID    CON_UID GUID
------------ ---------- ---------- ---------- --------------------------------
CDB$ROOT              1  897177241          1 86B637B62FDF7A65E053F706E80A27CA
PDB$SEED              2 1477469430 1477469430 ABBE35A910D454B9E0531441000A10F0
MYPDB                 3 2904156531 2904156531 ABBE617FD19267BEE0531441000A7FBB


SQL> select service_id,name,network_name,creation_date,pdb,con_id from cdb_services

SERVICE_ID NAME           NETWORK_NAME   CREATION_ PDB                CON_ID
---------- -------------- -------------- --------- -------------- ----------
         1 SYS$BACKGROUND                17-APR-19 CDB$ROOT                1
         2 SYS$USERS                     17-APR-19 CDB$ROOT                1
         3 MYCONT         MYCONT         17-SEP-20 CDB$ROOT                1
         5 CDBCSXDB       CDBCSXDB       31-JUL-20 CDB$ROOT                1
         6 CDBCS          CDBCS          31-JUL-20 CDB$ROOT                1
         1 MYPDB          MYPDB          17-SEP-20 MYPDB                   3

6 rows selected.
```

## Conclusion

Even with the new technology of pluggable databases, we can still use the old
methods of shutting the database down, copying it, and recreating the controlfile
to create a copy. We can also rename the pluggable database to suit our needs.
 