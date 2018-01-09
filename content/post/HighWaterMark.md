---
title: "High Water Mark"
date: 2018-01-09T15:23:14Z
tags: ["Oracle","DBA","Performance","Storage","latch","HW Contention"]
image: "HWM/HighWaterMark.jpg"
---

We are running a data conversion and got a wait event I don't normally see:
![Oracle enterprise manager showing configuration waits](../../HWM/Configuration.png)
The brown is identified as *Configuration*. It I drill down, I can see more detail.
![Oracle enterprise manager showing High Watermark waits](../../HWM/HWContention.png)
Here light purple is *HW Contention* (i.e. High Watermark Contention). Darker
purple is *Write Complete waits*, and yellow is *buffer busy waits*.

We have deferred segment creation switched on for the database. This means that the
segment needs to be created before data can be written. The high watermark can only
be moved by one process at a time.

It seemed obvious to me that all I need to do is allocate the extents, and
then everything will be faster. I
[asked on the Oracle-l mailing list](https://www.freelists.org/post/oracle-l/Expanding-a-table),
and [Harel told me how](https://www.freelists.org/post/oracle-l/Expanding-a-table,1).  
I took a list of tables, and generated the SQL necessary to create the extents:

{{< highlight sql >}}
set echo on
drop table mine.tables purge;
create table mine.tables (name varchar2(4000));
insert into mine.tables (name) values ('TABLE1');
insert into mine.tables (name) values ('TABLE2');
insert into mine.tables (name) values ('TABLE3');
insert into mine.tables (name) values ('TABLE4');
insert into mine.tables (name) values ('TABLE5');
insert into mine.tables (name) values ('TABLE6');
insert into mine.tables (name) values ('TABLE7');
insert into mine.tables (name) values ('TABLE8');
set lines 30000
set long 30000
set longc 30000
set pages 50000
set trimspool on
set hea off
set echo off
set feed off
spool allocates.sql
select
    'alter table owner.' ||
    table_name ||
    ' allocate extent ( size ' ||
    trunc( 1 + blocks /128 ) ||
    'm);'
  from dba_tables dt,
    mine.tables pt
  where pt.name = dt.table_name
  order by blocks;

select
    'alter index ' ||
    di.owner ||
    '.' ||
    index_name ||
    ' allocate extent ( size ' ||
    trunc( 1 + ds.bytes/1048576 ) ||
    'm);'
  from
    dba_tables dt,
    mine.tables pt,
    dba_indexes di,
    dba_segments ds
  where
    pt.name = dt.table_name and
    di.table_name = dt.table_name and
    ds.segment_name = di.index_name and
    dt.owner = ds.owner and
    ds.owner = di.owner and
    ds.segment_type = 'INDEX'
  order by ds.bytes;
spool off;
exit;
{{< /highlight >}}

This creates a nice script to allocate extents:

{{< highlight sql >}}
alter table user.TABLE1 allocate extent ( size 1219m);
alter table user.TABLE2 allocate extent ( size 1219m);
alter table user.TABLE3 allocate extent ( size 1260m);
alter table user.TABLE4 allocate extent ( size 1275m);
alter table user.TABLE5 allocate extent ( size 1758m);
alter table user.TABLE6 allocate extent ( size 1889m);
alter table user.TABLE7 allocate extent ( size 1890m);
alter table user.TABLE8 allocate extent ( size 2068m);
{{< /highlight >}}

And the same for the indexes

{{< highlight sql >}}
alter index user.INDEX1 allocate extent ( size 224m);
alter index user.INDEX2 allocate extent ( size 233m);
alter index user.INDEX3 allocate extent ( size 233m);
alter index user.INDEX4 allocate extent ( size 249m);
alter index user.INDEX5 allocate extent ( size 273m);
alter index user.INDEX6 allocate extent ( size 277m);
alter index user.INDEX7 allocate extent ( size 277m);
alter index user.INDEX8 allocate extent ( size 338m);
alter index user.INDEX9 allocate extent ( size 587m);
{{< /highlight >}}

Meanwhile, [Tim Gorman replied urging me to test](https://www.freelists.org/post/oracle-l/Expanding-a-table,7)
and suggesting it might not be the panacea I was hoping for. As he said, it
is pretty easy to create a table and put rows into it in parallel. I created some scripts. One to create a table: *01create.sql*

{{< highlight sql >}}
--drop table mine.instest purge;
--create table mine.instest (id number, filler varchar2(4000));
--alter table mine.instest allocate extent (size 1280m);
--alter table mine.instest storage(next 10m);
delete from mine.instest;
exit
{{< /highlight >}}

A script to insert some rows into it: *02ins.sql*

{{< highlight sql >}}
begin
for x in 1..10000
loop
  insert into mine.instest
    select dbms_random.random, dbms_random.string('A',2000) from dual;
end loop;
commit;
end;
/
exit
{{< /highlight >}}


And a script  to run them:

{{< highlight bash >}}
echo Enter pass
read pass
sqlplus mine/$pass @01create

for i in $( seq 1 48 )
do
  nohup sqlplus mine/$pass @02ins > test$i.log 2>&1 &
done
{{< /highlight >}}

Now I have a reproducable test case. It would probably be better if I
created a table with an index and a LOB and put some data in that, it wouldn't be too
difficult to do that, and I suspect these problems are worse for LOBs, or at least
there are more segments which will need to be grown at the same time, especially
at the start.

For the first test I ran the *01create* script as follows:

{{< highlight sql >}}
drop table mine.instest purge;
create table mine.instest (id number, filler varchar2(4000));
exit
{{< /highlight >}}

I found I had to run 48 in parallel before I started to get significant HW Contention latch waits. The HW Contention waits I had
in the conversion process were casused by running 53 in parallel, though that had more waits, presumably
becuase those tables had indexes and some had lobs. However, I didn't test for that because I just want to
see whether allocating space helps. The next run was:

{{< highlight sql >}}
drop table mine.instest purge;
create table mine.instest (id number, filler varchar2(4000));
alter table mine.instest allocate extent (size 1280m);
alter table mine.instest storage(next 10m);
exit
{{< /highlight >}}

The next extent shouldn't matter, as I hope I will be allocating as much as
the table will need in the allocate extent command. Running it showed no
significant difference in the HW Contention waits. This is rather disappointing.

Another test I did was to reuse the table from the last test, but delete
the rows from it. This would preserve  the high water mark. The *01create.sql*
script became:

{{< highlight sql >}}
delete from mine.instest;
exit
{{< /highlight >}}

This is the only thing I found that would make a difference.

I could have tried changing the tablespace, it currently uses Automatic Segment
Space Management (ASSM), but I didn't want to change the storage on the tables
to support the conversion when it might not be as good for OLTP workloads.

It is slightly irritating for me not to have a solution to this (Just set X=Y
and everything will be better!), but I have a better understanding from
thinking about the situation, testing, reading, and asking on Oracle-l.

Here are things I have learned:

* When Oracle moves the high watermark on a table, only on session can do this at a time, so the HW Contention latch is used to single stream this.
* Blocks have to be formatted as part of this process. This seems to be a relatively expensive thing to do.
* There are ways to move the high water mark "down" to reduce the space taken by a table.
* There is no way to move the high water mark "up" to increase the space taken by a table, apart from inserting rows.
* You can allocate an extent using an alter table command.
* You can increase the next extent size on a table.
* Neither of these have any impact on the HW Contention waits.
* ASSM doesn't give many ways of tuning underlying storage. Other storage options do, but that would involve changing the tablespace and a lot more testing.
* There is value in testing assumptions. It can be quite easy!

In addition to the discussion on the Oracle-l mailing list referred to above,
I found
[this Ask TOM question about High watermark on a table](https://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:492636200346818072)
and the article it refers to
[Resolving HW enqueue contention by Riyaj](https://orainternals.wordpress.com/2008/05/16/resolving-hw-enqueue-contention/)
were also useful. There is a lot more to investigate here.
