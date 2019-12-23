---
title: "SQL Performance Issues"
date: 2019-12-23T11:03:50Z
tags: ['Oracle', 'SQL', 'Performance','Tuning','Peoplesoft','DBA']
---

The initial investigation of this issue is written in the 
[Production Emergency](../productionemergency/) post. You might want to read  that 
first if you haven't already.

## Addressing the underlying problem

We had identified the source of the problem. The SQL text can be extracted from the database as follows:

```sql
set linesize 300
set pagesize 0
set long 30000
set longchunksize 300

select sql_text from v$sql where sql_id = '67bqun92ngrsj';
```

This displays the SQL text. If the application populates the `module`, `program` and `client_id` 
of `v$session` using
`dbms_application_info`, then there is enough information to find out what code is causing the
problem and even who was running the program that caused it.

It is possible to find the execution plan that the database has chosen for the SQL as follows:

```sql
select * from table(dbms_xplan.display_cursor(sql_id=>'sql_id',format=>'ALLSTATS');
```

It is even possible to find out how much of the plan has been executed:

```sql
SQL> set lines 300
SQL> set pages 50000
SQL> set long 30000
SQL> set longc 30000  
SQL> select dbms_sql_monitor.report_sql_monitor(sql_id=>'sql_id'
  ,report_level=>'ALL') report from dual;
```

Having identified the SQL that is running slow, how do we force a good plan?
That is difficult. Sometimes just purging  the plan and forcing the SQL to get
parsed again will work:

```sql
select address || ',' || hash_value as name from v$sqlarea where sql_id = 'sql_id';
exec dbms_shared_pool.purge('address,hash_value;','C');
```

According to the documentation the first parameter is a concatenation of the address and
hash value. The second is anything apart from any single character in `PpQqRrSsTt`. According to 
[Oracle base](https://oracle-base.com/articles/misc/purge-the-shared-pool#purge-individual-cursors),
most people pick `C` for Cursor.

Another option if the problem SQL is a single statement, is to use SQL Plan Management to
fix a good plan or disable a bad one. To do this, we need to load the plans:

```sql
declare
  v_result pls_integer;
begin
  v_result := dbms_spm.load_plans_from_cursor_cache(sql_id => 'busg12afb9x71');
end;
/
```

It is somewhat difficult to match the `SQL_ID` and `plan_hash_value` of a SQL to an `SQL_Handle` and
`Plan_Name` recorded in `dba_sql_plan_baselines`. Typically it is easiest to search for parts of the
SQL that seem unique, or indeed it is possible to use the SQL as a key:

```sql
select distinct sql_handle 
from dba_sql_plan_baselines 
where SQL_TEXT like
  (select sql_fulltext from v$sqlarea where sql_id = 'sql_id');
```

This is a little slow, but it brings back the handle. Next, how to work out which plan is which?

To do this I used a 
[handy procedure from OracleProf](http://oracleprof.blogspot.com/2011/07/how-to-find-sqlid-and-planhashvalue-in.html)
It prompts for the SQL_Handle.

```sql
set serveroutput on size 30000
set lines 132

DECLARE
  v_sqlid VARCHAR2(13);
  v_num   NUMBER;
BEGIN
  dbms_output.put_line('SQL_ID       ' || ' '
    || 'PLAN_HASH_VALUE' || ' '
    || 'SQL_HANDLE                    ' || ' '
    || 'PLAN_NAME');
  dbms_output.put_line('-------------' || ' '
    || '---------------' || ' '
    || '------------------------------' || ' '
    || '--------------------------------');
  FOR a IN (
    SELECT sql_handle, plan_name,
      TRIM(substr(g.plan_table_output, 
      instr(g.plan_table_output, ':') + 1)) plan_hash_value,
      sql_text
    FROM
      (
        SELECT t.*, c.sql_handle, c.plan_name, c.sql_text
        FROM
          dba_sql_plan_baselines c,
          TABLE ( dbms_xplan.display_sql_plan_baseline(c.sql_handle, c.plan_name)) t
        WHERE
          c.sql_handle = '&sql_handle'
        ) g
      WHERE
        plan_table_output LIKE 'Plan hash value%'
  ) LOOP
    v_num := to_number(
	  sys.utl_raw.reverse(
	    sys.utl_raw.substr(
		  sys.dbms_crypto.hash(src => utl_i18n.string_to_raw(a.sql_text || 
		    CHR(0), 'AL32UTF8'), typ => 2), 9, 4))
          || sys.utl_raw.reverse(sys.utl_raw.substr(
		       sys.dbms_crypto.hash(src => utl_i18n.string_to_raw(a.sql_text
                 || CHR(0), 'AL32UTF8'), typ => 2), 13, 4)), rpad('x', 16, 'x'));
    v_sqlid := '';
    FOR i IN 0..floor(ln(v_num) / ln(32)) LOOP
      v_sqlid := substr('0123456789abcdfghjkmnpqrstuvwxyz', 
	    floor(MOD(v_num / power(32, i), 32)) + 1, 1)
        || v_sqlid;
    END LOOP;
  dbms_output.put_line(v_sqlid || ' '
    || rpad(a.plan_hash_value, 15) || ' '
    || rpad(a.sql_handle, 30) || ' '
    || rpad(a.plan_name, 30));
  END LOOP;
END;
/
```

Running  this prompts for an `sql_handle`. It gives the `sql_id` and the `plan_hash_value`
for each statement. The plan hash value can be matched with a poor plan identified e.g. from
the `sql_monitor` or `dbms_xplan.display_cursor`. I discovered that none of the plans loaded
was the one causing the problem. It appears it was already purged from the cache, suggesting
that the database had already abandoned it due to cardinality feedback.

It still seemed reasonable to give the database a hint that these are good plans to
use, so they are enabled by evolving them:

```sql
declare
    v_clob clob;
begin
    v_clob := dbms_spm.evolve_sql_plan_baseline(sql_handle => 'SQL_327e391735aac611');
    dbms_output.put_line(dbms_lob.substr(v_clob,4000,1));
end;
/
```

This took a really long time (Significantly more than 24 hours) to run, so I left it. 
However I could see that within 15 minutes or so, it had accepted the plans by looking
at the `ACCEPTED` column of dba_sql_plan_baselines.

## Conclusion

The above steps kept the system largely working while the dev team investigated.

The dev team worked out a more efficient way to write the SQL. There was no
obvious reason that I could see for the change in performance of this SQL statement, but
possibly it's complexity combined patching and minor changes in stats just pushed it over
a threshold from a good plan to a bad one.