---
title: "Tmadmin"
date: 2018-06-29T16:32:00+01:00
draft: true
---

## Reducing Downtime ##

I want to reduce down time. Is it possible to clear a cache, or to restart a server without downtime?
Tmadmin is provided by Oracle to perform low level tasks and includes this type of functionality.

Lets have a look.

## Purging the cache ##

This can't be done if cache sharing is enabled.

* 1) Application Server
* 1) APPDOM (Or whichever domain you want to purge)
* 8) Purge Cache

## Running tmadmin ##

Tmadmin can be run from psadmin. Administer an application server, then select tmadmin:

* 1) Application Server
* 1) APPDOM (Or whichever domain you want to purge
* 5) TUXEDO command line (tmadmin)

Alternatively it can be run from the command line. As the user running peoplesoft (Usually psadm2)

{{<highlight console>}}
export TUXCONFIG=${PS_CFG_HOME}/appserv/APPDOM/PSTUXCFG
tmadmin
{{</highlight>}}

Where APPDOM is the name of the application server domain to administer.
There are a number of options here. Let's say we want to restart an
application server that is taking up too much memory.

{{<highlight console>}}
$ ps -o pid,ppid,rss,args -u psadm2 --sort rss | grep PSAPPS
  PID  PPID   RSS COMMAND
 3349     1 902564 PSAPPSRV -C dom=APPDOM_209546 -g 99 -i 2 -u appvm -U $PS_HOME/8.55/appserv/APPDOM/LOGS/TUXLOG -m 0 -R 11065 -o ./LOGS/stdout -e ./LOGS/stderr -- -D APPDOM -S PSAPPSRV
 7254     1 2701808 PSAPPSRV -C dom=APPDOM_209546 -g 99 -i 1 -u appvm -U $PS_HOME/8.55/appserv/APPDOM/LOGS/TUXLOG -m 0 -R 3289 -o ./LOGS/stdout -e ./LOGS/stderr -- -D APPDOM -S PSAPPSRV
{{</highlight>}}

Look at that! An application server process is taking 2G RAM! No wonder the server is running out of memory. We can see from the process that the one using
the most memory is in group 99 and instance 1 from the -g and -i parameters.

So we can shut it down. It should be safe to because there is another one running. I start tmadmin like I did above, and run:

{{<highlight console>}}
shutdown -i 1
boot -i 1
{{</highlight>}}

