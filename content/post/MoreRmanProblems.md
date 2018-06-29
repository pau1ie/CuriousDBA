---
title: "More Recovery Manager Problems"
date: 2018-03-29T15:41:13Z
tags: ["Oracle","DBA","Backup","Recovery","Rman"]
image: images/recovery.png
---

A nice feature of RMAN is that if a restore fails, and you rerun it,
it realises that it doesn't have to redo all the work it has already
done.

At least, that is usually what happens.

Today we got:

{{< highlight console >}}
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of Duplicate Db command at 03/29/2018 12:54:29
RMAN-05501: aborting duplication of target database
RMAN-03015: error occurred in stored script Memory Script
RMAN-06004: ORACLE error from recovery catalog database:
    RMAN-20003: target database incarnation not found in recovery catalog
{{< /highlight >}}

As documented in oracle support note 2036644.1, this is caused by
oracle bug 14683854, and the workaround is to remove

  $ORACLE_HOME/dbs/\_rm\_dup\_&lt;SID>.dat

in the Oracle home of the database
being copied.

The problem with this is that RMAN starts the whole restore again.
And of course, the orginal problem that caused the error is probably
still there...
