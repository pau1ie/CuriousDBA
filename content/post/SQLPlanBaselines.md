---
title: "SQL Plan Baselines"
date: 2019-04-24T14:59:40+01:00
draft: true
---

The situation is: 

* The SQL runs fast in test, but slow in production.
* It is using a different plan.
* Using sqltxplain we can see there is not a significant difference between the systems.

So, what we can do is to take the plan from test using SQL Plan management and apply it to
production.

First we run the SQL or something like it in test.

Check it is in the shared pool.

{{< highlight console >}}
SQL> select sql_id, plan_hash_value, ELAPSED_TIME/executions as avgTime,
     executions, sql_text from v$sql where sql_text like
     '%somthing unique in the SQL text%';


SQL_ID        PLAN_HASH_VALUE AVGTIME    EXECUTIONS
------------- --------------- ---------- ----------
1mpfp2xr0jdvz      4201701094 51954180.5          2
SELECT * FROM HORRIBLY_COMPLEX_VIEW WHERE COMPLICATED
= COMPLEX
{{</highlight>}}

Load the plan into a baseline

{{< highlight console >}}
SQL> set serveroutput on
SQL> var res number
SQL> exec :res := dbms_spm.load_plans_from_cursor_cache(sql_id =>'1mpfp2xr0jdvz');

PL/SQL procedure successfully completed.

SQL> exec dbms_output.put_line(:res);
1

PL/SQL procedure successfully completed.
{{</highlight>}}

Find out what the SQL Handle is. The query lists sql plan baselines created today.

{{< highlight console >}}
SQL> set lines 132
SQL> col SQL_HANDLE for a20
SQL> col full_text for a65
SQL> col plan_name for a30
SQL> set long 130
SQL> select SQL_HANDLE, PLAN_NAME, ENABLED, ACCEPTED, FIXED, sql_text
     from DBA_SQL_PLAN_BASELINES where created > (sysdate - 1);

SQL_HANDLE           PLAN_NAME                      ENA ACC FIX
-------------------- ------------------------------ --- --- ---
SQL_TEXT
--------------------------------------------------------------------------------
SQL_144a7f367c7cec78 SQL_PLAN_18kmz6ty7tv3se163d42e YES YES NO
UPDATE
PS_UC_AF_GRADM_T4 GRADM
SELECT * FROM HORRIBLY_COMPLEX_VIEW WHERE COMPLICATED
= COMPLEX
{{</highlight>}}

Create a staging table if you haven't got one already

{{< highlight console >}}
SQL> exec dbms_spm.create_stgtab_baseline('STGTAB','ME','USERS');

PL/SQL procedure successfully completed.
{{</highlight>}}

Pack the baseline into a staging table ready for transferring to production:

{{< highlight console >}}
SQL> set serveroutput on
SQL> var res number
SQL> exec :res := dbms_spm.pack_stgtab_baseline('STGTAB','ME','SQL_144a7f367c7cec78')
SQL> exec dbms_output.put_line(:res);
1

PL/SQL procedure successfully completed.
{{</highlight>}}

Where the last parameter is the sql_handle from above. The 1 in res shows one SQL
statement was packed.

Now an export of the table can be done followed by an import to create the table:

{{< highlight console >}}
exp me tables=stgtab file=stgtab.dmp
imp me tables=stgtab file=stgtab.dmp
{{</highlight>}}

Now the staging table has been created the SQL plan baseline can be unpacked.
Since there is only one sql statement in the table, I don't need to specify
the handle, it will unpack all of the SQLs in the table.

{{< highlight console >}}
SQL> set serveroutput on
SQL> var res number
SQL> exec :res := dbms_spm.unpack_stgtab_baseline('STGTAB','ME') 

PL/SQL procedure successfully completed.

SQL> exec dbms_output.put_line(:res);
1

PL/SQL procedure successfully completed.
{{</highlight>}}


