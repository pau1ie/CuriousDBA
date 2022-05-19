---
title: "PeopleSoft 8.59 Infra DPK and Critical Patch Updates"
date: 2022-05-19T09:11:07+01:00
tags: ["Weblogic","PeopleSoft","Critical Patch","Java","Linux"]
---

As I write this it is a month after Oracle released the critical patch updates,
but there is still no sign of the *Infra DPK*, which contains Java and WebLogic
updates. If Oracle are not going to supply this patch reliably, we will have to
work out how to do it ourselves.

## WebLogic (On the Web Server)

### What WebLogic Patches does Oracle Install?

In a default install we can do the following as user `psadm1`:

```bash
ORACLE_HOME=/opt/oracle/psft/pt/bea /opt/oracle/psft/pt/bea/OPatch/opatch lsinventory
```
The following is displayed (Trimmed)

```console {hl_lines=[26,34,42,49]}
Oracle Interim Patch Installer version 13.9.4.2.8
Copyright (c) 2022, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/bea
Central Inventory : /opt/oracle/psft/db/oraInventory
   from           : /opt/oracle/psft/pt/bea/oraInst.loc
OPatch version    : 13.9.4.2.8
OUI version       : 13.9.4.0.0
Log file location : /opt/oracle/psft/pt/bea/cfgtoollogs/opatch/blah.log


OPatch detects the Middleware Home as "/opt/oracle/psft/pt/bea"

Lsinventory Output file location : /opt/oracle/psft/pt/bea/cblah.txt

--------------------------------------------------------------------------------
Local Machine Information::
Hostname: localhost
ARU platform id: 226
ARU platform description:: Linux x86-64


Interim patches (4) :

Patch  33902209     : applied on Fri Apr 29 15:06:15 BST 2022
Unique Patch ID:  24662060
Patch description:  "Bundle patch for Oracle Coherence Version 14.1.1.0.9"
   Created on 25 Mar 2022, 13:00:25 hrs PST8PDT
   Bugs fixed:
     31201347, 31214284, 31806281, 31944953, 32124447, 32341371, 32581868
     ...

Patch  34011596     : applied on Fri Apr 29 15:05:46 BST 2022
Unique Patch ID:  24705941
Patch description:  "WLS PATCH SET UPDATE 14.1.1.0.220329"
   Created on 29 Mar 2022, 11:10:18 hrs PST8PDT
   Bugs fixed:
     33380581, 32697451, 30961904, 33328978, 33828242, 31047981, 33063225
     ...

Patch  32720458     : applied on Thu Mar 31 11:40:32 BST 2022
Unique Patch ID:  24558359
Patch description:  "JDBC 19.3.0.0 FOR CPUJAN2022 (WLS 12.2.1.4, WLS 14.1.1)"
   Created on 26 Aug 2021, 01:10:18 hrs UTC
   Bugs fixed:
     32720458

Patch  33678607     : applied on Thu Mar 31 11:40:13 BST 2022
Unique Patch ID:  24558585
Patch description:  "RDA release 20.4-20211126 for OFM SPB"
   Created on 17 Dec 2021, 07:18:05 hrs PST8PDT
   Bugs fixed:
     31308887, 30850214, 30430187, 30067211, 30108739, 29639586, 29687335
     ...


--------------------------------------------------------------------------------

OPatch succeeded.
```

So Oracle apply the following patches to the delivered home:
  * Bundle patch for Oracle Coherence Version 14.1.1.0.9
  * WLS PATCH SET UPDATE 14.1.1.0.220329
  * JDBC 19.3.0.0 FOR CPUJAN2022 (WLS 12.2.1.4, WLS 14.1.1)
  * RDA release 20.4-20211126 for OFM SPB
  
So we should try and find those patches.


### Finding the Patches by Following the Security Advisory

PeopleTools 8.59 comes with WebLogic 14.1.1, so we need to search for patches for that.
To find WebLogic patches, I navigate as follows.

