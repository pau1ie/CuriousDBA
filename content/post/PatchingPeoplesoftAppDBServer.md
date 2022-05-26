---
title: "PeopleSoft 8.59 Infra DPK and Critical Patch Updates - Tuxedo and DB"
date: 2022-05-26T09:16:07+01:00
tags: ["Weblogic","PeopleSoft","Critical Patch","Tuxedo","Oracle","Database"]
---

Further to my last article about applying critical patches to the web server,
lets take a look at the application and database servers.

## Which Servers?

In my mind WebLogic runs on the web tier, and Tuxedo runs on the Application
and process scheduler tier. The Oracle Client is also on the Application and
process scheduler tier. However, since these and Java are installed
on all tiers, we should apply the patches on all tiers.


### Checking the Tuxedo Version

It seems that WebLogic and Tuxedo on my application server are not linked to
the central repository. So if I try to list  the installed patches, I just
get an error:

```console
$ ORACLE_HOME=/opt/oracle/psft/pt/bea/tuxedo \
    JAVA_HOME=/opt/oracle/psft/pt/jdk        \
    /opt/oracle/psft/pt/bea/tuxedo/OPatch/opatch lsinventory

Invoking OPatch 11.2.0.1.2

Oracle Interim Patch Installer version 11.2.0.1.2
Copyright (c) 2010, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/bea/tuxedo
Central Inventory : /srv/dpk/oracle
   from           : /etc/oraInst.loc
OPatch version    : 11.2.0.1.2
OUI version       : 12.2.0.1.0
  ...

List of Homes on this system:

  Home name= OraClient19Home1, Location= "/opt/oracle/psft/pt/oracle-client/19.3"
Inventory load failed... OPatch cannot load inventory for the given Oracle Home.
Possible causes are:
   Oracle Home dir. path does not exist in Central Inventory
   Oracle Home is a symbolic link
   Oracle Home inventory is corrupted
LsInventorySession failed: OracleHomeInventory gets null oracleHomeInfo

OPatch failed with error code 73
```

If I specify the inventory location by adding the `-invPtrLoc` parameter,
specifying the `invPrt.loc`. file in the Oracle Home, it works:

```console
$ ORACLE_HOME=/opt/oracle/psft/pt/bea/tuxedo     \
    JAVA_HOME=/opt/oracle/psft/pt/jdk            \
    /opt/oracle/psft/pt/bea/tuxedo/OPatch/opatch \
    lsinventory                                  \
    -invPtrLoc /opt/oracle/psft/pt/bea/tuxedo/oraInst.loc 
Invoking OPatch 11.2.0.1.2

Oracle Interim Patch Installer version 11.2.0.1.2
Copyright (c) 2010, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/bea/tuxedo
Central Inventory : /opt/oracle/psft/db/oraInventory
   from           : ./oraInst.loc
OPatch version    : 11.2.0.1.2
OUI version       : 12.2.0.1.0
OUI location      : /opt/oracle/psft/pt/bea/tuxedo/oui
  ...

--------------------------------------------------------------------------------
Installed Top-level Products (1): 

Oracle Tuxedo                                                        12.2.2.0.0
There are 1 products installed in this Oracle Home.


Interim patches (1) :

Patch  31786661     : applied on Thu Mar 31 11:42:51 BST 2022
   Created on 8 Sep 2020, 18:31:46 hrs PST8PDT
   Bugs fixed:
     25219794, 25083037, 23742025, 23549348, 23597937, 28808198, 23184083
  ...
     23294719, 24683124, 31681913, 25072979, 30386880
   This patch overlays patches:
     23024075, 23120181, 23029325, 23000300, 23034552, 22829476, 23085242
  ...
     31681913, 29599988


--------------------------------------------------------------------------------

OPatch succeeded.
```

So we can see we have Tuxedo 12.2.2.0.0 installed, and we need a patch for it.

### Optionally Attach the Home to the Central Inventory

The home can be attached to the central inventory using the following command:

```bash
ORACLE_HOME=/opt/oracle/psft/pt/bea/tuxedo \
    /opt/oracle/psft/pt/bea/tuxedo/oui/bin/attachHome.sh
```

This will save using the `-invPtrLoc` argument. I didn't do this, so I will
need to keep specifying that argument.


### Finding the Tuxedo Patch

