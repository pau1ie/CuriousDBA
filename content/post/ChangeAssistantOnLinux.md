---
title: "Change Assistant on Linux"
date: 2020-10-23T17:20:49+01:00
tags: ["PeopleSoft","Automation"]
---

## Automating Tools Patches

I have already automated the VM builds, so now I want to automate the application of the patch into
the database. Oracle have done some work here with their Change Assistant tool, which once the database
is set up can be used to apply the patch. Hopefully I can find a way to run change assistant from the
Linux command line and apply the patch using the automated procedure.

The problem is that Change Assistant is a Windows only tool... Or is it? You know the treasure hunt
that children like where you hide clues, and each clue leads to the next one? Following the Oracle 
documentation is rather like that!

## Documentation

I don't normally include links to Oracle Documentation because they change so often. If these links are
broken then a search engine might be able to find them. Search on the title in quotes. 
Include the tools version as a search term if you have trouble finding the correct version.

The 
[Change Assistant and Upgrade Manager documentation](https://docs.oracle.com/cd/F25245_01/psft/pdf/pt858tswu-b122019.pdf) 
says in Chapter 4:

> Change Assistant runs on supported Microsoft Windows workstations and limited use on Linux.
> 
> Complete installation instructions for Change Assistant appear in your PeopleSoft 9.2 Application
> Installation guide.
> 
> See _PeopleSoft 9.2 Application Installation_ for your database platform: Installing PeopleSoft Change
> Assistant

And after some discussion about the Windows install is the following:

> **Linux**
> 
> When you install the PeopleTools DPK for Linux, you can select to install Change Assistant.
> 
> Linux supports a command line version of Change Assistant.
> 
> Change Assistant on Linux is not supported for target databases that are below PeopleTools 8.55.16.

Following the trail we get to the 
[PeopleSoft install guides](https://docs.oracle.com/cd/F29384_01/psft/index.html),
and clicking on the 
[PeopleSoft 9.2 Application Installation on Oracle (PeopleSoft PeopleTools 8.58)](https://docs.oracle.com/cd/F29384_01/psft/pdf/PeopleSoft9.2_Application_Installation_Oracle_PeopleTools8.58_July2020.pdf)
we can see Task _17-1-6: Using Change Assistant on Linux_.

## Install

The documentation says to log in as a user with access to change the files under PsCA, which is _psadm1_ in the default case.
Run the setup command with parameters of `-p installdir` where `installdir` is the path to install to. In my case
it did not exist and was created by the installer.


```bash
cd $PS_HOME/setup/PsCA
./setup.sh -p ~/PsCA -t new
```

Now we have to find the next clue. The documentation says:

> Go to the installation directory, and run Change Assistant in Update Manager mode using the command-line
> options in the Change Assistant product documentation.
> 
> See _PeopleTools: Change Assistant and Update Manager_, "Running Change Assistant Job from the
> Command Line."

So that is back to the first document...

## First Run

The document says to run `changeassistant.bat`. Looking in the install directory I can see `changassistant.sh`
which is presumably what we need to run. I note that the command only works when
run from it's own directory, otherwise it can't find it's Java installation.

```console
$ ./changeassistant.sh ?
Picked up _JAVA_OPTIONS: -Djava.security.egd=file:/dev/./urandom
 document is created
PeopleTools 8.58.03

Copyright ? 1987, 2019, Oracle and/or its affiliates. All rights reserved. Oracle
 and Java are registered trademarks of Oracle and/or its affiliates. Other names 
 may be trademarks of their respective owners. Intel and Intel Xeon are trademarks 
 or registered trademarks of Intel Corporation. All SPARC trademarks are used under 
 license and are trademarks or registered trademarks of SPARC International, Inc. 
 AMD, Opteron, the AMD logo, and the AMD Opteron logo are trademarks or registered 
 trademarks of Advanced Micro Devices. UNIX is a registered trademark of The Open 
 Group. 


Usage:
changeassistant [-INI Path of ini file (Optional)]
                    -INI c:\temp.ini


changeassistant [-MODE mode of the CA action (Required)]
                    -MODE UM = Update manager Mode
                [-ACTION action name (Required)]
                    -ACTION ENVIMP = Import Environment
                    -ACTION ENVCREATE = Create Environment
                    -ACTION ENVUPDATE = Update Environment
                    -ACTION PRPAPPLY = Apply PRP
                    -ACTION PTPAPPLY = Apply PTP
                    -ACTION PTUAPPLY = Apply PTU
                    -ACTION UPGAPPLY = Apply UPG
                    -ACTION CPAPPLY = Apply Previously Created (non-PRP)
      					Change Package
                    -ACTION DLTAPPLY = Apply Tools Delta Package
                    -ACTION TDPAPPLY = Apply Translation Delta Package
                    -ACTION RFUAPPLY = Apply Required For Upgrade Package
                    -ACTION CPDEFINE = Define a New Change Package
                    -ACTION CPCREATE = Create Change Package
                    -ACTION UPLDTGT = Upload target info to PUM source
                    -ACTION EMFVAL = Validate EMF Settings
                    -ACTION OPTIONS = Set General Options
                    -ACTION EXPCFG = Export configuration
                    -ACTION IMPCFG = Import configuration
                    -ACTION UPLDCUSTDATA = Upload Customer Metadata And Test Data
                    -ACTION EXPCUSTDATA = Export Customer data from existing PUM
                         Source and save it as a data file
                    -ACTION IMPCUSTDATA = Import Customer data from the selected 
					     data file to the New PUM Source

```
Looking good! I notice one can run something like the following to get extra help. The
help seems more up to date than the documentation.



```bash
./changeassistant.sh -MODE UM -ACTION <action> ?
```

## Configuring The Database

It seems there are a couple of ways to configure the database. I can either
import a configuration I have exported from elsewhere, or else I can create a
new database, I will try to create a new one. Here is what I figured out from
the online help and the document. When setting up a database in Windows,
some of these parameters are read from the database and defaulted, so I
just copied those from a Windows installation.

Note the carriage returns below are for readability. The real command is all on
one line. Also the comments aren't in  the command at all.

```bash
changeassistant.sh 
  -MODE UM 
  -ACTION ENVCREATE 
  -TGTENV DBNAME
  -OUT /tmp/CA.log
  -REPLACE Y
  -CT 2                    # ORACLE
  -CS dbserver
  -UNI Y                   # Document erroneously refers to UDI. Unicode.
  -CA sysadm
  -CAP sysadmpass
  -CO oprid
  -CP oprpassword
  -CI people
  -CW peoplepassword
  -SQH "/opt/oracle/psft/pt/oracle-client/12.1.0.2/bin/sqlplus"
  -INP SA,EO,CMP,SC,AA,SAD,XCC,FA,SSF,SR
  -PL "CAMPUS SOLUTIONS"   # Coped from the Windows CA.
  -IND All                 # Coped from Windows
  -INL ENG                 # Coped from Windows
  -PSH $PS_HOME
  -PAH $PS_APP_HOME
  -PCH $PS_CUST_HOME
  -NPYN N                  # Don't enable new peoplesoft home
```

This reports back:
```console
Picked up _JAVA_OPTIONS: -Djava.security.egd=file:/dev/./urandom
 document is created
The new environment DBNAME has been created.
```

Great!

If however, you get the following:
```console
Picked up _JAVA_OPTIONS: -Djava.security.egd=file:/dev/./urandom
Meet exception : 
Could not find a default PSHOME. Make sure PSHOME\bin\client\winx86 is in your System Path variable.
        at com.peoplesoft.pt.changeassistant.PSSamAccess.SignoffDb(PSSamAccess.java:139)
        at com.peoplesoft.pt.changeassistant.PSSamAccess.SignoffDb(PSSamAccess.java:127)
        at com.peoplesoft.pt.changeassistant.client.commandline.CaCommandLine.retrieveInfomation(CaCommandLine.java:1398)
        at com.peoplesoft.pt.changeassistant.client.commandline.CaCommandLine.createEnvironment(CaCommandLine.java:1229)
        at com.peoplesoft.pt.changeassistant.client.commandline.CaCommandLine.execute(CaCommandLine.java:301)
        at com.peoplesoft.pt.changeassistant.client.main.frmMain.main(frmMain.java:3502)
```
This means you have got a password wrong, or maybe it can't connect to the
database. A better error message would be **very** much appreciated.
I suppose it is also possible it couldn't find a default PSHOME, but that wasn't the case here.

### Alternative

It is also possible to configure the database using an ini file containing the same information.
The ini file is deleted by change assistant to preserve the passwords.

## Setting the Options

When running change assistant there are options that need to be set in addition to setting
up the database, which pertain to directories on the local machine. This is done as per the
next section in the document _Command Line for Setting Options_. I ran the following, making sure
that the directories already existed beforehand. Once again, the real command doesn't have
carriage returns in it.

```bash
changeassistant.sh 
  -MODE UM 
  -ACTION OPTIONS
  -OUT ~psadm1/ca/output
  -REPLACE Y
  -SWP False
  -PSH $PS_HOME
  -STG ~/ca/staging
  -OD ~/ca/output 
  -DL ~/ca/download
  -SQH "/opt/oracle/psft/pt/oracle-client/12.1.0.2/bin/sqlplus"
```

I don't know if it is necessary to set the SQL client again, I did anyway.

## Applying the change set.

### What I want to achieve

The PUM database I cloned is at 8.58.04. I want to upgrade it to the latest critical patch of tools
which is 8.58.05, so a very small upgrade.

### Oracle Documentation.

Now we need to pick up our treasure hunt. The next step is found in the 
[PeopleSoft PeopleTools 8.58 Deployment Packages Installation Guide](https://docs.oracle.com/cd/F25690_01/psft/pdf/PeopleSoft_PeopleTools8.58_Deployment_Packages_Installation_July2020.pdf).
These are dated, the latest one as I write this is July 2020.
It can be found by following the links from the 
[PeopleTools Help Centre](https://docs.oracle.com/en/applications/peoplesoft/index.html)
Click on PeopleTools Documentation then _PeopleSoft PeopleTools 8.58 Deployment Packages Installation_ under Install and Upgrade.

_Task A-1-6_ in _Appendix A_ mentions how to run change assistant interactively. This is what
we want to achieve, but using the command line. 

_Task C-1-6_ says we can do the upgrade on Linux, and refers us to _PeopleTools: Change Assistant and Update Manager_, _Running Change Assistant Job from the Command Line_. We have already seen this document.

Helpfully there is a section on _Command line for Applying PeopleTools Patch_ which seems to be
what we want.

## The Command Line

This is similar to setting up the database. We need to run the command as follows:

```bash
changeassistant.sh -MODE UM -ACTION PTPAPPLY -TGTENV DBNAME -UPD PATCH858
```
The oputput looks good;
```console
$ bash ../runca.sh 
Picked up _JAVA_OPTIONS: -Djava.security.egd=file:/dev/./urandom
RESUMEJOB parameter not set, continue.
inProgress = false
There is no job set in progress.
Primary Checks

PTP Specific Validations

Check 1: 
PeopleTools Patch Version Check 

Passed. 
The Selected PeopleTools Patch Package is a Newer Release than the Target Database. 
PeopleTools Release of Selected Patch 8.58.05
PeopleTools Release of Target Database 8.58.04



Target Environment Validations

Check 2: 
Database type and PS home version. 

Passed. 



Check 3: 
Try to get lock on Target Environment. 

Passed. 



Change Package 'PATCH858' will be applied to 'DBNAME'.

Patch Directory: /opt/oracle/psft/pt/ps_home8.58.05/PTP/
Apply Type: Initial Pass (Target Steps Only)
Synchronize Target Metadata: No
Target Database Tools Release: 8.58.04
Target PS_HOME Tools Release: 8.58.05

Target Environment:
    Name: DBNAME
    Database: 
        Name: DBNAME
        Platform: Oracle
        Unicode: Yes
        User ID: PS
        Connect ID: people
        Access ID: sysadm
        DB Owner ID: Not Set
        Product Line: CAMPUS SOLUTIONS
        Industry: All
        Products: SG, SA, CSS, EO, AV, SC, AA, SAD, XCC, FA, SSF, SR
        Languages: ENG
        BaseLanguage: ENG
        Database Version: 8.58.04
        SQL Query Tool: /opt/oracle/psft/pt/oracle-client/12.1.0.2/bin/sqlplus
        PS Home: /opt/oracle/psft/pt/ps_home8.58.05/
        PS App Home: /psoft/APP_HOME/
        PS Cust Home: /psoft/CUST_HOME/
        SQR Executables(SQRBIN): <PS_Home>/bin/sqr/<plat>/bin
        SQR Flags(PSSQRFLAGS): -ZIF/opt/oracle/psft/pt/ps_home8.58.05/sqr/pssqr.unx
        PSSQR Path: <StagingDir>sqr:<Cust_Home>sqr:<App_Home>sqr:<PS_Home>sqr
        Use PeopleSoft Test Framework: No
        Use Process Scheduler: No

PUM SOURCE: Not Set

Initializing Logger
********Package Apply Start : Mon Oct 19 15:03:39 BST 2020
Sending pulse from 'com.peoplesoft.emf.peer:id=2'
*Preparing updates for environment Start :Mon Oct 19 15:03:39 BST 2020
*Retrieving CP Start :Mon Oct 19 15:03:39 BST 2020
*Creating template Start :Mon Oct 19 15:03:39 BST 2020
*Applying /opt/oracle/psft/pt/ps_home8.58.05/PTP/updPATCH858.zip Mon Oct 19 15:03:40 BST 2020
*Creating job softwareupdate.PATCH858.{-DBNAME}IP.20201019150247 Mon Oct 19 15:03:42 BST 2020
*Extracting scripts for softwareupdate.PATCH858.{-DBNAME}IP.20201019150247 Mon Oct 19 15:03:44 BST 2020
********Package Apply End : Mon Oct 19 15:03:44 BST 2020

********Job Start********

Job : softwareupdate.PATCH858.{-DBNAME}IP.20201019150247 starts at Mon Oct 19 15:03:45 BST 2020

**Running** Step "Running the Initial Filter Queries on the Target" at Mon Oct 19 15:03:45 BST 2020

```
Until it fails.
```console
**Running** Step "Running Datamover - clear_rowset_cache.dms" at Mon Oct 19 15:06:34 BST 2020

**Completed** Step "Running Datamover - clear_rowset_cache.dms" at Mon Oct 19 15:06:35 BST 2020

**Running** Step "Running Datamover - PTPatch.dms" at Mon Oct 19 15:06:35 BST 2020

**Failed** Step "Running Datamover - PTPatch.dms" at Mon Oct 19 15:06:35 BST 2020

Step Failed at: Running Datamover - PTPatch.dms


********Job End Status:Failed at step Running Datamover - PTPatch.dms********
```
We didn't get this error when running it interactively, so it is difficult to know why it happens,
however I can see the script `PTPatch.dms` is in the `scripts` subdirectory. Some old
Oracle support notes (e.g. 
[2644685.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2644685.1)
suggest PTpatch can be run manually, so I thought I would give it a try.

Running data mover from the command line is documented in _PeopleTools 8.58: Lifecycle Management Guide_
chapter 16 section _Running Data Mover Scripts from the Command Line_

```console
$ psdmtx -CD DBNAME -CT ORACLE -CI connectID -CW connectPW \
               -CO OprID -CP OprPW -fp PTPatch.dms 
PeopleTools 8.58.05 - Data Mover
Copyright (c) 2020 PeopleSoft, Inc.
All Rights Reserved

Started:  Mon Oct 19 16:22:10 2020
Data Mover Release: 8.58.05
Database: DBNAME (ENG)
SQL Successful - 
 UPDATE PSSTATUS SET PTPATCHREL = 5, PTLASTPTCHDTTM = %CURRENTDATETIMEIN
Ended: Mon Oct 19 16:22:10 2020
Successful completion
```

All good, now I need to complete the job. Back to the 
_PeopleTools 8.58: Change Assistant and Update Manager_ manual. It seems we can continue the 
process from the next step by running:

```bash
$ changeassistant.sh -MODE UM -ACTION PTPAPPLY -TGTENV DBNAME \
                     -UPD PATCH858 -RESUMEJOB COMPLETECONTINUE
```

But it doesn't work.:
```console
Picked up _JAVA_OPTIONS: -Djava.security.egd=file:/dev/./urandom
Restoring list {CACurrentSpJobGroup} from backing storage
Restoring list {CAEnvPackageInfo} from backing storage
The resuming job can't be reset.
```

It seems you have to pass another parameter: `RESETJOB N` to make it work.

```console
$ changeassistant.sh -MODE UM -ACTION PTPAPPLY -TGTENV DBNAME -UPD PATCH858 -RESETJOB N -RESUMEJOB COMPLETECONTINUE
Picked up _JAVA_OPTIONS: -Djava.security.egd=file:/dev/./urandom
Restoring list {CACurrentSpJobGroup} from backing storage
Restoring list {CAEnvPackageInfo} from backing storage
RESUMEJOB parameter set, do not need reset job.
Initializing Logger
2020-10-19 17:41:39,467 main EMF_CATEGORY - register called with ObjectName com.peoplesoft.emf.peer:id=2
2020-10-19 17:41:39,469 main EMF_CATEGORY - Unregistering the proxies from the server
Sending pulse from 'com.peoplesoft.emf.peer:id=2'

********Job Start********

Job : softwareupdate.PATCH858.{-DBNAME}IP.20201019150247 starts at Mon Oct 19 17:41:40 BST 2020

Finding Step softwareupdate.PATCH858.20201019150247.Administrative and Cleanup Tasks.Completing the PeopleTools Patch Process.Running Datamover - PTPatch.dms failed last time. Mark it complete and continue execution .
**Running** Step "Resetting Object Version Numbers" at Mon Oct 19 17:41:40 BST 2020

ExecuteReponse: 
**Completed** Step "Resetting Object Version Numbers" at Mon Oct 19 17:41:42 BST 2020

Job : softwareupdate.PATCH858.{-DBNAME}IP.20201019150247 ends at Mon Oct 19 17:41:42 BST 2020


********Job End Status:All steps applied successfully.********
```

It didn't do anything, but it did tidy things up.

## Conclusion

So now we can apply a tools patch from the command line. The next step is how to integrate it into
Ansible. It would also be helpful to be able to pre-emptively fix the issue with `PTPatch.dms` so that it
doesn't error.
