---
title: "PeopleSoft Critical patches on Windows"
date: 2022-06-30T11:12:07+01:00
tags: ["PeopleSoft","Security","Fail","Windows","Critical Patch","Java","WebLogic"]
---

Back to my series about applying patches to PeopleTools 8.59. I am now visiting
the Windows process scheduler. Like the Unix servers, Oracle installs all the
software, so it all needs to be patched. But there are always additional
complications with Windows.


## Java Issues

### Java Patch Download

If the same procedure as Unix is followed to download Java, we end up with an
executable installer. The only silent install option is `/s` - there is no
way to specify the folder it ends up in. Also this does a Windows install
updating the registry etc. It's likely not to want to have two copies of the
same Java in different locations.

What I discovered after a little searching is 
[Doc ID 1439822.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=1439822.1)
which lists
every Java version, and each version links to a patch file. These files
have both the installer and zipped versions in them. These then are the
files to use - they can be unarchived right where we want them. 


### Java Permissions

If the old Java is deleted and the new one copied in, we end up without
the permissions which were applied to the old version. So we need to
reapply those. This is how I did that:

Save the permissions before:
```
icacls c:\psft\pt\jdk /save c:\psft\pt\jdk-acl
```
This writes the access control list (ACL) into the `jdk-acl` file. Once the
new version of Java is unzipped into place, we need to restore the permissions
as follows:

Save the permissions before:
```
icacls c:\psft\pt /restore c:\psft\pt\jdk-acl
```
This writes the permissions back to the directory. We don't need to specify
the `jdk` directory, that is already in the file.


### Java Version Issues

If the latest java version is applied, the SPBAT tool will complain
that it isn't a supported version. This is incorrect, and is because of the
extra number at the end of the version. Oracle 
[Doc ID 2875676.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2875676.1)
recommends
downgrading Java to a previous version to apply the patch, which is a bit
rubbish. In my case I downloaded JDK 11.0.15 (instead of 11.0.15.1.2
which was the latest)

I found this also applies to Unix, but I hadn't noticed as I applied
WebLogic before I upgraded Java, which may be another way around the
issue. Hopefully Oracle fix SPBAT before next quarter!


## WebLogic 
### Long Path Names

We apply the identical SPBAT patch on Windows as we 
[did on Linux](../patchingpeoplesoftwebserver/#extract-the-weblogic-server-stack-patch-bundle)
as it is a generic patch. 
Unzipping the SPBAT patch fails on Windows gives an error
like the following:

```console
Error unzipping 'C:\temp\p34084007_141100_Generic.zip' to 'C:\temp\34084007'!.
Method: System.IO.Compression.ZipFile, 
Exception: Exception calling "ExtractToFile" with "3" argument(s):
Could not find a part of the path
'C:\temp\34084007\WLS_SPB_14.1.1.0.220418\binary_patches\wls\generic
  \34011596\files\oracle.fmwconfig.common.wls.shared\14.1.1.0.0
  \fmwconfig.common.symbol\modules\com.oracle.cie.config-wls_8.8.0.0.jar
  \com\oracle\cie\domain\jms\impl\JMSSystemResourceServiceImpl.class'.
```

There are two issues here.

#### System unzip and long paths

The system unzip struggles with zip files containing long paths, like the 
SPBAT file. 
To fix this we install the powershell community module 
[pscx](https://github.com/Pscx/Pscx). Alternatively we could use
[7-zip](https://www.7-zip.org/)
to unzip the file.


#### Windows Operating System and long paths

We also need to edit the registry so that Windows allows the use of long paths
by setting the following `dword` to `1`:

```
HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled
```

## Tuxedo

I upgraded OPatch as I did for Linux, but when I ran the same command as in Linux
I got an error:

```console
C:\> \psft\pt\bea\tuxedo\OPatch\opatch apply 33735306.zip -silent -jdk C:\psft\pt\jdk -oh c:\psft\pt\bea\tuxedo

OPatch failed with error code = 255
```

Oracle support wasn't particularly helpful, suggesting to reinstall Java or 
OPatch. I discovered that the OPatch delivered for Windows includes a
version of Java 1.8. So if I remove the `-jdk` parameter, OPatch will
default to using that version and it will work. The problem with Oracles
patching tools is that they also need to be kept up to date, we don't
want vulnerable versions of Java on the operating system!


## Oracle Database Client

There is a separate client patch for Windows, which means it has a different
patch number. It is still linked from 
[Doc ID 2844795.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2844795.1#orcl19)
but you can't just switch the platform from Unix to Windows, you have to refer
back to the document to find the Windows and Unix patch numbers. The same as
for Linux though the Oracle client contains an outdated version of Java, which
will need to be updated afterwards.



## Conclusion

Windows has some particular challenges when applying patches compared to
Linux, but with the above workarounds we can keep up to date.