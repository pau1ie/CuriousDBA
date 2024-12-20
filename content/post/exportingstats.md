---
title: "Exporting Statistics"
date: 2017-12-20T11:28:20Z
image: images/counters.JPG
tags: ["Oracle","DBA","Statistics","Performance","Export"]
---

This was surprisingly more difficult than I expected. We know that we can export stats
from the dictionary to a table, and from the table to a file, and that file can be copied
and imported to another database for the stats to be imported. Easy right?

{{< highlight sql >}}
$ sqlplus / as sysdba

SQL*Plus: Release 12.1.0.2.0 Production on Tue Dec 19 08:55:39 2017

Copyright (c) 1982, 2014, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production

SQL> exec DBMS_STATS.CREATE_STAT_TABLE('SCHEMA','MYSTATS','USERS');

PL/SQL procedure successfully completed.

SQL> @export_stats
BEGIN
DBMS_STATS.EXPORT_TABLE_STATS ( 'SCHEMA', 'MYTABLE', NULL, stattab => 'MYSTATS');
END;

*
ERROR at line 1:
ORA-20002: Version of statistics table "SCHEMA"."MYSTATS" is too old.  Please
try upgrading it with dbms_stats.upgrade_stat_table
ORA-06512: at "SYS.DBMS_STATS", line 18000
ORA-06512: at line 1

{{< /highlight >}}

Hang on, I just created it! How can it be too old? Still, I will upgrade it if it insists.

{{< highlight sql >}}

SQL> exec dbms_stats.upgrade_stat_table( 'SCHEMA','MYSTATS')

PL/SQL procedure successfully completed.

SQL> @export_stats
BEGIN
DBMS_STATS.EXPORT_TABLE_STATS ( 'SCHEMA', 'MYTABLE', NULL, stattab => 'MYSTATS');
END;

*
ERROR at line 1:
ORA-20002: Version of statistics table "SCHEMA"."MYSTATS" is too old.  Please
try upgrading it with dbms_stats.upgrade_stat_table
ORA-06512: at "SYS.DBMS_STATS", line 18000
ORA-06512: at line 1
{{< /highlight >}}

Desperate times call for desparate measures! Fire up Oracle support and a search turns up
document ID 2004828.1. This says I am encountering bug 20822264 or 19280897. The solution
is to create the table using NLS_LENGTH_SEMANTICS of BYTE rather than CHAR which our
database has by default. I gave it a try:

{{< highlight sql >}}
SQL> drop table SCHEMA.MYSTATS
  2  /

Table dropped.

SQL> alter session set nls_length_semantics=byte;

Session altered.

SQL> exec DBMS_STATS.CREATE_STAT_TABLE('SCHEMA','MYSTATS','USERS');

PL/SQL procedure successfully completed.

SQL> exec DBMS_STATS.EXPORT_TABLE_STATS('SCHEMA','MYTABLE',NULL,stattab => 'MYSTATS')

PL/SQL procedure successfully completed.
{{< /highlight >}}

Phew!

For completeness, here is what I did to export the table. I create the directory to use:

{{< highlight sql >}}
SQL> Create or replace directory exp_dir as '/path/to/dir';
SQL> grant read, write on directory exp_dir to schema;
{{< /highlight >}}

Then I export

{{< highlight console >}}
$ expdp SCHEMA directory=EXP_DIR tables=MYSTATS dumpfile=MYSTATS 

Export: Release 12.1.0.2.0 - Production on Tue Dec 19 09:19:53 2017

Copyright (c) 1982, 2014, Oracle and/or its affiliates.  All rights reserved.
Password: 

Connected to: Oracle Database 12c Enterprise Edition Release 12.1.0.2.0
  - 64bit Production
Starting "SCHEMA"."SYS_EXPORT_TABLE_01":  SCHEMA/********
  directory=EXP_DIR tables=MYSTATS dumpfile=MYSTATS 
Estimate in progress using BLOCKS method...
Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
Total estimation using BLOCKS method: 128 KB
Processing object type TABLE_EXPORT/TABLE/TABLE
Processing object type TABLE_EXPORT/TABLE/INDEX/INDEX
Processing object type TABLE_EXPORT/TABLE/INDEX/STATISTICS/INDEX_STATISTICS
Processing object type TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
Processing object type TABLE_EXPORT/TABLE/STATISTICS/MARKER
. . exported "SCHEMA"."MYSTATS"                         19.12 KB      12 rows
Master table "SCHEMA"."SYS_EXPORT_TABLE_01" successfully loaded/unloaded
******************************************************************************
Dump file set for SCHEMA.SYS_EXPORT_TABLE_01 is:
  /path/to/dir/MYSTATS.dmp
Job "SCHEMA"."SYS_EXPORT_TABLE_01" successfully completed at
   Tue Dec 19 09:27:04 2017 elapsed 0 00:06:58
{{< /highlight >}}
