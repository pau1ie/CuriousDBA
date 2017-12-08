---
title: "Broken PeopleTools Deployment Packages"
date: 2017-11-08T12:13:40Z
image: images/broken-package.jpg
tags: ["Oracle","PeopleSoft","Weblogic","Critical Patch"]
---

The October Critical patch saw Oracle release 

> Patch 26743107: PT 8.55.19 PRODUCT PATCH LINUX DPK.

This is important
to install because there was a serious security vulnerability in the
performance monitor component which was easily exploitable over the network
without authorisation.

However, once the DPK was installed, looking at the Weblogic home, we can see
that the October critical patch 

> Patch 26519417: WLS PATCH SET UPDATE 12.1.3.0.171017

hasn't been applied. It would be nice if it
was delivered fully patched, but presumably Oracle's internal time scales don't
allow this, so I will have to patch it myself.

{{< highlight console >}}
[psadm1@webserver bea]$ pwd
/opt/oracle/psft/pt/bea
[psadm1@webserver bea]$ export ORACLE_HOME=/opt/oracle/psft/pt/bea
[psadm1@webserver bea]$ $ORACLE_HOME/OPatch/opatch lsinventory
Picked up _JAVA_OPTIONS: -Djava.security.egd=file:/dev/./urandom
Oracle Interim Patch Installer version 13.2.0.0.0
Copyright (c) 2014, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/bea
Central Inventory : /opt/oracle/psft/db/oraInventory
   from           : /opt/oracle/psft/pt/bea/oraInst.loc
OPatch version    : 13.2.0.0.0
OUI version       : 13.2.0.0.0
Log file location : /opt/oracle/psft/pt/bea/cfgtoollogs/opatch/opatch2017-11-07_15-31-58PM_1.log


OPatch detects the Middleware Home as "/opt/oracle/psft/pt/bea"

Nov 07, 2017 3:32:04 PM oracle.sysman.oii.oiii.OiiiInstallAreaControl initAreaControl
INFO: Install area Control created with access level  0
Lsinventory Output file location : /opt/oracle/psft/pt/bea/cfgtoollogs/opatch/lsinv/lsinventory2017-11-07_15-31-58PM.txt

--------------------------------------------------------------------------------

Interim patches (1) :

Patch  25869659     : applied on Tue Nov 07 15:27:09 GMT 2017
Unique Patch ID:  21297953
Patch description:  "WLS PATCH SET UPDATE 12.1.3.0.170718"
   Created on 3 Jul 2017, 03:20:19 hrs PST8PDT
   Bugs fixed:
     18538501, 18376812, 21746415, 18746515, 17394051, 19668883, 20333386
     14236278, 22498352, 21522926, 20720853, 20585084, 22910817, 23063611
     19472793, 20692185, 25029531, 22749253, 22107941, 21081720, 23099318
[snip]
     22049932, 21947902, 20220959, 19947189, 20783846, 20551651, 22666897
     18806464, 22999996, 20671165, 19865550, 22759067, 20432957, 20256190
     25375968, 25522149, 24817968, 25743025, 25823774, 19721047, 24828619


--------------------------------------------------------------------------------

OPatch succeeded.
{{< /highlight >}}

OK, they delivered it with 

> Patch 25869659: WLS PATCH SET UPDATE 12.1.3.0.170718

This is really simple stuff, we can download and apply the patch?

{{< highlight console >}}
$ /opt/oracle/psft/pt/bea/OPatch/opatch apply -silent \
    /psoft/peoplesoft_applications/CriticalPatches/weblogic/p26519417_121300_Generic.zip  \
    -oh /opt/oracle/psft/pt/bea -force_conflict
Picked up _JAVA_OPTIONS: -Djava.security.egd=file:/dev/./urandom
Oracle Interim Patch Installer version 13.2.0.0.0
Copyright (c) 2014, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/bea
Central Inventory : /opt/oracle/psft/db/oraInventory
   from           : /opt/oracle/psft/pt/bea/oraInst.loc
