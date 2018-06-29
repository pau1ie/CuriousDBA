---
title: "Rman Fail"
date: 2018-02-09T15:37:15Z
tags: ["Oracle","DBA","Backup","Recovery","Rman"]
image: images/recovery.png
---

Here is another issue we had with RMAN. This one has been bugging me for years.

We were doing a duplicate using rman. For some reason the recovery catalogue didn't contain
the archive log we needed to do the recovery, so the restore completed, and finished with
the familiar error:

{{< highlight console >}}
...
executing command: SET until clause

Starting recover at 2018-02-02 16:03:13

starting media recovery

unable to find archived log
archived log thread=1 sequence=307
Oracle Error:
ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01152: file 1 was not restored from a sufficiently old backup
ORA-01110: data file 1: 'system.dbf'
{{< /highlight >}}


The normal thing to do here is to correct the error, and redo the backup. But we
already had the redo logs on the disc, and wanted to be able to apply them.

The problem with that is, we get another error:

{{< highlight console >}}
ORA-01103: database name 'PROD' in control file is not 'CLONE'
{{< /highlight >}}

Where PROD is the database we are copying from, and CLONE is the database we
are copying to. Now, it is pretty easy to run a create controlfile command to
change the name of the database in the controlfile, but we will still be left
with an inconsistent database.

The solution is to rename the database in the parameter file.

{{< highlight console >}}
db_name = 'PROD'
{{< /highlight >}}

Then the database will mount, and we can recover the redo, using  the command
that is etched on to my brain:

{{< highlight sql >}}
recover database until cancel using backup controlfile
{{< /highlight >}}


 Once this is done,
I created a controlfile using:

{{< highlight sql >}}
alter database backup controlfile to trace as 'ccf.sql' resetlogs
{{< /highlight >}}

Then it is a simple matter to:

* Edit ccf.sql to change the database name
* Add in the SET keyword to set the database name
* run it
* alter database open resetlogs

I can't believe something so simple has eluded me for so long!
