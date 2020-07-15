---
title: "Lessons About IO"
date: 2020-07-13T15:20:49+01:00
tags: ["DBA","Oracle","Performance","Storage","Tuning"]
---

# How Does a Database Use IO?
We have a shiny new database server. We ran some tests on it,
and found it was about the same speed as the old one. Worryingly
the IO system seemed to be struggling to keep up with the load.

In case you don't know, IO is short for Input/Output, and is
used to describe the use of a hard disc, or similar storage.
We all know that hard discs are slow compared to memory, but
for a database to work well, the IO system needs to be able
to keep up with the work the database is doing. Otherwise the
database will eventually appear to lock up.

In these situations it is useful to have a rough idea of how
the database works. 

## Updating data
If I want to update some data (Ignoring locking, latching, SQL parsing, and everything that isn't IO which I am not interested in for this exercise)
* The session checks if the block containing the data I want to update is in the buffer cache. If not, it reads it in.
* Then I can make the change to the data.
* The original data is written to the undo tablespace (in the buffer cache) so the database can roll the change back.
* This change to undo is written to the redo logs (The head of a ring buffer in memory)
* The new data is written to the table (in the buffer cache) and that block is marked as dirty.
* This change is written to the redo logs (The buffer in memory)

We have changed a lot of information,
but so far nothing has been written to disc. In fact, nothing has been
read either if we were lucky enough to discover our block was already
in the buffer cache. This is how the database can be so fast! What happens
next is the important bit.
* commit

Now the database promises when it returns from the commit, it will
preserve the transaction. The D out of ACID. All the database has to do is:
* Ask the log writer to make sure the part of the log buffer containing our transaction is written to disc.
* Let us know

Notice we haven't actually written the data to disc yet. This is OK because if the database
crashes, we have the redo log on disc which will tell the database what it needs to do.

## What can go wrong?
The above is all very clever, but what happens if our storage is too slow.
### What if my block isn't in the buffer cache
The block has to be read in from the disc. Oracle records the time waiting to do this as _db_file_sequential_read_.
There are a number of other wait events which this could be. See the _Database Reference_ manual.
### What if the buffer cache is full?
If the buffer cache is full, we can find a block which isn't dirty and nobody has used for a while, and overwrite it.
### What if there are no blocks we can overwrite?
Our session will ask the database writer to write some dirty blocks to disc. Then they won't be dirty any more
and we can overwrite them. We will have to wait while the database writer does this. Oracle records this as a
_free buffer wait_. There are a couple of places this can happen. When the block is read in from disc, and also
when the undo block is updated. 
Have a look at the documentation for the _free buffer wait_ event in the _Database Reference_ for more detail.
### What if the redo buffer is full
The redo buffer is a ring buffer. Conceptually a ring
buffer has a head pointer and a tail pointer chasing each 
other round the ring. If the head bumps into the tail, the buffer is full.
If the tail bumps into the head, the buffer is empty.

Our session asks the log writer to write the log buffer to disc, then it waits till there is some space.
Oracle records this wait as _log buffer space_.
### What about when I commit
The session will ask the log writer to write the data in the log buffer
to the disc. Then it will wait till that is done. Oracle records this
wait as _log file sync_.

## Do bigger buffers help?
When I make Elderflower Cordial, or Sloe Gin, I need to filter the drink once it has infused to
get the solid bits out. I put a funnel on top of a bottle, then I add a piece of muslin to filter
it, then I pour in the liquid. Eventually the funnel fills up. The top part of a funnel is a buffer
holding the liquid while it waits to flow through. Making that part bigger won't make things quicker, I 
need to increase the flow through the muslin.

It is like this with buffers. If the buffer is full because the disc can't keep up, making it
bigger won't really  help, it will just fill up again, because they aren't being drained fast
enough. This doesn't mean that buffers should never be made bigger, just in this case it won't
help. What is required is faster IO, and in particular faster writes if these buffers are filling
up.