* [Oracle Critical Patch Updates](https://www.oracle.com/security-alerts/)
* Click on the latest advisory, currently [April 2022](https://www.oracle.com/security-alerts/cpuapr2022.html)
* Click on one of the entries for *Fusion Middleware* in the right column which opens Oracle [Doc ID 2853458.2](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2853458.2)
* Click the *Products* button
* Click the *Oracle WebLogic Server* button at the bottom of the menu that opens
* A list of patches opens. If you think we need the Oracle WebLogic Server 14.1.1.0.0 patch which this quarter is [Patch 34011596](https://support.oracle.com/epmos/faces/ui/patch/PatchDetail.jspx?patchId=34011596), you would be wrong. Ignore that, it is only one of the patches we needed. 
* On the list of patches, click the *Click Here* link in the *Patch Advisor* column. 
* [Oracle Support Doc ID 2806740.2](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2806740.2#WLS1411) opens, handily scrolled to the correct position.
* Download WebLogic Server Stack Patch Bundle [patch 34084007](https://support.oracle.com/epmos/faces/ui/patch/PatchDetail.jspx?patchId=34084007) from table *Option 1: Download and apply the WebLogic Server Stack Patch Bundle (SPB):* in section *Step 3: Apply Patches to Oracle WebLogic Server 14.1.1* 
 option 1 section.
* Also download any overlay patches required. This quarter we need [merge request patch 34120447](https://support.oracle.com/epmos/faces/ui/patch/PatchDetail.jspx?patchId=34120447).

I find the patch download page says it is erroring due to too many redirects,
then the patch downloads four times.
There are often one off patches which need to be applied
in addition to the main WebLogic patch, so worth being prepared for this.


### Extract the WebLogic Server Stack Patch Bundle

Downloading the WebLogic Server Stack Patch Bundle gives us all the patches we
need, and also the samples. It won't open in Windows Explorer, so we need to
use [7-zip](https://www.7-zip.org/) or similar to unpack the file. The readme
suggests using java:
```bash
jar -xvf p34084007_141100_Generic.zip
```
Or on Unix unzip can be used:

```bash
unzip p34084007_141100_Generic.zip
```

Either way I end up with a directory called `WLS_SPB_14.1.1.0.220418` with all
the patch files in, which is nice.


### Run the WebLogic Server Stack Patch Bundle Pre-Checks

Now I can run the SPBAT as user `psadm1`:

```bash
cd WLS_SPB_14.1.1.0.220418/tools/spbat/generic/SPBAT
./spbat.sh -phase precheck -oracle_home /opt/oracle/psft/pt/bea
```
And the following output appears (edited for width).
```console
SPBAT Release Version: 2.0.2
The current patching user psadm1 matches with the product install user psadm1
-log_dir value is not set, defaulting it to WLS_SPB/tools/spbat/generic/SPBAT/logs
The consolidated SPBAT execution logs have been written to: logs/spbat.out 
Do not close this terminal as SPBAT precheck phase is currently executing...
[2022-05-17_15-29-18] Log file : spbat-precheck.log
[2022-05-17_15-29-20] [INFO] Oracle Home /opt/oracle/psft/pt/bea is registered
[2022-05-17_15-29-20] [SUCCESS] /opt/oracle/psft/pt/bea Middleware Home is present
[2022-05-17_15-29-20] [SUCCESS] The current jdk release 11.0 is a supported JDK
[2022-05-17_15-29-20] Log file : spbat-precheck.log
[2022-05-17_15-29-21] Minimum OPatch version required : 13.9.4.2.5
[2022-05-17_15-29-21] Environment has OPatch version : 13.9.4.2.8
[2022-05-17_15-29-21] [SUCCESS] Minimum OPatch version check
[2022-05-17_15-29-29] Log file : spbat-precheck.log
[2022-05-17_15-29-30] [WARNING] The Installed JDK version 11.0_14 should be
                                upgraded to the recommended JDK version 11.0_15
[2022-05-17_15-29-30] Refer to the SPB Readme for JDK installation and upgrade info
[2022-05-17_15-29-31] Log file : spbat-precheck.log
[2022-05-17_15-29-32] Middleware OPatch Version : 13.9.4.2.8
[2022-05-17_15-29-32] SPB OPatch version : 13.9.4.2.8
[2022-05-17_15-29-42] The environment already has the supported version of OPatch
[2022-05-17_15-29-51] List of patches present in the Oracle Home:

33902209;Bundle patch for Oracle Coherence Version 14.1.1.0.9
34011596;WLS PATCH SET UPDATE 14.1.1.0.220329
32720458;JDBC 19.3.0.0 FOR CPUJAN2022 (WLS 12.2.1.4, WLS 14.1.1)
33678607;RDA release 20.4-20211126 for OFM SPB

[2022-05-17_15-29-51] Patch compatibility check with the environment is in progress
[2022-05-17_15-30-27] CheckForNoOpPatches has Completed on /opt/oracle/psft/pt/bea
[2022-05-17_15-30-38] PATCH 33868014 APPLY WILL BE SKIPPED AS IT IS NOT APPLICABLE
[2022-05-17_15-30-39] PATCH 34011596 IS #ALREADY APPLIED# IN THE ENVIRONMENT
[2022-05-17_15-30-39] PATCH 34084037 IS #NOT APPLIED# IN THE ENVIRONMENT
[2022-05-17_15-30-40] PATCH 33902209 IS #ALREADY APPLIED# IN THE ENVIRONMENT
[2022-05-17_15-30-40] PATCH 34077664 IS #NOT APPLIED# IN THE ENVIRONMENT
[2022-05-17_15-30-40] PATCH 32720458 IS #ALREADY APPLIED# IN THE ENVIRONMENT
[2022-05-17_15-30-40] Patch conflict check is in progress ...
[2022-05-17_15-30-53] Patch conflict check has completed on /opt/oracle/psft/pt/bea
[2022-05-17_15-30-53] Log file : spbat-precheck.log
[2022-05-17_15-30-54] Napply precheck report is in progress ...
[2022-05-17_15-31-49] Napply precheck report has SUCCESSFULLY completed for wls
[2022-05-17_15-31-49] SPBAT precheck phase has completed. Summary as below:
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
PRECHECK SUMMARY: 
No Of FAILURES: 0
No Of WARNINGS: 1
[2022-05-17_15-31-49] Log file : spbat-precheck.log
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 
SPBAT precheck phase has completed successfully
Time Taken to run precheck phase:  00 hours 02 min 53 secs

```

This is nice, it checks what needs to be applied, and suggests I upgrade Java
to 11.0_15. I'm going to! 


### Apply the WebLogic Server Stack Patch Bundle (Take 1)

Lets shut down the webserver and apply the patches
then:

You may remember I 
[use systemd to auto restart WebLogic](../autorestartweblogicwithsystemd/), 
so I need to use systemd to stop it as `root`, as follows:
```bash
systemctl stop psft-pia-peoplesoft.service
```
Now I run the SPBAT apply to install the patches. As `psadm1` I run
```bash
./spbat.sh -phase apply -oracle_home /opt/oracle/psft/pt/bea
```

The same checks are run as above, then the software is installed.

```console
[2022-05-17_16-09-30] Application of patches is in progress ...
[05/17/2022 16:10:08] - [ERROR] - Opatch Napply has failed while Applying patches
[2022-05-17_16-10-08] Napply Exit Status - 73
[2022-05-17_16-10-08] Check the Log wls-Napply.log for more information.

[2022-05-17_16-10-08] Log file : spbat-apply.log
 
SPBAT apply phase has FAILED
Time Taken to run apply phase:  00 hours 03 min 33 secs
The console output has been written to: spbat.out
```

...or not! Let's have a look at that file:

```console {hl_lines=[33,36]}
$ cat wls-Napply.log
Oracle Interim Patch Installer version 13.9.4.2.8
Copyright (c) 2022, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/bea
Central Inventory : /opt/oracle/psft/db/oraInventory
   from           : /opt/oracle/psft/pt/bea/oraInst.loc
OPatch version    : 13.9.4.2.8
OUI version       : 13.9.4.0.0
Log file location : opatch.log


OPatch detects the Middleware Home as "/opt/oracle/psft/pt/bea"

Verifying environment and performing prerequisite checks...
Skip patch 33868014 from list of patches to apply: This patch is not needed.

The following patches are duplicate and are skipped:
[ 34011596 33902209 32720458  ]


Conflicts/Supersets for each patch are:

Patch : 34077664

        Bug Superset of 33678607
        Super set bugs are:
        28800895, 30108739, 23723609, 17966756, 24696555, 22218552, 

Prerequisite check "CheckActiveFilesAndExecutables" failed.
The details are:
Exception occured :     fuser could not be located:
Prerequisite check "CheckActiveFilesAndExecutables" failed.
The details are:
Exception occured :     fuser could not be located:
Log file location: opatch.log

OPatch failed with error code 73
```

OK, we need fuser. We could install it, or we could follow the advice in
[Oracle Support Document 2777501.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2777501.1)


### Apply the WebLogic Server Stack Patch Bundle (Take 2)

Let's try that again then. With WebLogic still shut down we run the following
as `psadm1`

```bash
OPATCH_NO_FUSER=true ./spbat.sh -phase apply -oracle_home /opt/oracle/psft/pt/bea
```

Once again we get the same precheck output as before. We also get the following messages:

```console
[2022-05-17_16-20-51] Application of patches is in progress ...
[2022-05-17_16-22-03] SUCCESSFUL - OPatch napply has completed for wls Home 
[2022-05-17_16-22-03] Opatch Napply Exit Status - 0
[2022-05-17_16-22-03] COMPLETED : Performing SPBAT Binary patching on wls Home
[2022-05-17_16-22-11] STARTED : Performing SPBAT binary audit on wls Home
[2022-05-17_16-22-21] NoOp patch#33868014# detected in Environment.Skipping Audit
[2022-05-17_16-22-22] SUCCESSFUL - SPB PATCH 34011596 IS #APPLIED#
[2022-05-17_16-22-22] SUCCESSFUL - SPB PATCH 34084037 IS #APPLIED#
[2022-05-17_16-22-23] SUCCESSFUL - SPB PATCH 33902209 IS #APPLIED#
[2022-05-17_16-22-23] SUCCESSFUL - SPB PATCH 34077664 IS #APPLIED#
[2022-05-17_16-22-24] SUCCESSFUL - SPB PATCH 32720458 IS #APPLIED#
[2022-05-17_16-22-33] List of patches present in the Oracle Home:

34084037;WLS STACK PATCH BUNDLE 14.1.1.0.220418 (Patch 34084007)
34077664;RDA release 22.2-20220307 for OFM SPB
33902209;Bundle patch for Oracle Coherence Version 14.1.1.0.9
34011596;WLS PATCH SET UPDATE 14.1.1.0.220329
32720458;JDBC 19.3.0.0 FOR CPUJAN2022 (WLS 12.2.1.4, WLS 14.1.1)

[2022-05-17_16-22-33] Log file : spbat-apply.log
 
SPBAT apply phase has completed successfully
Time Taken to run apply phase:  00 hours 04 min 37 secs
Perform the post install actions as documented in the SPB README.txt

```
That looks more promising!
Now we need to follow *Section 5: Post Installation Steps* of 
`WLS_SPB/README.html` which says to check the patches are installed using
`opatch lspatches` I am pretty sure that's what happened above, but
lets try it anyway:

```bash
ORACLE_HOME=/opt/oracle/psft/pt/bea /opt/oracle/psft/pt/bea/OPatch/opatch lspatches 
```
And we get the following
```console
34084037;WLS STACK PATCH BUNDLE 14.1.1.0.220418 (Patch 34084007)
34077664;RDA release 22.2-20220307 for OFM SPB
33902209;Bundle patch for Oracle Coherence Version 14.1.1.0.9
34011596;WLS PATCH SET UPDATE 14.1.1.0.220329
32720458;JDBC 19.3.0.0 FOR CPUJAN2022 (WLS 12.2.1.4, WLS 14.1.1)

OPatch succeeded.
```
Great! We have finally applied the WebLogic patches!


### Applying the Interim Patch

Looks good! Now we need to apply that interim (sometimes called an *overlay*)
patch. I think these are patches which were created and passed testing after
the deadline for creating the rollup patch. They still deliver security fixes,
so still need to be applied. Looking at the readme we just need to check the
opatch version is high enough, then install it using opatch.

Let's see if OPatch is good enough. The readme says we need
*OPatch version 13.9.4.2.8 or higher*
```console
$ ORACLE_HOME=/opt/oracle/psft/pt/bea /opt/oracle/psft/pt/bea/OPatch/opatch version
OPatch Version: 13.9.4.2.8

OPatch succeeded.
```
That looks good to me. Lets run the patch:

```console {hl_lines=[20]}
$ ORACLE_HOME=/opt/oracle/psft/pt/bea /opt/oracle/psft/pt/bea/OPatch/opatch \
    apply p34120447_141100_Generic.zip
Oracle Interim Patch Installer version 13.9.4.2.8
Copyright (c) 2022, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/bea
Central Inventory : /opt/oracle/psft/db/oraInventory
   from           : /opt/oracle/psft/pt/bea/oraInst.loc
OPatch version    : 13.9.4.2.8
OUI version       : 13.9.4.0.0
Log file location : /opt/oracle/psft/pt/bea/cfgtoollogs/opatch/opatch.log


OPatch detects the Middleware Home as "/opt/oracle/psft/pt/bea"

Verifying environment and performing prerequisite checks...
Skip patch 34120447 from list of patches to apply: This patch is not needed.

After skipping patches with missing components, there are no patches to apply.
OPatch Session completed with warnings.
Log file location: /opt/oracle/psft/pt/bea/cfgtoollogs/opatch/opatch.log
```

Looking at  the readme, it is just patching samples which aren't in the 
WebLogic home supplied with PeopleSoft. I think it might have been nice
if Oracle had mentioned that.


## Java (On the Web, Application and Process Scheduler tiers)
### Finding the Java Patch


This looks good. Now we need to upgrade Java like the tool above said. This
involves going on to My Oracle Support again.

Remember [for WebLogic](#finding-the-patches-by-following-the-security-advisory) we
followed the critical patch information for *Fusion Middleware* and 
*WebLogic*? Well for Java we follow the same procedure including clicking the
*Click Here* link on the *Patch Advisor* column
* [Oracle Support Doc ID 2806740.2](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2806740.2#WLS1411) opens, handily scrolled to the correct position.
* *Step 2* mentions the patch needed for Java SE 11. It looks like this might always be the patch number for the latest Java 11 SE, which would be useful.
* Download [Patch 27838191](https://support.oracle.com/epmos/faces/ui/patch/PatchDetail.jspx?patchId=27838191) as we use Java 11.
* Extract the patch zip file.
* Copy the tgz version of the patch to the server.

The outer zip file also contains a readme explaining that we need to
have a license to use Java. We have a license for PeopleSoft which allows
us to use WebLogic and Java.


### Installing the Java Patch

So I can install the new Java patch as follows. Log in as `psadm1` and run:

```bash
cd /opt/oracle/psft/pt
tar xvf jdk-11.0.15.1_linux-x64_bin.tar.gz 
rm -rf jdk
mv jdk-11.0.15.1 jdk
```

### Optional - Installing Java under a Different Name

Oracle doesn't recommended this approach, but
if we didn't want to delete the old directory and preserve the name,
[Doc ID 2806740.2](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2806740.2#WLS1411)
also points us at
[Oracle Support Doc ID 1492980.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=1492980.1#aref_section33)
which explains how to maintain Java for WebLogic. It lists the files to
update, but mentions there is an alternative scripting method, and links
to the manual for Oracle Internet Directory version 12. I feel a link to the
[Installing and Configuring Oracle WebLogic Server and Coherence](https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/wlsig/shared-updating-jdk-installing-and-configuring-oracle-fusion-middleware-product.html)
might be more appropriate. This mentions the java used by WebLogic can be
checked and set using `getProperty.sh` and `setProperty.sh`

```console
$ /opt/oracle/psft/pt/bea/oui/bin/getProperty.sh JAVA_HOME
/opt/oracle/psft/pt/jdk
$ /opt/oracle/psft/pt/bea/oui/bin/setProperty.sh -name JAVA_HOME \
    -value /opt/oracle/psft/pt/jdk-11.0.15.1
May 18, 2022 1:14:01 PM com.oracle.cie.nextgen.common.version.VersionHelper compare
WARNING: Invalid integer in version  or  at position 0
May 18, 2022 1:14:01 PM com.oracle.cie.nextgen.common.version.VersionHelper compare
WARNING: Reporting that versions are equal
Property JAVA_HOME successfully set to "/opt/oracle/psft/pt/jdk-11.0.15.1"
$ /opt/oracle/psft/pt/bea/oui/bin/getProperty.sh JAVA_HOME
/opt/oracle/psft/pt/jdk-11.0.15.1
```

It seems to have worked, but it seems to have trouble detecting version
numbers, so thinks they are both the same version, which doesn't inspire
confidence. The versions are different as can be seen below:

```console
$ /opt/oracle/psft/pt/jdk/bin/java -version
java version "11.0.14" 2022-01-18 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.14+8-LTS-263)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.14+8-LTS-263, mixed mode)

$ /opt/oracle/psft/pt/jdk-11.0.15.1/bin/java -version
java version "11.0.15.1" 2022-04-22 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.15.1+2-LTS-10)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.15.1+2-LTS-10, mixed mode)
```

I don't think I will do this, I will just replace Java in the generically
named directory.

### Java in PS_HOME

There is also a Java in `PS_HOME`. So we had better upgrade that one as well.
Oracle don't seem to have any documentation for Linux, but 
[Doc ID 2323346.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2323346.1)
mentions how to upgrade Java on Windows, we just need to copy the new Java
into place. Despite the directory naming it is a full JDK, there doesn't seem
to be a JRE delivered for Java 11. All we need to do is copy the Java in again:

As `psadm1` run the following (Noting that the ps_home name will change each 
quarter)

```bash
rm -rf /opt/oracle/psft/pt/ps_home8.59.09/jre
cp -pr /opt/oracle/psft/pt/jdk  /opt/oracle/psft/pt/ps_home8.59.09/jre
```

Version 8.58 used to supply extra font files for BI Publisher
under `$PS_HOME/jre/lib/fonts`. 8.59 has more sensibly moved those into their
own directory under `$PS_HOME/fonts`. This means we don't need to worry about
the critical patch overwriting the fonts folder, but any additional fonts
we deliver will probably also need to end up in that folder too.
