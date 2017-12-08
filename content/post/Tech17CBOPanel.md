---
title: "Tech17 - Cost Based Optimizer - The Panel session"
date: 2017-12-07T14:31:18Z
tags: ["Tech17","UKOUG","Oracle","Performance","Optimizer"]
image: images/Engine.jpg
---

On Wednesday 5th December, I attended [Tech 17](http://www.tech17.ukoug.org/).
While it is a pain to organise getting there and back, I found in previous
years that it was useful for me to go and find out what is happening in the
industry.

Cost Based Optimisation - The Panel Session
=======

Panelists:

* [Jonathan Lewis](https://jonathanlewis.wordpress.com/)
* [Christian Antognini](https://antognini.ch/)
* [Richard Foote](https://richardfoote.wordpress.com/ "Richard Foote Consulting")
* [Nigel Bayliss](https://blogs.oracle.com/optimizer/oracle-optimizer-2 "Oracle")
* [Maria Colgan](https://sqlmaria.com/)

Slow Start
-

This took questions from the internet and the audience, so it is a bit
variable. The question has to match something the panel can talk about.
The first questions about the Autonomous database was disappointing
in this regard as very little information has been released about it,
and the Oracle employees had to stick to  the company line, so couldn't
speculate.


Favourite 12.2 feature.
-

Nigel Bayliss said he liked the new cost based **OR** transformation, where
an **OR** can be replaced by a **UNION ALL**, and the statements in the union
can be run in parallel. I leaned more about this later.

Another panelist mentioned they liked the fact that you can now 
reorganise a table online, including indexes. This means that the
cluster factor of tables can be changed to match the index for important
queries, making index use more likely for these queries.

Clustering Factor
-

There was some discussion of clustering factor after another question on
tuning. This is a number calculated
for an index when gathering statistics which the optimizer uses to 
judge whether it is worth using an index or a full table scan. It counts
the number of blocks that would need to be visited in the table to read
it in the order of the index. So the best is the number of blocks in the
table, where the table is ordered exactly as the index. The worst is the
number of rows in the table, where every row is in another block.

The problem occurs when the rows are actually clustered together, but
subsequent rows are in different blocks (e.g. to read rows in order
you need to skip between two blocks. Apparently in some circumstances
a parallel load of data can cause this to happen. This means that
the data is clustered, but the optimiser thinks it isn't.

The solution is to set the 
`table_cache_blocks`
preference on the stats
gathering process to something higher. Jonathan Lewis preferred 8,
and Richard Foote prefers 42. The idea is that if the row is in a
block that has been read in the last few blocks, it is not counted
as another read, as the block will have been cached. This makes it
more likely that an index will be selected.

We should be cautious of this though, as other parameters such as
`optimizer_index_cost_adj`
and
`optimizer_index_caching`
may already
have been set to solve this problem. A query on a poorly clustered
table with all these parameters set may end up using the index
when it is not appropriate, therefore generate more IO than a 
table scan, so as with many changes it should be done cautiously.

Index Monitoring
-
Also mentioned was that 12.2 now monitors indexes by default
making it more easy to understand whether it is being used.

Histograms
-
A questioned asked Maria Colgan and Jonathan Lewis about their
differing view of histograms. They both agree they are useful in some
circumstances and not others. Maria Colgan takes the view that
you should allow the optimiser to create them and remove the
ones that are not helpful. It is difficult to know when a
histogram will be helpful, so this allows you to get the benefit.
Jonathan Lewis takes the view that histograms can cause problems
with plan instability, and that they should only be used when
they are needed to avoid these problems. However, he seemed to
be of the opinion that the *Top N*  histograms don't have this sort
of problem, though the *height balanced* and *hybrid* histograms still
need to be treated with caution.

Performance
-
There was some discussion on proactive SQL monitoring. To see where
the optimiser is going wrong, you can look at the different between
the expected and actual rows, and at the first point in the plan
(by execution) where these numbers are different, look into
why the optimiser generated in incorrect estimate.

SQL Plan directives were automatically generated and used in 12.1
for queries with similar predicates, which caused problems in
some cases, particularly for OLTP and mixed workloads. This
is turned off by default in 12.2 with the `optimizer_dynamic_statistics`
parameter.