OPatch version    : 13.2.0.0.0
OUI version       : 13.2.0.0.0
Log file location : /opt/oracle/psft/pt/bea/cfgtoollogs/opatch/opatch2017-11-07_15-35-38PM_1.log


OPatch detects the Middleware Home as "/opt/oracle/psft/pt/bea"

Nov 07, 2017 3:35:49 PM oracle.sysman.oii.oiii.OiiiInstallAreaControl initAreaControl
INFO: Install area Control created with access level  0
Applying interim patch '26519417' to OH '/opt/oracle/psft/pt/bea'
Verifying environment and performing prerequisite checks...
Interim patch 26519417 is a superset of the patch(es) [  25869659 ] in the Oracle Home
OPatch will roll back the subset patches and apply the given patch.
Prerequisite check "CheckRollbackable" on auto-rollback patches failed.
The details are:

Patch 25869659:
Jar Action: Source file "/opt/oracle/psft/pt/bea/.patch_storage/25869659_Jul_3_2017_03_20_19/files/oracle.wls.libraries/12.1.3.0.0/wls.server.symbol/server/lib/weblogic-classes.jar/org/python/modules/sets/PyImmutableSet$1exposed_copy.class", does not exist or is not readable.
'oracle.wls.libraries, 12.1.3.0.0': Cannot update file '/opt/oracle/psft/pt/bea/wlserver/server/lib/weblogic-classes.jar' with '/org/python/modules/sets/PyImmutableSet$1exposed_copy.class'
Jar Action: Source file "/opt/oracle/psft/pt/bea/.patch_storage/25869659_Jul_3_2017_03_20_19/files/oracle.wls.libraries/12.1.3.0.0/wls.server.symbol/server/lib/weblogic-classes.jar/org/python/core/PyDictionary$1exposed_copy.class", does not exist or is not readable.
'oracle.wls.libraries, 12.1.3.0.0': Cannot update file '/opt/oracle/psft/pt/bea/wlserver/server/lib/weblogic-classes.jar' with '/org/python/core/PyDictionary$1exposed_copy.class'
Jar Action: Source file "/opt/oracle/psft/pt/bea/.patch_storage/25869659_Jul_3_2017_03_20_19/files/oracle.wls.libraries/12.1.3.0.0/wls.server.symbol/server/lib/weblogic-classes.jar/org/python/modules/sets/PySet$1exposed_copy.class", does not exist or is not readable.
'oracle.wls.libraries, 12.1.3.0.0': Cannot update file '/opt/oracle/psft/pt/bea/wlserver/server/lib/weblogic-classes.jar' with '/org/python/modules/sets/PySet$1exposed_copy.class'

Log file location: /opt/oracle/psft/pt/bea/cfgtoollogs/opatch/opatch2017-11-07_15-35-38PM_1.log

Recommended actions: Please roll back the conflict patches using 'opatch rollback' command.

OPatch failed with error code 70
{{< /highlight >}}

Oh dear. It seems the delivered home is corrupt. Of course we can't roll back the
patch as suggested, that is exactly what is failing, as OPatch tries to roll
back the patch before installing the new one.

I proved the home is corrupt by installing the dpk 

> Patch 25942984: PT 8.55.16 PRODUCT PATCH LINUX DPK.

Then I installed

> Patch 25869659: WLS PATCH SET UPDATE 12.1.3.0.170718. 

I can see this home has the 
missing files from the previous home in the .patch_storage directory. I made a copy
of them for later. I found that

> Patch 26519417: WLS PATCH SET UPDATE 12.1.3.0.171017

could be applied to this home as well.

So, the workaround is to copy the missing files into the patch storage 

{{< highlight bash >}}
cp /tmp/wls_patch_storage/PyDictionary\$1exposed_copy.class \
  /opt/oracle/psft/pt/bea/.patch_storage/25869659_Jul_3_2017_03_20_19/files/oracle.wls.libraries/12.1.3.0.0/wls.server.symbol/server/lib/weblogic-classes.jar/org/python/core/
