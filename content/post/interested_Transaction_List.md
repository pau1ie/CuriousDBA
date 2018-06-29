---
title: "Interested_Transaction_List"
date: 2017-11-13T11:23:08Z
draft: true
---

I look at interested transaction lists from time to time. 

We have an issue in our peoplesoft system where we get locks on the PSLOCK or PSVERSION table.
There have been suggestions that to reduce this we increase the INITTRANS parameter and rebuild
the table by exporting and importing it.

I don't think this will help. Here is an explaination as to why.

The interested transaction list (ITL) is how Oracle records a transaction has a lock on a row
or rows within a block. The maximum possible size of the ITL is the number of transactions that
can hold locks on rows in a table at the same time. Since only one transaction can aquire a
lock at a time on a row, this can't be more than the number of rows in the block and is likely
to be much fewer.

The INITTRANS parameter is the initial size of the interested transaction list in each block
of the table. It defaults to 1, but can grow if there is space in the block and MAXTRANS is
the maximum, which is 255 (Maxtrans is ignored in later Oracle versions). There is likely
to be space in the block, as PCTUSED defaults to 10, so there should be some free space in
blocks. A block is 8K, so 10% of 8K is ~800 bytes. Each ITL row takes 24 bytes, so this is
enough space for another 33 entries.

We can dump out the blocks in the PSLOCK table and take a look. Lets find out which block
the problem row is in.



If there is not space in the block, and a transaction wants to lock a row in the block,
you get an extra wait on ITL waits. 

The only way to tune this is for the lock to be held for a shorter period of time.

See http://arup.blogspot.co.uk/2011/01/more-on-interested-transaction-lists.html

{{<highlight console>}}
  1* select statistic_name, value
  2  from v$segment_statistics
  3  where object_name = 'PSLOCK' order by value
SQL> /

STATISTIC_NAME                                                        VALUE
---------------------------------------------------------------- ----------
gc cr blocks received                                                     0
IM repopulate CUs                                                         0
IM prepopulate CUs                                                        0
IM populate CUs                                                           0
IM scans                                                                  0
segment scans                                                             0
space allocated                                                           0
space used                                                                0
gc buffer busy                                                            0
physical reads direct                                                     0
physical writes direct                                                    0
optimized physical reads                                                  0
optimized physical writes                                                 0
IM repopulate (trickle) CUs                                               0
gc current blocks received                                                0
ITL waits                                                                 0
IM non local db block changes                                             0
physical read requests                                                    1
physical reads                                                            1
buffer busy waits                                                        90
row lock waits                                                          407
physical write requests                                                 453
physical writes                                                         453
logical reads                                                        141888
db block changes                                                     149680
{{</highlight>}}

The same is true for PSVERSION, and for the indexes which only have waits on reads.

Having said that, I don't think the proposed change will do any harm. It might make the table longer as we are allocating more space for the ITL (Actually it will make it shorter as there is so much free space - there must have been a lot of rows removed from this table), but apart from that it will have no effect.

I can only assume those who see performance improvements were suffering ITL waits, or the business process has moved to a stage where they get fewer locks and they will return.

