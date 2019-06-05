---
title: "Patching Java for PeopleSoft"
date: 2019-05-28T13:44:07+01:00
tags: ["Oracle","Critical Patch","Automation","Java","PeopleSoft","Upgrades","VM"]
---

## Security Updates Not Included!

Oracle's Deployment Package DPK is supposed to deliver all the software required and at the correct versions.
However, it appears this isn't true. PeopleSoft itself is at the current version in the DPK,
but all the software it depends on only has the previous quarters security updates applied. This often
leaves us vulnerable to some quite serious security flaws.

The solution is to apply the patches after applying the DPK. It is annoying that we have to
do this, because the sales documentation at the time suggested everything would be at the correct 
version with the latest patches.


## Applying the Java Patch

To patch Java is fairly simple, just unzip the archive and repoint the application at the new 
Java home. On Linux I set the path and Weblogic configuration to
point to a latest symbolic link which gets updated whenever Java is updated. Windows doesn't
have symbolic links in the same way Linux does, so I have to
update the configuration and the path to point to the latest Java version.


## Java Runtime

For some reason Oracle supply two copies of the Java Runtime Environment (JRE) with the PeopleTools
installation. One in the Java Development Kit (JDK) itself which is instaled at the same level as all
the other homes (e.g. $PS_HOME/..). The other is actually within
the PeopleSoft home. This also needs to be updated. The way to do it is to
copy the jre from the jdk into the ps_home. This is documented in Oracle Support
[document ID 2323346.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2323346.1) for Windows
and [document ID 2355569.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2355569.1) for Linux.


## Fonts

When I did this, reports started displaying squares instead of Unicode characters. Since
a significant number of our students have Unicode characters in their names, this is 
quite a problem, and potentially quite rude to them. I discovered that the fonts directory delivered
has some extra fonts in it, in both Linux and Windows. Once the jre directory is updated
these fonts need to be preserved.

## Conclusion

Oracle seem to have two problems

  1. They don't deliver fully patched versions of their software,
  1. They customise the jre home.

These are fairly easy to fix once they are understood. Our automated patching process ensures
that once this is scripted it will always be run in an identical way.
