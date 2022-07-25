---
title: "PeoplesoftPerformance"
date: 2017-08-16T17:37:19+01:00
tags: ["PeopleSoft","Performance"]
image: images/cars-engine.jpg
---

# PeopleSoft Performance

Occasionally I have to diagnose a performance issue in PeopleSoft. This is a reminder to myself of the tools available and what I can check.

## PeopleSoft Ping

This is a really useful tool because it gives real performance indications as to how the system as a whole is performing from the desktop to the database.
The user can see the time taken at the browser, web server, application server and the database server for a simple query. When a slow response time is seen 
it is important to drill down to see what cause it. It could just be Windows downloading updates for example. PeopleSoft ping data is stored in the database
used by the application in the table sysadm.ps_ptp_tst_cases. I used the following columns
(See [Dave Kurtz' website](https://www2.go-faster.co.uk/peopletools/ptp_tst_cases.htm) for the rest).

Column           | Notes
-----------------|-----------------------------------------------------------------------------
PTP_TEST_CASE_ID | The test case identifier entered when the ping test was started.
PTP_DTTM_A1      | Timestamp from the Start time of the Ping transaction in PIA JavaScript on the client browser (according to the clock on the client)
PTP_TOTAL_TIME   | Total duration of ping as measured on the client browser in seconds.

So to record the whole time, I can use the following SQL:
                                                                                                           
{{< highlight sql >}}
select to_char(ptp_dttm_a1, 'yyyy-mm-dd hh24:')||
   to_char(trunc(to_char(ptp_dttm_a1,'mi')/:n)*:n,'FM00') as "Interval"
, trunc(avg(ptp_total_time),3) as "Avg Secs"
, count(*) as "Samples"  
from sysadm.ps_ptp_tst_cases 
where ptp_dttm_a1 > (sysdate - 3)
group by to_char(ptp_dttm_a1, 'yyyy-mm-dd hh24:')||
   to_char(trunc(to_char(ptp_dttm_a1,'mi')/:n)*:n,'FM00')
order by to_char(ptp_dttm_a1, 'yyyy-mm-dd hh24:')||
   to_char(trunc(to_char(ptp_dttm_a1,'mi')/:n)*:n,'FM00')
/
{{< /highlight >}}

This takes an average over N minutes of the recorded response times, PTP_TOTAL_TIME. N is a bind variable, which I set to 5 to get the results below:

Interval  | Avg Secs | Samples
-------|----------|------------
2017-08-16 13:00|0.346|5
2017-08-16 13:05|0.371|5
2017-08-16 13:10|0.279|5
2017-08-16 13:15|0.283|5
2017-08-16 13:20|0.282|5

## PeopleSoft Performance Monitor

This is a separate database, where performance data is stored. Similar to the PeopleSoft ping, data is stored at each tier, but this time a sample of
all accessed pages is taken and stored. This is valuable as it is real performance suffered by users, but the numbers can be skewed if a few people run
large queries at the same time. Again I average the data and hope this will smooth any effects caused by long running reports. I used the following SQL:

{{< highlight sql >}}
select to_char(pm_mon_strt_dttm, 'yyyy-mm-dd hh24:')||
   to_char(trunc(to_char(pm_mon_strt_dttm,'mi')/:n)*:n,'FM00') as "Interval"
, trunc(avg(pm_trans_duration))/1000 as "Avg Secs"
, count(*) as "Samples"  
from sysadm.pspmtranshist 
where pm_Instance_id = pm_top_inst_id
and pm_mon_strt_dttm > (sysdate - 3)
group by to_char(pm_mon_strt_dttm, 'yyyy-mm-dd hh24:')||
   to_char(trunc(to_char(pm_mon_strt_dttm,'mi')/:n)*:n,'FM00')
order by to_char(pm_mon_strt_dttm, 'yyyy-mm-dd hh24:')||
   to_char(trunc(to_char(pm_mon_strt_dttm,'mi')/:n)*:n,'FM00')
/
{{< /highlight >}}

This is very similar to above. The table I query is sysadm.pspmtranshist, where the latest performance data should be found. If 
archiving is enabled, older data may be in sysadm.pspmtransarch. The following columns 
are used:

Column | Notes
---|---
pm_mon_strt_dttm | Timestamp when the monitor received the event. This is used rather than pm_agent_strt_dttm as it appears in indexes.
pm_trans_duration | The length of time taken by the transaction. Units are milliseconds.
pm_instance_id | A unique identifier for  this record.
pm_top_instance_id | The performance monitor keeps a record of response times at the web server, application server and database server. These records can be grouped together by the top_instance_id which includes the times of all instances below. Since I want the total transaction time, I only take the top events.

The SQL takes the web server time including application and database time of the transaction, for each transaction in each 5 minute period and averages them. The output is something like the following:

Interval  | Avg Secs | Samples
-------|----------|------------
2017-08-16 13:00|0.425|1549
2017-08-16 13:05|0.345|1369
2017-08-16 13:10|0.214|1646
2017-08-16 13:15|0.43|1361
2017-08-16 13:20|0.201|2011

In my experience not too much can be read into the number of samples, they can't be graphed as a proxy for number of users for example, as the 
fraction of sessions sampled is reduced as the server gets more busy. The reason for including it here is to make sure we have enough transactions
to get a meaningful average from.

## Google Analytics

We have incorporated Google Analytics into our site, which gives us the user experience. There are alternatives such as [Piwik](https://piwik.org/) which give more control
over the collected data. The data is collected from the users browser, so it fills in the gap between the web server and the browser.

## The way forward

I would like to take the PeopleSoft performance monitor data as above and graph it so we can see how the response of the application varies over time.
We could then set performance thresholds and be alerted if performance falls below acceptable levels. There are two problems to bear in mind with this.
One is that averages can hide problems, or if there is a small dataset, an average can be swayed unduly by a single outlier. However, hopefully the
fact that we are monitoring performance will encourage people to pay attention to it and improve the overall performance of the system.