cp /tmp/wls_patch_storage/PyImmutableSet\$1exposed_copy.class \
  /opt/oracle/psft/pt/bea/.patch_storage/25869659_Jul_3_2017_03_20_19/files/oracle.wls.libraries/12.1.3.0.0/wls.server.symbol/server/lib/weblogic-classes.jar/org/python/modules/sets/
cp /tmp/wls_patch_storage/PySet\$1exposed_copy.class \
  /opt/oracle/psft/pt/bea/.patch_storage/25869659_Jul_3_2017_03_20_19/files/oracle.wls.libraries/12.1.3.0.0/wls.server.symbol/server/lib/weblogic-classes.jar/org/python/modules/sets/
{{< /highlight >}}

Once these files were copied in, the install works:

{{< highlight console >}}
$ /opt/oracle/psft/pt/bea/OPatch/opatch apply -silent /psoft/peoplesoft_applications/CriticalPatches/weblogic/p26519417_121300_Generic.zip -oh /opt/oracle/psft/pt/bea -force_conflict
Picked up _JAVA_OPTIONS: -Djava.security.egd=file:/dev/./urandom
Oracle Interim Patch Installer version 13.2.0.0.0
Copyright (c) 2014, Oracle Corporation.  All rights reserved.


Oracle Home       : /opt/oracle/psft/pt/bea
Central Inventory : /opt/oracle/psft/db/oraInventory
   from           : /opt/oracle/psft/pt/bea/oraInst.loc
OPatch version    : 13.2.0.0.0
OUI version       : 13.2.0.0.0
Log file location : /opt/oracle/psft/pt/bea/cfgtoollogs/opatch/opatch2017-11-07_15-42-20PM_1.log


OPatch detects the Middleware Home as "/opt/oracle/psft/pt/bea"

Nov 07, 2017 3:42:29 PM oracle.sysman.oii.oiii.OiiiInstallAreaControl initAreaControl
INFO: Install area Control created with access level  0
Applying interim patch '26519417' to OH '/opt/oracle/psft/pt/bea'
Verifying environment and performing prerequisite checks...
Interim patch 26519417 is a superset of the patch(es) [  25869659 ] in the Oracle Home
OPatch will roll back the subset patches and apply the given patch.
All checks passed.

Please shutdown Oracle instances running out of this ORACLE_HOME on the local system.
(Oracle Home = '/opt/oracle/psft/pt/bea')


Is the local system ready for patching? [y|n]
Y (auto-answered by -silent)
User Responded with: Y
Backing up files...
Rolling back interim patch '25869659' from OH '/opt/oracle/psft/pt/bea'

Patching component oracle.wls.workshop.code.completion.support, 12.1.3.0.0...

Patching component oracle.wls.workshop.code.completion.support, 12.1.3.0.0...

Patching component oracle.css.mod, 12.1.3.0.0...

Patching component oracle.css.mod, 12.1.3.0.0...

Patching component oracle.fmwconfig.common.shared, 12.1.3.0.0...

Patching component oracle.fmwconfig.common.shared, 12.1.3.0.0...

Patching component oracle.wls.common.nodemanager, 12.1.3.0.0...

Patching component oracle.wls.common.nodemanager, 12.1.3.0.0...

Patching component oracle.webservices.base, 12.1.3.0.0...

Patching component oracle.webservices.base, 12.1.3.0.0...

Patching component oracle.wls.server.shared.with.core.engine, 12.1.3.0.0...

Patching component oracle.wls.server.shared.with.core.engine, 12.1.3.0.0...

Patching component oracle.wls.shared.with.cam, 12.1.3.0.0...

Patching component oracle.wls.shared.with.cam, 12.1.3.0.0...

Patching component oracle.webservices.orawsdl, 12.1.3.0.0...

Patching component oracle.webservices.orawsdl, 12.1.3.0.0...

Patching component oracle.wls.libraries.mod, 12.1.3.0.0...

