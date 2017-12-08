---
title: "Tech 17 - New Indexing Features"
date: 2017-12-07T14:52:38Z
image: images/Engine.jpg
---

For the last session before lunch I listened to [Richard Foote](https://richardfoote.wordpress.com/) talking about
new index features in 12.2. Richard is well known as the Oracle Indexing
expert, and so I was eager to hear what he had to say.

Object name length
--

Indexes can now have 128 character names,
where before they could only be 30 characters. I expect this will make
life easier where naming standards dictate long names for whatever reason.

Index Compression.
-

There are three possibilities here:

* **Basic compression** - This in effect deduplicates index data, so is no use for
a unique index that only has one field. Oracle can use it anyway and it will
increase the size of the index. It is possible to specify which columns are
compressed, but there may be some trial and error identifying the best ones
to compress.

* **LOW** - If advanced compression has been licensed, then LOW compression can
be used. This is the same as basic compression, but it is smarter as it can
decide whether or not to compress, and which columns should be compressed.

* **HIGH** - This uses a compression algorithm to compress the index data, so as
to shrink the index blocks. This leads to smaller indexes, but at the cost
of the time taken to compress the data, so isn't appropriate for indexes
on tables with high levels of DML.

It is possible to set the default compression type to HIGH:

    alter system set db_index_compression_inheritance=tablespace scope=both;
    alter tablespace <name> default index compress advanced high;

Richard didn't recommend this due to the overhead on high DML tables.

Index Monitoring
--

In 12.1, the index monitoring is limited. It is possible to switch on
monitoring, then look in `v$object_usage` to see if an index has been
used or not. Note that this doesn't record when statistics on an index
are used by the optimiser to measure cardinality etc. when the index
is not eventually used, so dropping an unused index may have performance
implications. Also not recorded is when an index is used for locking
of foreign keys.
 
In 12.2 there is a new view, `v$index_usage_info`, which shows the status,
of the index monitoring, and `dba_index_usage`, which shows which indexes
have been used. Index usage is sampled,
so is not exact, and some rarely used indexes may be missed altogether.

Columns of interest are:

* `total_execution_count`, which records the executions
sampled where the index was used.
* `total_rows_returned` - Total rows returned by the index.

Moving Tables
--

In 12.1, you can move a table online, but it invalidates indexes.
In 12.2, indexes are also rebuilt. It is a truly online process.


Clustering Factor
--

There is also a method for clustering tables. Previously one had to
create table as select with an order by clause to get the data in
the correct order. Now the clustering by option on the create or
alter table command allows the desired clustering to be specified
when the table is created. This has no impact on how the table
is managed, but when it is moved, the rows are created in the
desired order.

Partition conversion online
--

Online conversion to partition table (Requires partitioning option)
also converts indexes to local indexes. The global indexes remain global,
but are maintained so they remain valid. An update index clause can be
used if the default treatment of the indexes are not what was required.

Invalidation of SQL on index creation.
--

Previously, all SQL on a table
was invalidated if an index was created. Now it is possible to
defer invalidation. This is useful if an index is being created
to support new SQL which isn't in the shared pool. Existing SQL
will continue to use the same access path. If an index is dropped with
deferred invalidation, an SQL which doesn't use the index won't have to
be reparsed. Clearly if the index is used the SQL will have to be 
parsed again to generate a new plan which doesn't use the index.

Case Insensitive search.
--

In 12.1 it was possible to use case insensitive search using

    alter session set NLS_SORT=binary_ci;
    alter session set NLS_COMP=LINGUISTIC;

Clearly this affects every SQL run in the session, which
may not be what is required.

In 12.2 the collation can be set at a table/index column level.
Create the index normally once the collation is set, and it
follows what was specified. This uses functional indexes under
the covers. 

JSON
--
There are a number of enhancements to storing JSON in the database.
There is a constraint where the database can check that  the field
contains valid JSON. Dot notation can be used to select data, and
this can be used in the where clause. Indexes can be created on 
JSON data structures using dot notation. This uses an index on a
virtual column under the covers. This can be used as a constraint
on JSON data, e.g. to ensure a JSON field is a number.

Search index on JSON fields is a big new thing in 12.2. Previously
one would have used a text index.


