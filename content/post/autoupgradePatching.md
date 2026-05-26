---
title: "Patching using Auto Upgrade"
date: 2026-05-16T09:15:10+00:00
tags: ['Automation','Oracle']
---

Oracle are recommending we use out of place patching, and AutoUpgrade for
patching. Until now we have used a custom patch download script and an
ansible patching process. Let's see how AutoUpgrade works for patching.

Documentation is in the Oracle Database Upgrade guide under
[AutoUpgrade Patching](https://docs.oracle.com/en/database/oracle/oracle-database/19/upgrd/autoupgrade-patching.html).
As with many Oracle things, it takes some effort to understand how it is
supposed to work.


## Downloading Auto Upgrade

We download auto upgrade from Oracle Support document 
[KB123450](https://support.oracle.com/support/?documentId=KB123450),
or from the
[Oracle website](https://www.oracle.com/database/upgrades/)
So we can do:

```bash
wget https://download.oracle.com/otn-pub/otn_software/autoupgrade.jar
```

we can check the options of AutoUpgrade for patching using:

```bash
java -jar autoupgrade.jar -patch -help
```


## Downloading and Staging Patches

### Oracle Credentials

I am not happy about this, because this stores my personal credentials on
a server in a keyfile which apparently can be opened by AutoUpgrade
without providing a password. So a colleague (Or malware) could download
patches using my account without my permission.

According to AutoUpgrade: Passwords must have a minimum length of eight
characters and contain alphabetic characters combined with numbers or
special characters. 


```
$ java -jar autoupgrade.jar -config ./keystore.cfg -patch -load_password
Processing config file ...

Starting AutoUpgrade Patching Password Loader
  - Type help for available options
Creating new AutoUpgrade Patching keystore - Password required
Enter password:   
Enter password again:   
AutoUpgrade Patching keystore was successfully created

MOS> add -user my.email@example.com
Enter your secret/Password:   
Re-enter your secret/Password:   
MOS> exit
Save the AutoUpgrade Patching keystore before exiting [YES|NO] ? yes
Convert the AutoUpgrade Patching keystore to auto-login [YES|NO] ? yes

AutoUpgrade Patching Password Loader finished - Exiting AutoUpgrade Patching
$
```

### Configuration File

To download patches, we need to specify a list of patches to download.

```
global.keystore=/home/me/autoupgrade/keystore
global.global_log_dir=/home/me/autoupgrade/logs
patch1.folder=/home/me/autoupgrade/patches
patch1.patch=RECOMMENDED
patch1.target_version=19
```

This downloads all recommended patches when passed to AutoUpgrade:

```
$ java -jar autoupgrade.jar -patch -mode download -config config
AutoUpgrade Patching 26.3.260401 launched with default internal options
Processing config file ...
Loading AutoUpgrade Patching keystore
AutoUpgrade Patching keystore is loaded

Connected to MOS - Searching for specified patches

----------------------------------------------------
Downloading files to /home/me/autoupgrade/patches
----------------------------------------------------
DATABASE RELEASE UPDATE 19.31.0.0.0
    File: p39034528_190000_Linux-x86-64.zip - VALIDATED

OPatch 12.2.0.1.51 for DB 19.0.0.0.0 (Apr 2026)
    File: p6880880_190000_Linux-x86-64.zip - VALIDATED

DATAPUMP BUNDLE PATCH 19.31.0.0.0
    File: p39196236_1931000DBRU_Generic.zip - VALIDATED

OJVM RELEASE UPDATE 19.31.0.0.0
    File: p38906621_190000_Linux-x86-64.zip - VALIDATED

autoupgrade.jar 26.3 (April 2026)
    File: autoupgrade.jar - VALIDATED
----------------------------------------------------
 
```

## Creating the Oracle Home

### Base Image

AutoUpgrade doesn't download base or gold images. A base image can be
downloaded once and then upgraded whenever, while a gold image is already
patched. It seems easier to download the base image once, then patch it
as the home is being created. Our base image was downloaded in a zip file
called V982063-01.zip, but it seems autoupdate wants the base image to be
called  LINUX.X64_193000_db_home.zip, so we renamed it, and placed it in
the patch folder. Also if we are not using a gold image, we need to have
an existing home. This is done by specifying either a database, or a
source home.

The parameters to create and patch the home are documented in the
[AutoUpgrade guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/upgrd/patch-parameters-autoupgrade-config-file.html).

We need to ensure that the oracle inventory contains a copy of
`oraInst.loc`, otherwise we get an _invalid home_/_No  error when trying to
patch.

```bash
cp /etc/oraInst.loc "$(grep inventory_loc /etc/oraInst.loc | cut -f2 -d=)"
```

We need to run AutoUpgrade in create_home mode. First we create a
configuration. Call it (say) `create_home.cfg`.

```
global.global_log_dir=/oracle/AutoUpgrade/logs
crehome.download=no
crehome.patch=RU:19.31,OPATCH
crehome.target_home=/oracle/product/test19home
crehome.folder=/oracle/AutoUpgrade/patches
crehome.gold_image=no
crehome.home_settings.binopt.asm=no
crehome.home_settings.binopt.dnfs=no
crehome.home_settings.binopt.dm=no
crehome.home_settings.binopt.jox=yes
crehome.home_settings.binopt.olap=no
crehome.home_settings.binopt.part=no
crehome.home_settings.binopt.rac=no
crehome.home_settings.binopt.rat=no
crehome.home_settings.binopt.sdo=no
crehome.home_settings.binopt.uniaud=no
crehome.home_settings.edition=EE
crehome.home_settings.ignore_prereq_failure=yes
crehome.home_settings.oracle_base=/oracle
crehome.target_version=19
```

Then we run AutoUpgrade to install the home.

```
java -jar autoupgrade.jar -config create_home.cfg -patch -mode create_home
```

### When it fails

When the create home process failed, I found I had to remove the log
directory, because AutoUpgrade detects that a previous run had failed,
and refused to restart itself. The home hadn't registered itself, so I
removed that as well.


## Analyze/fixups

We are used to running the analyze and fixups step while upgrading, but
not while patching. We use the same file as below.

```
java -jar autoupgrade.jar -config create_home.cfg -patch -mode analyze
```

## Patching the Database

Lastly we patch the database itself. This is the downtime.

```
global.autoupg_log_dir=/oracle/AutoUpgrade/logs
patch.folder
patch.target_home=/oracle/product/old19home
patch.target_home=/oracle/product/test19home
patch.sid=UPGR
```

We do have to supply a source home. It seems AutoUpgrade can't work it
out by itself.

And we run it using
```
java -jar autoupgrade.jar -config create_home.cfg -patch -mode upgrade
```


The listener is not upgraded. We have one listener per database, so we
do the listener at the same time as the database. AutoUpgrade has
changed the listener.ora for us.

## Uninstalling the old home


Old homes need to be removed. We are on RHEL9, so we need to tell deinstall
to pretend we are on RHEL8.

```
cd /oracle/product/mytest19home/deinstall
CV_ASSUME_DISTID=RHEL8 ./deinstall
```
