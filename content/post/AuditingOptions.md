---
title: "Auditing Options in the Database"
date: 2020-02-04T11:35:00Z
tags: ['Oracle', 'Auditing', 'DBA']
---


# Oracle Auditing

Oracle have a number of different types of auditing, and in recently created databases
they all coexist. I looked at this recently and thought I had better make some notes
before I forget.

There are three types of auditing:

* Traditional auditing. This is the way things worked before 12.1
* Unified auditing. This is the new rewrite of auditing, but needs some effort to get working.
* Fine grained auditing. This stays the same between traditional and unified auditing.
* Mixed mode auditing. In the Database Security Guide it mentions that newly created databases
  can use mixed mode auditing. This allows us to use the functionality of both types of audit.

Auditing is one tool that can be used to help secure the system. A particular risk is that of an
attacker updating or removing the audit trail. So auditing should be considered as one tool
to keep the system safe. Unfortunately it is one of those things that doesn't look shiny or
help users to do their work, so doesn't tend to get enough resources to make it work
properly.


# Switching on Traditional Auditing

Switching on auditing is a matter of setting the `audit_trail` to `DB, EXTENDED` and restarting the
database. See the documentation of the `audit_trail` parameter for more options.


## Audit Session

After a month or so in development, our audit trail had millions of rows. Looking at it, the
action field was mostly 100,101 and 102.
The action_name field of the dba_audit_trail view explains what the action is:

action | Name
-------|------
   100 | Logon
   101 | Logoff
   102 | Logoff by clean up

I stopped this by running `noaudit session`. This stops auditing the session information
for traditional auditing. Looking at `dba_stmt_audit_options` before running the noaudit we
can see that `CREATE SESSION` is removed by the `noaudit` command. However, since we were only
interested in using fine grained auditing we could switch this off. The reason for all these
rows appears to be mostly the application logging in and out of the database.

It might be a good idea to audit this type of information, but a way of excluding the large number
of valid application logons would need to be found.

The currently audited actions can be listed using
```sql
select * from dba_stmt_audit_opts;
```

The list of possible audit options can be found in the SQL reference guide under the
audit command - traditional auditing.

# Fine grained auditing

This is the main thing we wanted, to audit who did what to particular data and when.

To set up auditing, we need to tell the database what we want to audit by calling
`dbms_fga.add_policy`:

```sql
begin    
    dbms_fga.add_policy (
        object_schema    =>'USER',
        object_name      =>'TRIAL_TBL',
        policy_name      =>'audit_trial_tbl',
        audit_column     =>'DESCR,EFFDT',
        enable           => TRUE,
        statement_types  =>'SELECT,UPDATE');
end;
```

The list of audit policies can be seen by looking in `dba_audit_policies` The documentation says
this view isn't populated if unified auditing is enabled, but it does appear to be populated
when mixed mode auditing is on. The documentation mentions `UNIFIED_AUDIT_TRAIL`, but this
doesn't make sense, because that is where audit records are sent, not where the audit
policy is viewed.

Anyway, the results are placed in `dba_fga_audit_trail` in mixed mode auditing. It appears they
don't end up in the `unified_audit_trail` in mixed mode auditing.


# Unified auditing

Similar results can be gained from unified auditing. Here I set up and test an audit policy:

```sql
SQL> create audit policy mytest actions update, insert on me.TRIAL_TBL;
Audit policy created.

SQL> audit policy mytest;
Audit succeeded.

SQL> connect user
Enter password:
Connected.
SQL> update trial_tbl  set descr ='my test' 
     where campus = 'thing1' and effdt = '04-SEP-2013';
1 row updated.

SQL> commit;
Commit complete.

SQL> connect / as sysdba
Connected.

SQL> select object_name, sql_text from unified_audit_trail
     where OBJECT_NAME = 'TRIAL_TBL';
OBJECT_NAME     SQL_TEXT
--------------- -----------------------------------------------------------------
PS_CAMPUS_TBL   update ps_campus_tbl set descr ='my test' where campus = 'thing1'
```

Policies are stored in `audit_unified_policies` and the `audit_unified_enabled_policies` view
lists the policies that are enabled.
