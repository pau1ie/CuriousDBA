---
title: "Oracle 19c Critical Patching"
date: 2021-01-20T15:20:49+01:00
tags: ["Ansible","Automation","Critical Patch","Database","DBA","Oracle","Scripting"]
---

It is the time of year to apply critical patches. This time round we have some databases
at version 19c. We tend to install a new oracle home
for each patch, as we find this helps us manage the migration. It also reduces 
downtime, especially if you have not yet taken advantage of pluggable databases
as we haven't.

## Oracle Installer

I notice from the documentation that there is an `applyRU` parameter that can be passed
to the installer to apply the patch. The documentation is quite poor though because
it doesn't specify what should be passed after that. Even more annoying is that if the
patch attempt fails, it will leave the home in an intermediate state and it will have 
to be deleted and recreated.

After much trial and error with a combo patch I discovered the following procedure works:

  * Unzip the installer zip file into the new `ORACLE_HOME`.
  * Upgrade `OPatch` by unzipping the patch into the `ORACLE_HOME`.
  * Unzip the combo patch somewhere convenient. I chose `$ORACLE_HOME/patch`.

The combo patch contains two patch directories, database roll-up and Java. It seems they need to be applied
as follows:

Make sure `ORACLE_HOME` is unset. I prefer to log out and in again and don't set an environment,
I like  to make sure `LD_LIBRARY_PATH` and `PATH` don't contain anything under an oracle home in
case it causes problems.

```bash
cd /my/new/oracle/home
./runInstaller -applyRU patch/32126828/32218454 -applyOneOffs patch/32126828/32067171
```

The `applyRU` parameter specifies the database release update patch, and `applyOneOff` specifies the java patch. This
is also a release update, but only one release update can be specified in the `applyRU` parameter. Any number of 
`applyOneOff` patches can be supplied, they are comma delimited.

It takes a few minutes to run each patch in, but it does work! Then the installer 
window appears. I picked the following options:

  1. *Set up software only*
  2. *Single instance database installation*
  3. *Enterprise Edition*
  4. Oracle base: I chose `/oracle`
  5. `dba` group for all groups apart from *osoper* which was left blank.
  6. Left *automatically run configuration scripts* unticked.
  7. Ticked *Ignore all* on the prequisite checks (We have a smaller swap than Oracle would like but we have masses of RAM.
  8. On the summary page I chose to *save the response file*. 
  9. I ran the `root.sh` when instructed.
  10. The registration was successful! I clicked *close*.

## More Automation

Armed with the response file generated above, I can use the following to run the installer in
silent mode using the response file I generated above:

```bash
cd /my/new/oracle/home
./runInstaller -silent -responseFile /path/to/my/db.rsp \
     -applyRU patch/32126828/32218454 \
	 -applyOneOffs patch/32126828/32067171
```

We still need to run `root.sh` afterwards, as prompted by the standard output. Note that the behaviour is slightly different
because `dbhome`, `oraenv` and `coraenv` are copied to `/usr/local/bin`, where if it is run interactively and the defaults
taken these files are not copied to `/usr/local/bin`.

## Switching Options with chopt
  
Once the home is installed, I make a habit of switching off cost options we  haven't licensed to
prevent accidental usage. Some Oracle scripts check and use the options if they are
switched on, so I think it is worth doing. We don't use any options, so we do:
 
```bash
chopt disable oaa
chopt disable olap
chopt disable paritioning
chopt disable rat
```

It is general good practice to check whether something needs doing before doing it, especially when
using languages like Ansible. Looking at the `Makefile` called by the `chopt` script I can see
that it manges the `libknlopt.a` file. This is an archive file, which is kind of like a
zip file, but it is controlled using the Unix `ar` command to list, add and remove files from the
archive. `ar` commands are similar to `tar` commands, so `t` lists the contents of the archive.

```bash
cd $ORACLE_HOME/rdbms/lib
ar t libknlopt.a
```

The options supported by `chopt` end up with the following objects being included in the archive:

  * `dmwdm.o` Oracle Advanced Analytics is on.
  * `dmndm.o` Oracle Advanced Analytics is off.
  * `xsyeolap.o` Oracle OLAP is on.
  * `xsnoolap.o` Oracle OLAP is off.
  * `kkpoban.o` Oracle Partitioning is on.
  * `ksnkkpo.o` Oracle Partitioning is off.
  * `kecwr.o` Oracle Real Application Testing is on.
  * `kecnr.o` Oracle Real Application Testing is off.
  
There are other options in here such as RAC which is set during installation (`kecwr.o` is on, `kecnr.o` is off)
and unified auditing which is controlled by another script (`kzaiang.o` is on, `kzanang.o` is off)

Armed with this information we can tell Ansible whether it needs to switch options on or off.