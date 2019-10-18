---
title: "Missing File"
date: 2019-10-18T11:09:47Z
tags: ["DBA","Fail","Oracle","Recovery","SQL"]
---

This is a post which was sitting in my drafts since the start of last year, but it still seems useful to me.

It's been a while since I have had a file that was deleted. What course to take depends on context - what do you want to achieve? 
In this case I wanted to remove the tablespace. I offlined all the files in the tablespace and deleted it. It is pretty easy really. 
The other thing I could have done is recovered the datafiles from the redo logs. Maybe I should try that another time.

I wanted to apply the critical patch, so I try to shut down the database prior to altering the oracle home.

{{< highlight sql >}}
$ sqlplus / as sysdba

SQL*Plus: Release 12.1.0.2.0 Production on Tue Jan 23 18:28:07 2018

Copyright (c) 1982, 2014, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production

SQL> shutdown immediate;
ORA-01115: IO error reading block from file 173 (block # 1)
ORA-01110: data file 173: '/WRONGPLACE/datafile.dbf'
ORA-27072: File I/O error
Additional information: 4
Additional information: 1
{{< /highlight >}}

Oh dear. It won't shut down. I know that my colleague was doing an experiment,
and doesn't need that tablespace any more. Lets have a look at what files were
created:

{{< highlight sql >}}
SQL> select file_name, tablespace_name from dba_data_files
  2  where file_name like '%WRONGPLACE%'
  3  /

FILE_NAME                   TABLESPACE_NAME
--------------------------- -----------------
/WRONGPLACE/datafile.dbf    TBSP1
/WRONGPLACE/datafile2.dbf   TBSP1
{{< /highlight >}}

There are a couple. Of course what I should have done is checked which files
were a part of the tablespace. We will see this later.

Let's try dropping the tablespace.

{{< highlight sql >}}
SQL> drop tablespace TBSP1 including contents and datafiles;
drop tablespace TBSP1 including contents and datafiles
*
ERROR at line 1:
ORA-01115: IO error reading block from file 173 (block # 1)
ORA-01110: data file 173: '/WRONGPLACE/datafile.dbf'
ORA-27072: File I/O error
Additional information: 4
Additional information: 1
{{< /highlight >}}

You can't. First the files need to be offlined. Even this can't be done
without telling Oracle that you are going to drop them.

{{< highlight sql >}}
SQL> alter database datafile '/WRONGPLACE/datafile.dbf' offline drop;

Database altered.

SQL> alter database datafile '/WRONGPLACE/datafile2.dbf' offline drop;

Database altered.

SQL> drop tablespace TBSP1 including contents and datafiles;
drop tablespace TBSP1 including contents and datafiles
*
ERROR at line 1:
ORA-01116: error in opening database file 175
ORA-01110: data file 175: '/ANOTHERPLACE/anotherfile.dbf'
ORA-27041: unable to open file
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3
{{< /highlight >}}

And that is why I should have checked which files were in the tablespace!
I missed one which was in another place.

{{< highlight sql >}}
SQL> alter database datafile 175 offline drop;

Database altered.

SQL> drop tablespace TBSP1 including contents and datafiles;

Tablespace dropped.

SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> exit
Disconnected from Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
{{< /highlight >}}

Note that I can use the file number instead of the file name to drop the data
file, which is useful, especially in situations where the file name is
blank. This was the last file left in the tablespace, so I could drop
the tablespace, shut down the database and carry on with what I was doing.

Oh yes, patching!