Tuxedo is a component of Fusion Middleware. We find the patches by
following the security advisory 
[like we did for WebLogic](../patchingpeoplesoftwebserver//#finding-the-patches-by-following-the-security-advisory)
From the latest security advisory:
* Click on one of the entries for *Fusion Middleware* in the right column which opens Oracle [Doc ID 2853458.2](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2853458.2)
* Click the *Products* button
* This time we need to click on *Oracle Tuxedo*
* Click on the *Click here* link in the *Patch Advisor* column
* [Oracle Support Doc ID 2806740.2](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2806740.2#TUX1222) opens, this time the Tuxedo tab is already selected for us.
* In *Step 3: Update Tuxedo Products*, we can see there are three patches. I note that the one for *SALT*, and the one for *TSAM* are old. If they are required, they will already be in the DPK. It is only the current quarters patches that are missing.
* Click to download the patch from the table *Oracle Tuxedo 12.2.2*
* The patch download page displays. Ensure the correct operating system is selected. I find the web page has a tendency to change the OS back to Oracle Solaris, so double check it! If you get an error applying the patch this might be why!


### Shut down the Application Servers


We can use systemd to do this as `root`
```bash
systemctl stop psft-appserver-APPDOM.service
systemctl stop psft-prcs-PRCSDOM.service
```

Or we can switch users to `psadm2` and use psadmin.


### Updating OPatch

The zip file contains a README.txt file containing instructions to apply the
patch. We need to check if the OPatch version is high enough:

```console
$ ORACLE_HOME=/opt/oracle/psft/pt/bea/tuxedo       \
    JAVA_HOME=/opt/oracle/psft/pt/jdk              \
    /opt/oracle/psft/pt/bea/tuxedo/OPatch/opatch   \
    version                                                       
Invoking OPatch 11.2.0.1.2

OPatch Version: 11.2.0.1.2

OPatch succeeded.
```

That isn't correct, it needs to be version 12.1.0.1.0 or later. So we need to 
download it from patch `6880880`. I find the readme to be rather unclear. It 
says:

> In ARU, select the patch for release 18.0.0.0.0 

ARU is Automated Release Update, only accessable to Oracle staff. So that's
impossible for us to do. Looking in the Oracle Support Portal I can see
there are opatch versions for Fusion Middleware, but they are
all really old. It seems Oracle want us to download the version for the
database. I downloaded

> OPatch 12.2.0.1.30 for DB 18.0.0.0.0 (Apr 2022)(Patch)

But I am not patching a database, so that doesn't make sense. But it 
seems to work. It is installed by unzipping the
file. As `psadm1`:

```bash
cd /opt/oracle/psft/pt/bea/tuxedo
rm -rf OPatch
unzip p6880880_180000_Linux-x86-64.zip
```

### Applying the Tuxedo Patch

Now that we have the correct version of OPatch, we can apply the patch. We need
to unzip the downloaded zip file to extract the actual patch zip file that
we can apply. Once again I don't have `fuser` installed and OPatch needs it.
This version of OPatch doesn't respect the `OPATCH_NO_FUSER` variable,
so I created my own dummy fuser in the path. As root:

```bash
ln -s /bin/true /usr/local/bin/fuser
```

Then I can apply the patch as `psadm1`. As always long lines have been edited
in the following output.

```console
$ ORACLE_HOME=/opt/oracle/psft/pt/bea/tuxedo     \
    JAVA_HOME=/opt/oracle/psft/pt/jdk            \
    /opt/oracle/psft/pt/bea/tuxedo/OPatch/opatch \
    apply 33735306.zip                           \
    -silent                                      \
    -invPtrLoc /opt/oracle/psft/pt/bea/tuxedo/oraInst.loc 

Oracle Interim Patch Installer version 12.2.0.1.30
Copyright (c) 2022, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/bea/tuxedo
Central Inventory : /opt/oracle/psft/db/oraInventory
   from           : /opt/oracle/psft/pt/bea/tuxedo/oraInst.loc
OPatch version    : 12.2.0.1.30
OUI version       : 12.2.0.1.0
Log file location : opatch.log

Verifying environment and performing prerequisite checks...
OPatch continues with these patches:   33735306  

Do you want to proceed? [y|n]
Y (auto-answered by -silent)
User Responded with: Y
All checks passed.
Backing up files...
Applying interim patch '33735306' to OH '/opt/oracle/psft/pt/bea/tuxedo'

Patching component joltJrly, 12.2.2.0.0...

Patching component tuxedoClientCorbaCore, 12.2.2.0.0...

Patching component tuxedoServer, 12.2.2.0.0...

Patching component joltClient, 12.2.2.0.0...

Patching component Tuxedo, 12.2.2.0.0...

Patching component tuxedoClientCore, 12.2.2.0.0...

Patching component tuxedoClientAtmiCore, 12.2.2.0.0...
Patch 33735306 successfully applied.
Log file location: opatch.log

OPatch succeeded.
```

So that's Tuxedo done. Now on to the database client!


## Database Client

Once again the database client is installed on all tiers, so should be patched
on all tiers.

### What is Installed

The database client is installed (by default) as `oracle2`, so that is the user
we need to use to maintain it. So as `oracle2` we can run the following to see
what is installed:

```console
$ /opt/oracle/psft/pt/oracle-client/19.3.0.0/OPatch/opatch lsinventory
Oracle Interim Patch Installer version 12.2.0.1.28
Copyright (c) 2022, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/oracle-client/19.3.0.0
Central Inventory : /srv/dpk/oracle
   from           : /opt/oracle/psft/pt/oracle-client/19.3.0.0/oraInst.loc
OPatch version    : 12.2.0.1.28
OUI version       : 12.2.0.7.0
Log file location : opatch.log

Lsinventory Output file location : lsinventory.txt
--------------------------------------------------------------------------------
Local Machine Information::
Hostname: licalhost
ARU platform id: 226
ARU platform description:: Linux x86-64

Installed Top-level Products (1): 

Oracle Client 19c                                                    19.0.0.0.0
There are 1 products installed in this Oracle Home.


Interim patches (1) :

Patch  33515361     : applied on Thu Mar 31 11:30:58 BST 2022
Unique Patch ID:  24589353
Patch description:  "Database Release Update : 19.14.0.0.220118 (33515361)"
   Created on 13 Jan 2022, 06:14:07 hrs UTC
   Bugs fixed:
     7391838, 8460502, 8476681, 14570574, 14735102, 15931756, 15959416
... 
     33656608, 33674035, 33661960, 30166257, 33384092, 33618962

--------------------------------------------------------------------------------

OPatch succeeded.
```

We could also use lspatches, which gives a shorter output.

```console
$ /opt/oracle/psft/pt/oracle-client/19.3.0.0/OPatch/opatch lspatches
33515361;Database Release Update : 19.14.0.0.220118 (33515361)

OPatch succeeded.
```

So we see Oracle have applied the last quarters release update. We just need
to apply the one from this quarter. 


### Downloading the Patch

From the [security alerts](https://www.oracle.com/security-alerts/) page, we
follow the link to 
[this quarters Critical Patch Update announcement](https://www.oracle.com/security-alerts/cpuapr2022.html).
Normally I don't pay much attention to the actual security alerts, as we tend
to apply the security updates as fast as we can regardless of their severity.
However, since this patch is a database roll up, you might decide that being
a quarter behind is OK if none of the security alerts affect the client.
If you want to have a look, then click on the text in the left hand column:
[Oracle Database Server, versions 12.1.0.2, 19c, 21c](https://www.oracle.com/security-alerts/cpuapr2022.html#AppendixDB)
We can see this quarter than all of the security alerts affect a running database
rather than the client. However, lets patch it anyway. It could be argued
vulnerable code on the operating system might be used in some way by an 
attacker. Also I'm writing automation so I need to know how to patch the
client in case we do need to apply a client patch.

* Click on *Database* in the right hand column of the security alerts page.
* Oracle support [Doc ID 2844795.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2844795.1) displays.
* We noted above that the database client was version 19, so we need section [3.1.7.3 Oracle Database 19](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2844795.1#orcl19)
* Looking at the table *Patch Availability for Oracle Database 19* we want the second row titled *Oracle Database Server home*, and from there we need to download the Database Release Update 19.15.0.0.220419 [Patch 33806152](https://support.oracle.com/epmos/faces/ui/patch/PatchDetail.jspx?patchId=33806152)


### Downloading OPatch

Attempting to apply the patch gives the following error (Despite the README 
not mentioning a minimum OPatch version):

```console
The OPatch being used is version 12.2.0.1.28 while the following 
       patch(es) require higher versions: 
Patch 33806152 requires OPatch version 12.2.0.1.29 or later.
Please download the latest OPatch from My Oracle Support.
```

Let's download the latest one.

Downloading OPatch is similar to above.
* Log in to [Oracle Support](https://support.oracle.com)
* Select the *Patches & Updates* tab
* Search for patch *6880880* for your operating system (In my case *Linux x86-64*)
* Find the patch with description *OPatch 12.2.0.1.30 for DB 19.0.0.0.0 (Apr 2022) (Patch)* Of course the version and date will change.
* Select the line and click Download.


### Upgrading OPatch

Similarly to what we did with [Tuxedo](#updating-opatch). As user `oracle2`

```bash
cd /opt/oracle/psft/pt/oracle-client/19.3.0.0
rm -rf OPatch
unzip p6880880_190000_Linux-x86-64.zip
```

### Patching the client

Now we can patch the client. As normal it is good practice to shut things down
before patching. `OPATCH_NO_FUSER` is not honoured, so I created my own fuser
as with Tuxedo. As `root`

```bash
ln -s /bin/true /usr/local/bin/fuser
```

To apply the patch, as `oracle2` run the following

```console
$ /opt/oracle/psft/pt/oracle-client/19.3.0.0/OPatch/opatch  \
    apply p33806152_190000_Linux-x86-64.zip -silent
Oracle Interim Patch Installer version 12.2.0.1.30
Copyright (c) 2022, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/oracle-client/19.3.0.0
Central Inventory : /srv/dpk/oracle
   from           : /opt/oracle/psft/pt/oracle-client/19.3.0.0/oraInst.loc
OPatch version    : 12.2.0.1.30
OUI version       : 12.2.0.7.0

Verifying environment and performing prerequisite checks...
OPatch continues with these patches:   33806152  

Do you want to proceed? [y|n]
Y (auto-answered by -silent)
User Responded with: Y
All checks passed.

Please shutdown Oracle instances running out of this ORACLE_HOME
(Oracle Home = '/opt/oracle/psft/pt/oracle-client/19.3.0.0')


Is the local system ready for patching? [y|n]
Y (auto-answered by -silent)
User Responded with: Y
Backing up files...
Applying interim patch '33806152' to OH 
    '/opt/oracle/psft/pt/oracle-client/19.3.0.0'
ApplySession: Optional component(s) [ oracle.rdbms.locator, 19.0.0.0.0 ]
...
    [ oracle.jdk, 1.8.0.191.0 ]
    not present in the Oracle Home or a higher version is found.

Patching component oracle.bali.ewt, 11.1.1.6.0...

...

Patching component oracle.precomp.lang, 19.0.0.0.0...

Patching component oracle.jdk, 1.8.0.201.0...
Patch 33806152 successfully applied.
Sub-set patch [33515361] has become inactive due to the
    application of a super-set patch [33806152].
Please refer to Doc ID 2161861.1 for any possible further required actions.
Log file location: opatch.log

OPatch succeeded.

```
Sub patch 33515361 is the previous quarters CPU which was applied by Oracle,
so that makes sense. 
[Doc ID 2161861.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2161861.1)
refers to rolling back patches. The new functionality is that the previous
patch is inactivated rather than rolled back. If the patch is rolled back,
the previous roll-up patch is reactivated, which actually makes more sense.
Anyway, there is nothing else we need to do, so we can restart the
processes.


### Things I do not Understand

There is still a vulnerable version of Java in the database client home:

```console
$ jdk/bin/java -version
java version "1.8.0_321"
Java(TM) SE Runtime Environment (build 1.8.0_321-b07)
Java HotSpot(TM) 64-Bit Server VM (build 25.321-b07, mixed mode)
```

even though it is fully patched. So the latest tools patch contains the
oracle client patched from last quarter, and the Java within the client
home from six months ago. Also it is version 1.8 so to patch that I would
have to download that version of Java. 

In 
[Understanding PeopleSoft Deployment Packages for Update Images](https://docs.oracle.com/cd/F34168_01/psft/pdf/Understanding_PeopleSoft_Deployment_Packages_for_Update_Images.pdf) 
Oracle say:

> DPKs allow fast deployment of a PeopleSoft environment on supported hardware
> platform allowing you to skip the manual steps associated with the following:
> * Installing third-party products such as Oracle Tuxedo and WebLogic and the latest patches (CPUs)

That is simply not true. I have had to apply 5 patches on each tier.

## Conclusion

So now we have patched the web, application and process scheduler tiers with
the latest critical patch updates. We don't need to wait for Oracle to deliver
the infra-dpks before we patch, which may enable us to be more secure.