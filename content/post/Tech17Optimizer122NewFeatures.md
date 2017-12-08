---
title: "Tech 17 - Oracle Database 12c Release 2 - What is new in the Oracle Optimiser"
date: 2017-12-07T14:55:12Z
tags: ["Tech17","UKOUG","Oracle","Performance","Optimizer"]
image: images/Engine.jpg
---

The first session after lunch was Oracle Database 12c Release 2 - What is new
in the Oracle Optimiser. It was taken by [Nigel Baylis](https://blogs.oracle.com/optimizer/oracle-optimizer-2) who is the product
manager for the oracle optimiser.

I was impressed by Nigel's honesty when he talked about what worked, what
didn't and how things are being changed to correct the mistakes of the past.
This is the main reason to come to user groups of course. It made me feel
like Oracle are willing to accept where things haven't worked out as they
hoped, and take action to correct it.

Apaptive Features
--
He started by talking about 12.1. He said that adaptive encompasses
two features, which had different levels of success.

The successful feature is *adaptive plans at execution time*, where the
optimiser changes the plan e.g. between hash joins (Better for larger data sets)
and nested loops (Better for smaller data sets) at run time depending
on the amount of data returned at that point in the plan. This worked
very well.

What was less successful in certain systems are *adaptive statistics* and
*dynamic sampling*. This caused plan instability which was not popular.

So in 12.2, by default *adaptive plans* are switched on, and *dynamic 
stats* are switched off using the following parameters:

    optimizer_adaptive_plans = True
    optimizer_adaptive_stats = False

*Adaptive stats* are more suited to longer running queries, where a slightly
longer parse time caused by dynamic sampling is worth it compared to the
time the query will take to run. Also for ad-hoc queries, especially with
literals which will have to be parsed anyway.

SQL  PLan Directives
--
This is another feature which was not as successful as hoped.
This allows the database to learn from poor performance, and directives can be copied
from one database to another. However, often this feature works in test,
and the query performs well. Since administrators aren't aware that
good performance is due to an automatically created SQL Plan directive, code is
released to production and runs slowly.

So now SQL plan directives are automatically generated and stored, but
not used.

Statistics
--

For similar reason column group statistics will also default to being switched off in 12cR2. 

In total, this means the feel of 12cR2 is much more like 11g. Some proven
enhancements are still on by default, but the ones which have caused problems
elsewhere are switched off by default.

[Expression tracking](https://blogs.oracle.com/optimizer/expression-tracking) has been added.

Synopsis data can become large in the sysaux tablespace. 12.2 improves this by use of the [hyperloglog](https://en.wikipedia.org/wiki/HyperLogLog)
algorithm.

There is a preference override parameter which allows the gather stats
command to be overridden. This is particularly useful where a sample
size was specified, and `DBMS_STATS.AUTO_SAMPLE_SIZE` would be preferred.
The change can be made without the risk of changing the scripts.

Approximate Query Processing
--

There have been improvements in approximate query processing. This is
particularly useful when creating visualisations or graphs where being
one or two pixels out doesn't matter too much. Apparrently they will
be 97% accurate 96% of the time.

An example is approx_count_distinct. The entire table is still scanned. The
performance benefit is in the memory used for sorting - e.g. spill to disc
should not be necessary. This can be used with materialised views.
The higher the number of distinct values, the more benefit there is to
using this approach.

Query Optimisation
--

There are changes to query optimisation. These tend not to be documented
and Nigel was of the opinion that the DBA shouldn't need to care. However,
he highlighted the **OR** expansion where a query is rewritten from using **OR**
to using a **UNION ALL**. The union branches can be run in parallel to allow
the query to complete faster.

Statistics Advisor
--

Lastly he highlighted the optimizer stats advisor. It searches for problems
with statistics and generates scripts to fix them. 
There is not much overhead to running this. It runs as part of stats gathering
and stores the results in the data dictionary.
