---
title: "ORA-13831: SQL profile or patch name specified is invalid"
date: 2019-11-25T10:06:09Z
tags: ['Oracle','DBA','SQL','Fail','Parsing','Optimizer','Performance']
---

We had a process error with the above error message. I believe this is related to Oracle bug 27496360 which is a 
duplicate of 29942554, which is apparently in QA, but is targeted for database 20.1. It being 2019, Oracle 
haven't released Oracle 20 yet, so we have no fix. We note it seems to be triggered by applying critical patches.
We have an SR open for this which is attached to the bug.

We need to work around this problem, which appears to be a corruption in the part of the dictionary which
stores SQL Plan baselines, and causes the parser to crash rather than ignore the baseline and and try to
parse the SQL normally!

The first task is to identify the SQL which is failing. Since it never runs, this is more difficult than
the normal performance issue.

The way to do this is to set an event in the database to create a trace file when the error is encountered.
As sysdba run the following:

```sql
alter system set events '13831 trace name errorstack level 3';
```

Rerun the process and it will fail. Look at the trace file that is generated, and it will have the SQL ID
in it.

Armed with this information, we can see the associated baselines. The _Managing SQL Plan Baselines_
chapter of the Oracle _Database SQL Tuning guide_ gives the information we need:

> The following query displays execution plans for the statement with the SQL ID 31d96zzzpcys9:

```sql
SELECT PLAN_TABLE_OUTPUT
FROM   V$SQL s, DBA_SQL_PLAN_BASELINES b, 
       TABLE(
       DBMS_XPLAN.DISPLAY_SQL_PLAN_BASELINE(b.sql_handle,b.plan_name,'basic') 
       ) t
WHERE  s.EXACT_MATCHING_SIGNATURE=b.SIGNATURE
AND    b.PLAN_NAME=s.SQL_PLAN_BASELINE
AND    s.SQL_ID='31d96zzzpcys9';
```

As above, the _Managing SQL Plan Baselines_
chapter of the Oracle _Database SQL Tuning guide_ tells us how to drop an SQL Plan baseline

```sql
DECLARE
  v_dropped_plans number;
BEGIN
  v_dropped_plans := DBMS_SPM.DROP_SQL_PLAN_BASELINE (
     sql_handle => 'SQL HANDLE HERE'
     );
  DBMS_OUTPUT.PUT_LINE('dropped ' || v_dropped_plans || ' plans');
END;
/
```

While dropping the baseline may cause performance issues, it at least means the SQL won't crash any more.
It is possible that the SQL baseline could be regenerated if desired. In our case they were automatically
generated baselines, and the SQL seems to perform adequately without them.
generated baselines, and the SQL seems to perform adequately without them.