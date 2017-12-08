---
title: "Tech 17 - Oracle 18c The Next Update"
date: 2017-12-07T17:40:54Z
image: images/Engine.jpg
tags: ["Tech17","UKOUG","Oracle","Performance","Upgrades","Critical Patch"]
---

Dominic Giles drew our attention to the safe harbour statement. Anything can change
between now and when the new version is released.

There has been some confusion about the announcements regarding autonomous
database. This is a *service* on top of oracle 18c. Oracle 18c is a product.

Release Schedule
--
Oracle will change its release process to release annually. Hopefully this
will lead to more stable releases, as there won't be a rush to get the new
features into the current version of the database. Changes will be
incremental, but will happen every year. There will be quarterly release
updates. These will go back to using a number rather  than a date format,
These will include all fixes, not just security fixes, but performance
changes will be switched off by default.

Between the quarterly Release Updates (RU) will come Release Update
Revisions (RUR). There are expected to be 1-2 per quarter. There will
also be individual patches released.

Oracle will choose every 2-3 years an extended support release. This
is the release to choose to save time updating annually. Oracle expect
that most users will choose to upgrade annually though.

Incremental Improvements
--
18c starts as Oracle intend to go on, delivering a lot of incremental
improvements. He went through a list which I include below. Many are to
additional cost options, so a license is required to take advantage of them.

- RAC on Exadata has performance improvements in the maintenance
of undo. As I understand it they only copy the changes rather than the
entire block.
- There are a number of improvements to the in memory
option for OLTP workloads.
 + In memory dynamic scans.
 + In memory optimised
arithmetic (The Oracle number format is changed to a binary format so
the CPU vector unit can be used for calculations). 
- Per PDB switchover. The idea is to be able to run a pluggable database
on each of two servers, half open, and half as standbys. 
- Faster upgrades (About 5% faster)
- Snapshot carousel - snapshot of PDBs. I think this is similar to the
SAN functionality of creating a snapshot of environments at points
in time.
- Improved availability. Zero downtime Grid Infrastructure patching.
- Sharded RAC
- Sharding improvements.
- Improved Security
 + Per PDB Key stores
 + Passwordless schema creation.
 + Active Directory authentication, without the need for Oracle Directory
Services.
- Data Warehouse improvements.
- Alter table merge partition online e.g. to merge hourly partitions to daily ones.
- Automatic propagation of nologged data to standby 
- Polymorphic tables - can encapsulate sophisticated algorithms
- part of the SQL ANSI 2016 standard.
- Improved JSON support
- Property graph improvements
- Rolling Patchets for OJVM in the database (Not till 18.2)
- Private temporary tables (i.e. per session).
- Official docker support (See Oracle Support document id 2216342.1)

DB REST API
--

All administrative actions can now be performed using a REST API
which should be useful for cloud services.

- Includes management and monitoring
- Same API for cloud and on premise
- API for all database life cycle operations
- Shipped with ORDS and updated quarterly
- Every feature will have a REST API.


Installation 
--

Gold image as a service (i.e. Oracle Home)
On demand image creation including application of

 - Release Updates
 - Release Update Revisions
 - One off patches

Can be deployed as a:

- zip file
- docker image
- Virtual Machine
- RPM file, but for base un-patched home only.

Oracle XE
--
XE is also coming into line with the yearly updates. Allowed is:
12G User storage
2G SGA
It contains all Enterprise Edition features
It will install under both Windows and Linux. There is the possibility
of an Apple Mac version, though given VMs etc this may not be needed.
