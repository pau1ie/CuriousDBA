---
title: "More Lessons About IO"
date: 2020-08-19T15:20:49+01:00
tags: ["DBA","Oracle","Performance","Storage","Tuning"]
---

I expect the question after the last I/O article is: How did we make our server go faster?

## Things That Did Work
Looking at the AWR report, I could see the main problem was that
async I/O was slow. We set 
```ini
filesystemio_optoins='SETALL'
```
in the spfile, and this helped a lot. As I understand it, 
this enables asynchronous I/O, which means
that rather than waiting for confirmation after writing, the database keeps
on sending data to the SAN. This sped up the I/O by a factor of about 3.

The sysadmins also purchased a faster card, which gave a slight bump in performance.

## Things That Didn't Work

Part of the process is forming theories, testing them and rejecting them
if they turn out not to be valid. This can take time, but there is no way
of knowing whether the theory is correct or not until it has been tried.

While it may seem like a waste of time to run tests like this
which don't end up showing a way to better performance, this does help our
understanding of the problem, and stops us incorporating unnecessary complexity
to our build.

### Multiple File Systems

We tried splitting the database across two file systems. This made no difference
at all. 

### Use Fibre Channel Rather Than iSCSI

We also tried running the test on another server which had an FCAL
card rather than an iSCSI. This was no faster than the iSCSI card when compared
with that servers internal disc, so we could rule out that iSCSI itself
was the problem. 

## The Way Forward.
The two changes above gave consistently better results in the 
SLOB tests, which was encouraging.
However it would seem wise to test with a realistic workload. The testing team
was rather busy. My management asked me if we really needed to test given the
very positive results from SLOB. This is a tricky thing to manage, because I
have to advise my management to do things they would prefer not to, i.e. hold
up the project.
 
### Managing the Manager
I feel here it is important  to understand what my job is and what my managers
job is. We have someone called a Service Manager whose job it is to maintain the
service. It is their job to make this type of decision. It is my job to provide
the information they can use to make that decision, including my recommendation.
 
I explained my recommendation was we test with a more representative workload, because
there may be additional issues SLOB didn't flag up. Management accepted my
recommendation, but equally I would have been happy to go against my own advice,
so long as management accepted the risk of our encountering a performance issue
that was difficult to fix during a business critical period. 
 
### The Results
The results were positive. Most things were much quicker. However, the testers
noticed the number of failures had increased. These were all timeouts.
Clearly something was still slow. 

Looking at the ASH report I noticed that
direct path reads were slower than before. A direct path read is where a session
bypasses the cache. This is during a table scan, or (by default) when accessing
a LOB. Drilling down, I could see that the objects causing the issues were LOBs.
One holds html fragments
that are used to build the web page. These would clearly benefit from being
cached. Also the way the application works means that a document upload is done
to a LOB in one table, then it is moved to another table. The intermediate table
would also benefit from caching.

Some colleagues asked why we don't cache all the LOBs. The database server has
a lot of RAM, and we are not using much of it. However the total size of the LOB
segments far exceeds the RAM, and it is pointless caching things that aren't
going to be used for a while, e.g. the final location of the uploaded documents.

## Conclusion and Next Steps

We need to cache the LOBs and run another test. I expect it will all work fine,
and we will be able to move the new hardware into production. The lesson learned
again from the experience is that we shouldn't just assume things will be OK.
We should test the assumption. Testing
is annoying. It takes time. But it is less annoying and time consuming
than dealing with production issues!