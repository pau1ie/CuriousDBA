---
title: "Getting Shorter Garbage Collection Pauses"
date: 2024-12-20T16:59:47+00:00
tags: ['Cgroup','Java','Linux','PeopleSoft','Performance','Systemd','Unix','Version Control']
---

We noticed that our test PeopleSoft system was very slow on occasion, such
that the load balancer decided it was broken. Sessions were redirected
to the webpage we have for the system being in maintenance.

Since the webserver is Weblogic, it runs in a Java Virtual Machine (JVM).
The first thing to check is how long the garbage collection pauses
are. Fortunately I had garbage collection logging switched on, so
I could see that they were over 100 seconds on occasion, which is
far too long. This is what my garbage collection log parameters
were set to (Java 11).

```
-Xlog:gc*,safepoint=info:file=gc_%p_%t.log:time:filecount=20,filesize=10M
```

## Tuning Garbage Collection

It looks like for JDK 11 the G1 garbage
collector is used, unless it is told otherwise. So let's work
through the 
[tuning advice for G1](https://docs.oracle.com/en/java/javase/11/gctuning/ergonomics.html#GUID-BC516CBE-700D-44DB-8485-3FD5CA9A411B)
on Java 11.


### Maximum Pause-Time Goal

This looks like the goal we want. We don't want the pause time
to be too long. So we should set: `XX:MaxGCPauseMillis=200` 
to set the maximum pause time to 1/5s. But this is the default
according to a 
[G1 tuning article on oracle.com](https://www.oracle.com/technical-resources/articles/java/g1gc.html)
So it is already set. So we should try the next thing.


### Footprint

This recommends setting the maximum and minimum heap sizes to the same
values. This is done by the PeopleSoft Deployment Packages already.


### Tuning for Latency

The
[Garbage First Garbage Collector Tuning chapter](https://docs.oracle.com/en/java/javase/11/gctuning/garbage-first-garbage-collector-tuning.htm)
has a section on 
[Tuning for Latency](https://docs.oracle.com/en/java/javase/11/gctuning/garbage-first-garbage-collector-tuning.html#GUID-4914A8D4-DE41-4250-B68E-816B58D4E278).
This is what we want. The first paragraph in that section is about
Unusual System or Real-Time Usage.

So (I think!) I list all the garbage collection information and safepoint
information as well. This means I have logs like this (Timestamp removed,
and wrapped for readability).

```
Entering safepoint region: GenCollectForAllocation
GC(86) Pause Young (Allocation Failure)
GC(87) Pause Full (Allocation Failure)
GC(87) Phase 1: Mark live objects
GC(87) Phase 1: Mark live objects 80594.661ms
GC(87) Phase 2: Compute new object addresses
GC(87) Phase 2: Compute new object addresses 12417.559ms
GC(87) Phase 3: Adjust pointers
GC(87) Phase 3: Adjust pointers 241.936ms
GC(87) Phase 4: Move objects
GC(87) Phase 4: Move objects 9406.268ms
GC(87) Pause Full (Allocation Failure) 971M->237M(989M) 102667.971ms
GC(86) DefNew: 309702K->0K(314560K)
GC(86) Tenured: 685365K->242922K(699072K)
GC(86) Metaspace: 226846K(232768K)->226846K(232768K)
         NonClass: 202016K(207040K)->202016K(207040K)
         Class: 24830K(25728K)->24830K(25728K)
GC(86) Pause Young (Allocation Failure) 971M->237M(989M) 102671.618ms
GC(86) User=0.76s Sys=1.60s Real=102.67s
Leaving safepoint region
Total time for which application threads were stopped: 102.6720217 seconds,
    Stopping threads took: 0.0000231 seconds
```

So here User + Sys = 2.36s, which isn't quick, but is nothing like the
102.67 seconds that the system was paused for. Sys is quite high compared
to User, which suggests there is something happening in the environment
(i.e. on the VM), but the main thing that stands out is the following
line in the documentation:

> Another situation to look out for is real time being a lot larger than
> the sum of the others this may indicate that the VM did not get enough
> CPU time on a possibly overloaded machine.

This seems to be our problem. It's not the JMV itself, for some reason
the JVM didn't get enough CPU time to do garbage collection in a
reasonable timescale.


## Operating System Stats

So we need to look at the Operating system statistics.
Looking at the SAR statistics
(Our sysadmins helpfully gather SAR stats every minute on our systems)
shows that the CPU was idle with outstanding IO requests. %iowait was
quite high during this time.

```
03:35:01 CPU  %usr %nice %sys %iowait %steal %irq %soft %guest %gnice %idle
03:36:01 all 16.95  3.78 4.87   41.20   0.00 0.54  0.62   0.00   0.00 32.05
03:37:01 all  6.17  0.00 2.25   22.43   0.00 0.45  0.28   0.00   0.00 68.42
03:38:01 all 11.08  3.92 2.58    0.28   0.00 0.42  0.32   0.00   0.00 81.41
```

Looking at top, our java process is using 1.5G RAM. The server has
6G RAM, and 2G swap, so in my opinion it should have plenty of RAM
to do it's work. But this is starting to feel familiar, the OS swapping
out memory and slowing the system down? I seem to remember having dealt
with a
[similar swapping performance problem at the application server level](../memoryswapcgroupandsystemd/).
I have already 
[altered my systemd unit files for the webserver so it will auto restart](../autorestartweblogicwithsystemd/).
So it seems reasonable to try to make the web server JVM not swap in the
same way I did for the application server. I don't need to limit it's
memory though, as I already do that using the JVM parameters.

## How to Prevent the JVM from Swapping

Let's try switching off swap for Weblogic.

We are on RedHat Enterprise Linux 8 (RHEL8). I believe this all 
"just works" in RHEL9, but in RHEL8
it seems we need to 
[enable swap accounting](https://lists.freedesktop.org/archives/systemd-devel/2021-July/046680.html)
and 
[cgroups-v2](https://lists.freedesktop.org/archives/systemd-devel/2021-July/046662.html)
using kernel boot parameters from grub.

There is more information in the 
[RHEL8 manual](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/using-cgroups-v2-to-control-distribution-of-cpu-time-for-applications_managing-monitoring-and-updating-the-kernell#mounting-cgroups-v2_using-cgroups-v2-to-control-distribution-of-cpu-time-for-applications)
about enabling cgroups-v2.

I understand that the above is not compatible with Podman, Docker or
Kubernetes, which is why it was not enabled by default, so this might not
be appropriate for every workload.

Then we can use systemd to prevent the JVM process from using swap by
using systemd as I did for the application. We need to 
[set MmeorySwapMax = 0](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html).

It turns out that the reason I gave for processes using swap even
though I asked them not to in on the application server is wrong.
I just didn't have swap accounting switched on.


### Checking if Swap Accounting is Available and Enabled

According to
[stack exchange](https://unix.stackexchange.com/questions/531480/what-does-swapaccount-1-in-grub-cmdline-linux-default-do),
file `/boot/config-$( uname -r )` contains the kernel configuration. 
There are two parameters that are relevant. If `CONFIG_MEMCG_SWAP` is
set to `Y`, then swap accounting is available to be enabled using a
boot parameters. If `CONFIG_MEMCG_SWAP_ENABLED` is also set to `Y`,
then it is enabled by default. 

```
# grep CONFIG_MEMCG /boot/config-$( uname -r ) 
CONFIG_MEMCG_SWAP=y
```

This shows that accounting is switched on, which is the default in
RedHat, but is not enabled. 


### The Actual Fixing

So taking all the above, we can do the following:

Set the following boot parameters

```
swapaccount=1 systemd.unified_cgroup_hierarchy=1
```

Then we can prevent the cgroup for the web server from using swap as
follows:

```
systemctl set-property psft-pia-peoplesoft.service MemorySwapMax=0
```


### Why Am I Looking at Stack Exchange?

Good question. Google likes Stack Exchange, and so it's questions and
answers tend to come at the top of the results. The problem is that
the answers there are not definitive, and date quickly, so I would much
rather link to current documentation. This will have a version number
on it. Newer documentation will hopefully be in the same format, and
we will be able to tell quickly whether the procedure has changed for
newer versions of Linux. The problem is that RedHat doesn't document
this completely, so far as I can see, and so we need to go to the
Linux documentation. My kernel version is 4.18, so we will need to
look at this version of the documentation. On the 
[kernel parameters](https://www.kernel.org/doc/html/v4.18/admin-guide/kernel-parameters.html)
documentation page, it says the following:

>        swapaccount=[0|1]
>                    [KNL] Enable accounting of swap in memory resource
>                    controller if no parameter or 1 is given or disable
>                    it if 0 is given (See Documentation/cgroup-v1/memory.txt)

There is no mention of `systemd.unified_cgroup_hierarchy`.

There is no link to the file, but I found the latest version of the
document at
[kernel.org/doc/Documentation/cgroup-v1/memory.txt](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)
This is the wrong version. Looking at the Linux source tree I found
[cgroup-v1/memory.txt for version 4.18](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/cgroup-v1/memory.txt?h=linux-4.18.y).
But both files are the same, and say:

>  NOTE: This document is hopelessly outdated and it asks for a complete
>        rewrite. It still contains a useful information so we are keeping it
>        here but make sure to check the current code if you need a deeper
>        understanding.

There is no mention of `swapaccounting`.

There is no mention of `systemd.unified_cgroup_hierarchy` on [systemd.io](https://systemd.io).
It is mentioned on
[freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.html#History),
but only to note that it is deprecated
in version 252. I am using systemd version 239, so I do still need to use
this. Where is the documentation?

Looking at `/usr/share/doc/systemd/NEWS` on my Linux box, I found the following:

>     * The default control group setup mode may be selected both a boot-time
>       via a set of kernel command line parameters (specifically:
>       systemd.unified_cgroup_hierarchy= and
>       systemd.legacy_systemd_cgroup_controller=), as well as a compile-time
>       default selected on the configure command line
>       (--with-default-hierarchy=). The upstream default is "hybrid"
>       (i.e. the cgroups-v1 + cgroups-v2 mixture discussed above) now, but
>       this will change in a future systemd version to be "unified" (pure
>       cgroups-v2 mode). The third option for the compile time option is
>       "legacy", to enter pure cgroups-v1 mode. We recommend downstream
>       distributions to default to "hybrid" mode for release distributions,
>       starting with v233. We recommend "unified" for development
>       distributions (specifically: distributions such as Fedora's rawhide)
>       as that's where things are headed in the long run. Use "legacy" for
>       greatest stability and compatibility only.

I don't want to read the kernel source, also I don't understand what the above means
so I will have to defer to others on Stack Exchange, and also test.

I think the documentation could be much better.


## Testing it

To test it, we either need to wait until the operating system decides to
swap out the memory to use for something else (Maybe disc cache), or we
need to allocate some memory to force the Linux kernel to swap something
out. Since I am impatient, I will do the following. I get two Linux
sessions open. In the first one I run `top`, and add the column for swap
using the `f` command, arrowing down to `SWAP`, and press `space`, or `d` to
select it. I like to move the column next to the other memory parameters,
which is done by pressing the `right arrow` key to select it for moving,
then `arrow up` until the column is where it needs to be. Then press `enter`
to confirm. Then press `escape` to get back to the main top screen.

Now we can see how much swap each process is using, we need to force
Linux to use some swap. There is a 
[controversial one-liner on stack exchange](https://unix.stackexchange.com/questions/99334/how-to-fill-90-of-the-free-memory)
which is handy to do this. My VM has 6G Ram, and the Java process only
has 1G, so I started at 2G and worked up in 500M increments until the
system started to swap out processes.

```
head -c 3500m /dev/zero | tail | sleep 60
```

Also, leaving the system running for a while I see the garbage collection
pauses are much shorter. It seems this is sorted!
