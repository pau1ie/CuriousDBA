---
title: "Tech 17 -The Answer to the Ultimate Question of SQL, Performance Tuning and Everything"
date: 2017-12-07T14:41:38Z
tags: ["Tech17","UKOUG","Oracle","Performance","Optimizer","eDB360"]
image: images/Engine.jpg
---

This is second session I attended in Tech 17. It was 
presented by Martin Bach and [David Kurtz](http://blog.psftdba.com). The answer is [eDB360](https://carlos-sierra.net/2014/07/27/edb360/).
The session was in two
parts with a presentation by Martin, then a demo with David.

Martin discussed the problems of doing a database health check, particularly
for a third party company who may not actually have access to the database.
It needs a standard approach, which is consistent across databases, and
repeatable. Ideally performance data in AWR should be persisted for just over
a year so that the performance of annual business cycles can be compared
e.g., year end for financial systems.

eDB360 was created by [Carlos Sierra](https://carlos-sierra.net/) and others.
The suite is under development, so they will accept patches from others to add
any other information gathering scripts that makes sense given the scope of the tool.
Ideally the database should have tuning and diagnostic packs licensed.
If they are not the tool can still be run, but there is much less data
available for the tool to collect. It doesn't install anything to the database
There is plenty of documentation available, and it can be downloaded from Oracle
support, [Carlos' blog](https://carlos-sierra.net/ "Carlos Sierra") or [github](https://github.com/carlos-sierra/edb360/ "eDB360 on Github").

To minimise the load on the database, all the information is gathered serially
in a single session. The information is gathered using SQL statements
which can be viewed, and it is all in clear text or HTML. You need to pass
a parameter of **T**, **D** or **N** to indicate whether Tuning and diagnostic, just
diagnostic or neither pack is licensed. There is a script to run it on
all databases on a server, but Martin recommended just  to run it on the
database you are interested in.

The tool will then run in the database for several hours in its single
session. It will abort if it runs for more than 24 hours. It creates
a zip file containing around a thousand files. The zip file unzips all
these into the current directory, so it is important to ensure that this
is a clean directory to start with. Open 00001_*.html in a browser to view
the data.

David Kurtz had created a report earlier and opened it to show us. There
are around 1000 links, so it can be quite daunting, but just concentrate
on what you are interested in. Some of the pages contain charts which are
generated using the Google APIs. These are downloaded to the browser to
generate the chart from the inline data, so an internet connection is
required to display these. It is possible to drill into them, which
is more useful than an AWR report which just shows averages. For example
you can see how the performance of the IO system changes with time.

David demonstrated a few interesting pages, e.g. top SQL queries, redundant
indexes, system time model (This had a zoomable graph), average active sessions
Top plan hash values, analysis of wait events, top SQL by force Matching
signature. ASH reports are generated for time periods that the tool feels are
interesting. It will generate a couple of reports in adjacent time periods.
It won't generate reports on time periods of greater than an hour as the
authors feel the averaging loses interesting data.

Martin pointed to the usefulness of the histogram data in the top 24 wait
events. This clearly shows the IO subsystem latency increasing
as the load increased.

It generates SQLD360 reports for the top 16 SQLs which means that this
tool is included with eDB360. It isn't installed into the database, so
it doesn't export stats or data like SQLT does. That tool is good for
generating test cases.

It works on all versions of Oracle 10.2 and greater.


