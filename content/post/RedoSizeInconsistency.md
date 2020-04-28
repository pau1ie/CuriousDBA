---
title: "Redo Inconsistency"
date: 2020-04-28T11:03:50Z
tags: ['Oracle', 'DBA','Performance','SQL','Tuning']
---

# Redo Investigation

I was investigating a performance issue on new hardware and installed 
[SLOB](https://kevinclosson.net/2012/02/06/introducing-slob-the-silly-little-oracle-benchmark/)
to do so.
I discovered in the AWR report, that the redo generated per transaction was around four times
that in development.

I checked everything I could think of, including copying the parameter file from development to 
the new hardware and using that, but the redo stayed stubbornly high. I 
[asked the oracle-l mailing list for ideas](https://www.freelists.org/post/oracle-l/Redo-per-transaction-inconsistency-when-running-SLOB).
[Jonathan Lewis](https://jonathanlewis.wordpress.com/)
responded [suggesting the high redo could be
caused by the following](https://www.freelists.org/post/oracle-l/Redo-per-transaction-inconsistency-when-running-SLOB,7):

1. private redo disabled - so no "large" redo entries generated in private
2. some quirky little bug when auditing was enabled 
3. a couple of features that change "update" into "select for update/ update" 
4. trigger declarations - even NULL ones 
5. supplemental logging

The last option caught my eye because I know we have it enabled in production for the logical standby.

## Dumping Logs

I decided to try something really simple, so I could understand it. I created a table:

```sql
create table mytest ( id number, descr varchar2(4000) );
```

Then ran the following from a script so on a quiet system nothing else ran between the
two system change numbers (SCNs):

```sql
connect me
!date
set lines 132
set pages 50000
col current_scn for 999999999999999
select current_scn from v$database;
insert into me.mytest values (1,'Some handy text to insert and use up space');
commit;
select class, name, value 
  from v$mystat m, v$statname n
  where m.statistic# = n.statistic# and value > 0
  order by class;
!date
select current_scn from v$database;
exit
```

I could compare the results of `v$mystat`, and also since I had the SCNs, I could dump the redo for this period.
Jonathan Lewis had [pointed me](https://www.freelists.org/post/oracle-l/Redo-per-transaction-inconsistency-when-running-SLOB,9)
at [one of his blog posts](https://jonathanlewis.wordpress.com/2019/06/11/redo-dumps/), 
but this is also documented under Oracle Support note 1031381.6

```sql
alter system dump redo scn min 1234567890 scn max 1234567899;
```

This ends up in the trace file for the session. Since we are only dumping one update, it is much smaller than I expected.
The database with supplemental logging switched on had more information in it. It would probably have been better to do an
update on a table that had more columns because then the supplemental logging would be more verbose, as it
would have to include the values of all columns, unless there was a unique index.

## Asking the Database

Another way to understand the supplemental logging is to ask the database as follows.
The first SQL tells me that supplemental logging is switched on at the database level
to enable the database to uniquely identify columns.

```console
SQL> set lines 132
SQL> set pages 50000
SQL> col owner for a20
SQL> col log_group_name for a20
SQL> col table_name for a32
SQL> col column_name for a32
SQL> select supplemental_log_data_fk, 
            supplemental_log_data_all, 
            supplemental_log_data_pk, 
            supplemental_log_data_ui, 
            supplemental_log_data_min
     from v$database;

SUP SUP SUP SUP SUPPLEME
--- --- --- --- --------
NO  NO  YES YES IMPLICIT

SQL> select OWNER, LOG_GROUP_NAME, TABLE_NAME, always, generated, log_group_type
     from dba_log_groups;

OWNER             LOG_GROUP_NAME   TABLE_NAME ALWAYS GENERATED LOG_GROUP_TYPE
----------------- ---------------- ---------- ------ --------- --------------
SYS               SEQ$_LOG_GRP     SEQ$       ALWAYS USER NAME USER LOG GROUP
SYS               ENC$_LOG_GRP     ENC$       ALWAYS USER NAME USER LOG GROUP
GSMADMIN_INTERNAL SHARD_TS$LOG_GRP SHARD_TS   ALWAYS USER NAME USER LOG GROUP

SQL> select owner, log_group_name, table_name, column_name, logging_property
     from dba_log_group_columns;

OWNER             LOG_GROUP_NAME   TABLE_NAME COLUMN_NAME     LOGGIN
----------------- ---------------- ---------- --------------- ------
SYS               SEQ$_LOG_GRP     SEQ$       OBJ#            LOG
SYS               ENC$_LOG_GRP     ENC$       OBJ#            LOG
SYS               ENC$_LOG_GRP     ENC$       OWNER#          LOG
GSMADMIN_INTERNAL SHARD_TS$LOG_GRP SHARD_TS   TABLESPACE_NAME LOG
GSMADMIN_INTERNAL SHARD_TS$LOG_GRP SHARD_TS   CHUNK_NUMBER    LOG
```

## Switching Supplemental Logging Off

From the output above, I understand what is switched on. The first SQL is what is important, the
rest look like defaults. We have Primary Key, and Unique column logging switched on. 
I can switch this supplemental logging off using the following:

```sql
alter database drop supplemental log data (unique, primary key) columns;
```

If the minimal supplemental logging can't be dropped at this point, oracle support note 2114639.1 says to run:

```sql
alter database drop supplemental log data for procedural replication;
```
before running

```sql
alter database drop supplemental log data;
```

Now supplemental data is switched off. Rerunning SLOB shows the redo to be the same as in dev.

## Why the Difference?

I prefer all my databases to be identical because that means code will behave the same way in all environments.
Clearly production needs more resource than development because more people are using it, and the costs in financial and
environmental terms don't allow allocatiuon of memory or CPU that won't be used. There have to
be some differences. However things like this should be the same. The question is, how come development
has a different setting to my test on the new hardware?

The two databases were created differently. The dev one was created using a SAN snapshot and 
the `create controlfile` SQL command. The test database was created using an RMAN duplicate from a backup.
I infer from this that the supplemental logging configuration is stored in the control file.