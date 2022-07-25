---
title: "SessionHistory"
date: 2017-09-18T10:14:21+01:00
image: images/History.jpg
tags: ["Performance","Oracle","DBA"]
---

Here is another note to myself about how to look at what sessions were doing over time.

The situation was that we saw a lot of blocking sessions during a performance
test. The developer wants information about the blocking sessions. To do this
we want historical information from v$session, which helpfully tells us the
blocking session, and also the SQL being run. This can be sampled every so often
using a simple script, but Oracle have already thought about this and created
v$active_session_history which you can use if you have licensed the Diagnostics pack.

This dynamic performance view is saved to DBA_HIST_ACTIVE_SESS_HISTORY periodically.
So, to see which session is blocking another we can do the following:

{{< highlight sql >}}
select blocking_session, blocking_session_serial#
from dba_hist_active_sess_history where sample_time > trunc(sysdate)+14/24
{{< /highlight >}}

We can look at the entries for the blocking session using something like this:

{{< highlight sql >}}
select *
from dba_hist_active_sess_history h
where sample_time > trunc(sysdate)+8/24
and (session_id, session_serial#, sample_id) in
 (select blocking_session, blocking_session_serial#, sample_id
  from dba_hist_active_sess_history
  where sample_time > trunc(sysdate)+8/24
 )
;
{{< /highlight >}}

And we can even see what the session is running using the following:

{{< highlight sql >}}
select sql_id, count(*),
  (select username from dba_users u where u.user_id = h.user_id),
  (select sql_text from v$sqltext s where s.sql_id = h.sql_id and piece=0)
 from dba_hist_active_sess_history h
 where sample_time > trunc(sysdate)+14/24
  and (session_id, session_serial#, sample_id) in 
   (select blocking_session, blocking_session_serial#, sample_id
    from dba_hist_active_sess_history
    where sample_time > trunc(sysdate)+14/24
   )
 group by sql_id, user_id
 order by count(*) desc;
{{< /highlight >}}

I notice [Arup Nanda did an Oracle Magazine article on this in January 2013](https://asktom.oracle.com/Misc/oramag/beginning-performance-tuning-active-session-history.html).