Patching component oracle.wls.libraries.mod, 12.1.3.0.0...

Patching component oracle.wls.admin.console.en, 12.1.3.0.0...

Patching component oracle.wls.admin.console.en, 12.1.3.0.0...

Patching component oracle.wls.core.app.server, 12.1.3.0.0...

Patching component oracle.wls.core.app.server, 12.1.3.0.0...

Patching component oracle.webservices.wls, 12.1.3.0.0...

Patching component oracle.webservices.wls, 12.1.3.0.0...

Patching component oracle.wls.clients, 12.1.3.0.0...

Patching component oracle.wls.clients, 12.1.3.0.0...

Patching component oracle.wls.wlsportable.mod, 12.1.3.0.0...

Patching component oracle.wls.wlsportable.mod, 12.1.3.0.0...

Patching component oracle.fmwconfig.common.wls.shared, 12.1.3.0.0...

Patching component oracle.fmwconfig.common.wls.shared, 12.1.3.0.0...

Patching component oracle.wls.libraries, 12.1.3.0.0...

Patching component oracle.wls.libraries, 12.1.3.0.0...
RollbackSession removing interim patch '25869659' from inventory


OPatch back to application of the patch '26519417' after auto-rollback.


Patching component oracle.wls.workshop.code.completion.support, 12.1.3.0.0...

Patching component oracle.wls.workshop.code.completion.support, 12.1.3.0.0...

Patching component oracle.css.mod, 12.1.3.0.0...

Patching component oracle.css.mod, 12.1.3.0.0...

Patching component oracle.fmwconfig.common.shared, 12.1.3.0.0...

Patching component oracle.fmwconfig.common.shared, 12.1.3.0.0...

Patching component oracle.wls.common.nodemanager, 12.1.3.0.0...

Patching component oracle.wls.common.nodemanager, 12.1.3.0.0...

Patching component oracle.webservices.base, 12.1.3.0.0...

Patching component oracle.webservices.base, 12.1.3.0.0...

Patching component oracle.wls.server.shared.with.core.engine, 12.1.3.0.0...

Patching component oracle.wls.server.shared.with.core.engine, 12.1.3.0.0...

Patching component oracle.wls.shared.with.cam, 12.1.3.0.0...

Patching component oracle.wls.shared.with.cam, 12.1.3.0.0...

Patching component oracle.webservices.orawsdl, 12.1.3.0.0...

Patching component oracle.webservices.orawsdl, 12.1.3.0.0...

Patching component oracle.wls.libraries.mod, 12.1.3.0.0...

Patching component oracle.wls.libraries.mod, 12.1.3.0.0...

Patching component oracle.wls.admin.console.en, 12.1.3.0.0...

Patching component oracle.wls.admin.console.en, 12.1.3.0.0...

Patching component oracle.webservices.wls, 12.1.3.0.0...

Patching component oracle.webservices.wls, 12.1.3.0.0...

Patching component oracle.wls.core.app.server, 12.1.3.0.0...

Patching component oracle.wls.core.app.server, 12.1.3.0.0...

Patching component oracle.wls.clients, 12.1.3.0.0...

Patching component oracle.wls.clients, 12.1.3.0.0...

Patching component oracle.wls.wlsportable.mod, 12.1.3.0.0...

Patching component oracle.wls.wlsportable.mod, 12.1.3.0.0...

Patching component oracle.fmwconfig.common.wls.shared, 12.1.3.0.0...

Patching component oracle.fmwconfig.common.wls.shared, 12.1.3.0.0...

Patching component oracle.wls.libraries, 12.1.3.0.0...

Patching component oracle.wls.libraries, 12.1.3.0.0...

Verifying the update...
Patch 26519417 successfully applied
Log file location: /opt/oracle/psft/pt/bea/cfgtoollogs/opatch/opatch2017-11-07_15-42-20PM_1.log

OPatch succeeded.
{{< /highlight >}}

Hooray! Hopefully Oracle will fix this for the next DPK that is released.
