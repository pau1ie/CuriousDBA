---
title: "Fixing Logical Standby Go Slow"
date: 2020-01-06T14:03:50Z
tags: ['Oracle', 'Fail', 'DBA']
---


## Logical Standby Databases

We use a logical standby database. 
The logical standby database is inherently fragile, because it mines the primary
logs, and attempts to rebuild the SQL to apply on the logical.

This rebuilt SQL is fine most of the time, but if an application release
has altered a table, or updated most of the rows in a large table, this
generated SQL often performs very poorly. 

## Logical Standby Failure Modes

Typically there are two failure modes for the logical standby database. Either
it will fall over because a SQL statement it has generate doesn't work, or the
generated SQL statement will perform really badly and take forever. The first
is easy to deal with. Find the error message, fix the error, or ignore the SQL
and continue. The second is more difficult, as there is no error message
to investigate. Typically we will notice that the logical standby is getting
a backlog of redo to apply. We need to investigate what is taking a long time.

## Skipping an error

So, the easy one first. If looking at the error we see it can be skipped (e.g.
a row being inserted already exists) it can be skipped using:

```sql
alter database start logical standby apply skip failed transaction;
```

## Currently Running SQL

The currently running SQL on the logical standby database can be seen using the
following:

```sql
select sid, serial#, sql_id from v$session where 
  sid in (select sid from V$LOGSTDBY_PROCESS)
  and sql_id is not null;
```

## Fixing the Problem

Once the problem SQL has been identified it is a fairly simple process
to fix the issue. Kill the session. Generally the usual:

```sql
alter database stop logical standby apply;
```

won't work as it will wait for the same slow SQL to complete, so it is a case
of running:

```sql
alter system kill session 'sid,serial#';
```

The sid and serial# are identified from the first SQL above. Armed with the
slow SQL statement we can see which table was causing the slowdown, and it
can be skipped:

```sql
execute dbms_logstdby.skip('DML','OWNER','TABLE');
```

Then the apply can be restarted using:

```sql
alter database start logical standby apply;
```

## Reinstantiating

If the table is required, it can be re-instantiated once the logical standby
has cleared it's backlog using:

```sql
alter database stop logical standby apply;
dbms_logstdby.instantiate_table('OWNER','TABLENAME','DBLINK');
execute dbms_logstdby.unskip('DML','OWNER','TABLE');
alter database start logical standby apply;
```

Where DBLINK is a database link back to the primary so that the table can be
fetched across it. This makes the table the same as the primary, so the logical standby
can now continue and keep it up to date.
