---
title: "Parsing"
date: 2017-12-19T10:54:05Z
tags: ["Oracle","DBA","Parsing","Performance","NLS_LANG","Language"]
image: images/book.JPG
---

We are running a data conversion. The powers that be decided to use APIs to
convert the data as they contain error checking. The problem is that  they
are generally designed for interactive use updating one row at a time, so
they are very slow to update large batches of data.

This was tuned and is getting much faster, however we noticed that there are
a lot of waits on *cursor: pin: S wait on X*. From experience I know that this
is caused by excessive hard parsing. However, I couldn't find any statements
with a lot of hard parses. The most was about 40, so nothing like the
thousands which were a cause of this problem early in the release cycle of 12c,
where cursors weren't being shared properly (Bug 20476175 fixed by the patch
of the same number).

[Carlos Sierra has a useful script](https://carlos-sierra.net/2017/09/01/poors-man-script-to-summarize-reasons-why-cursors-are-not-shared/)
to help with this. It was linked from [discussion on Oracle Support document
ID 296377.1](https://community.oracle.com/message/14595209#14595209)
 which contains a similar script, but that one
installs procedures and views into the database, which makes me reluctant to
run it. Carlos comes to the rescue! (Thanks by the way to User3641887 for
pointing me at Carlos' script).

{{< highlight sql >}}
SQL> @shared
( DIFFERENT_LONG_LENGTH
, LOGICAL_STANDBY_APPLY
, DIFF_CALL_DURN
, BIND_UACS_DIFF

  -- Snip! --

, NO_TRIGGER_MISMATCH
, FLASHBACK_CURSOR
, ANYDATA_TRANSFORMATION
, PDDL_ENV_MISMATCH
, TOP_LEVEL_RPI_CURSOR
  1  ( DIFFERENT_LONG_LENGTH
  2  , LOGICAL_STANDBY_APPLY
  3  , DIFF_CALL_DURN

  -- Snip! --

 61  , FLASHBACK_CURSOR
 62  , ANYDATA_TRANSFORMATION
 63  , PDDL_ENV_MISMATCH
 64* , TOP_LEVEL_RPI_CURSOR
  1  SELECT COUNT(*) cursors, COUNT(DISTINCT sql_id) sql_ids, reason_not_shared
  2  FROM v$sql_shared_cursor UNPIVOT
  3  ( value FOR reason_not_shared IN
  4  ( DIFFERENT_LONG_LENGTH
  5  , LOGICAL_STANDBY_APPLY
  6  , DIFF_CALL_DURN

  -- Snip! --

 64  , FLASHBACK_CURSOR
 65  , ANYDATA_TRANSFORMATION
 66  , PDDL_ENV_MISMATCH
 67  , TOP_LEVEL_RPI_CURSOR
 68  )
 69  )
 70  WHERE value = 'Y'
 71  GROUP BY reason_not_shared
 72* ORDER BY cursors DESC, sql_ids DESC, reason_not_shared
please wait

   CURSORS    SQL_IDS REASON_NOT_SHARED
---------- ---------- -----------------------------
      1795       1453 LANGUAGE_MISMATCH
       127        122 HASH_MATCH_FAILED
       104         84 USE_FEEDBACK_STATS
        34         29 OPTIMIZER_MISMATCH
        27         19 OPTIMIZER_MODE_MISMATCH
         6          6 BIND_MISMATCH
         5          3 BIND_LENGTH_UPGRADEABLE
         4          4 PURGED_CURSOR
         4          2 PLSQL_CMP_SWITCHS_DIFF
         2          2 AUTH_CHECK_MISMATCH
         2          2 TRANSLATION_MISMATCH
         1          1 BIND_UACS_DIFF
         1          1 LOAD_OPTIMIZER_STATS
         1          1 ROLL_INVALID_MISMATCH
{{< /highlight >}}

This is interestng. The processes are running from Process Schedulers (This
is what PeopleSoft calls the batch controllers). These are built using
automation so should be the same?

On investigation, I found in Unix:

{{< highlight console >}}
$ echo $NLS_LANG
AMERICAN_AMERICA.UTF8
{{< /highlight >}}

This is set in delivered file $PS_HOME/setup/psdb.sh, which is called from
$PS_HOME/psconfig.sh, which gets called from the .bashrc file.

But on Windows (Approach from Oracles
[NLS LANG FAQ](http://www.oracle.com/technetwork/database/database-technologies/globalization/nls-lang-099431.html#_Toc110410545):

{{< highlight console >}}
C:\psft>sqlplus /nolog

SQL*Plus: Release 12.1.0.2.0 Production on Wed Dec 13 15:52:14 2017

Copyright (c) 1982, 2014, Oracle.  All rights reserved.

SQL> @.[%NLS_LANG%].
SP2-0310: unable to open file ".[ENGLISH_UNITED KINGDOM.WE8MSWIN1252]..sql"
SQL> exit

C:\psft>echo %NLS_LANG%
%NLS_LANG%
{{< /highlight >}}

This shows that NLS_LANG is set to ENGLISH_UNITED KINGDOM.WE8MSWIN1252 in the
registry. This is presumably done by the installer, which must notice some
other United Kingdom localisations in the registry and copy those.

We tried setting NLS_LANG in the Windows process scheduler to AMERICAN_AMERICA.UTF8,
by changing the registry as per the
[NLS LANG FAQ](http://www.oracle.com/technetwork/database/database-technologies/globalization/nls-lang-099431.html#_Toc110410545)
and now we don't get the high waits on cursor S pin on X.




