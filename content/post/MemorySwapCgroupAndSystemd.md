---
title: "Controlling Memory and Swap Usage in Linux with Systemd"
date: 2023-06-07T09:15:07+01:00
tags: ["PeopleSoft","Linux","PeopleSoft","tuning","Unix","Upgrades","VM","CGroup"]
---

## The Problem

We have an application that uses a lot of memory when a user takes particular
actions. I feel the application should take steps to protect itself - 
it shouldn't allow an action from a user which will cause issues.
I wasn't able to influence the application so we lived with occasional
out of memory issues, and times where processes could not be forked
due to lack of memory.

This came to a head during a recent OS upgrade which delivered a new
Linux Kernel. This seemed a lot slower to invoke the out of memory
killer, such that the VM locked up completely for about an hour.
During this time the application was unresponsive, and we could not even log
in to the VM as root to sort things out. Clearly this is unacceptable,
and in the absence of being able to fix the application the problem 
needed to be mitigated in some way.

The upgrade was:

|Component | Old Version | New Version |
|---|---|---|
| RHEL[^RHEL] | 7 | 8 |
| Linux | 3.10 | 4.18 |
| Systemd | 219 | 239 |

[^RHEL]: RedHat Enterprise Linux, which is what we use on our VMs at present.

## Unsuccessful Attempts

### Restoring the old behaviour

There may be a way to tune the kernel to bring back the old behaviour.
If so I wasn't able to find it. Please email me if you know how to do this!
My email address is at the bottom of the page currently - click on the 
envelope icon.

### Using Ulimit

Setting the soft limit doesn't work:
```bash
ulimit -Sm 3145728
```

This limit is silently ignored, the process can still use more memory.

Setting a hard limit errors.

```bash
ulimit -Hm 3145728
-bash: ulimit: max memory size: cannot modify limit: Invalid argument
```

The 
[bash builtins manual](https://man7.org/linux/man-pages/man1/bash.1.html#SHELL_BUILTIN_COMMANDS) 
for `ulimit` does in fact note that 

```
               -m     The maximum resident set size (many systems do not
                      honor this limit)
```

It looks like Linux is one of these many systems.

### Changing the Number of Malloc Arenas

The C library provides a function called malloc to allocate memory. The 
[glibc version of malloc](https://sourceware.org/glibc/wiki/MallocInternals) 
creates a number of pools for memory that it calls arenas.
Each thread can only use a particular arena. The PeopleSoft application server 
reuses allocated memory, but potentially if a thread tries to reuse previously 
memory allocated by another thread in another arena it won't be able to, so
will create more memory itself. The number of arenas allowed is controlled by 
environment variable `MALLOC_ARENA_MAX`[^GNUClib], or `GLIBC_TUNABLES`[^GNUCmemTune][^GNUCtune]. By setting it to 1 all threads use
the same arena so all allocated memory can be reused by all threads. Since
PeopleSoft only uses on thread at a time in each process this doesn't impact
performance, and should reduce memory usage. My testing showed that this 
approach didn't help with the issue we experienced because the memory was
grabbed by one thread.
[^GNUClib]: [Malloc Tunable Parameters](https://www.gnu.org/software/libc/manual/html_node/Malloc-Tunable-Parameters.html)
[^GNUCmemTune]: [GNU C Tunable Parameters](https://www.gnu.org/software/libc/manual/html_node/Tunables.html)
[^GNUCtune]: [GNU Memory Allocation Tunable Parameters](https://www.gnu.org/software/libc/manual/html_node/Memory-Allocation-Tunables.html)


## What Did Work?

### Control Groups

Linux has had a feature called 
[control groups](https://en.wikipedia.org/wiki/Cgroups) 
(often shortened to cgroups). These can be used to control
the access of a group of processes to a resource. While systemd isn't 
required to make use of control groups, it is a useful interface to use,
and can be controlled by Ansible. RedHat has some 
[documentation on control groups](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/assembly_configuring-resource-management-using-systemd_managing-monitoring-and-updating-the-kernel#proc_managing-memory-with-systemd_assembly_configuring-resource-management-using-systemd_managing-monitoring-and-updating-the-kernel)
and using systemd to control then, which is useful as I am working on a RHEL 
system. There is also some 
[documentation in the systemd section of freedesktop.org](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html)

### Changes To Make

First we had to alter the 
[systemd unit file](https://www.freedesktop.org/software/systemd/man/systemd.unit.html). 
The delivered one just calls
the legacy init.d scripts with a service type of oneshot. This means the 
processes escape
from the cgroup. To enable them to be caught by systemd we needed to
write a unit file from scratch. These processes are started from a
control process called `psadmin`, which then exits but starts a daemon that
starts and stops processes. This is a 
[service type](https://www.freedesktop.org/software/systemd/man/systemd.service.html#Type=) 
of forking. Systemd can 
also switch to the correct user. So the unit file:
`/usr/lib/systemd/system/psft-appserver-APPDOM.service`
ends up looking like this:

```ini
[Unit]
Description=Peoplesoft Application Server Domain
Before=getty-pre.target
After=network-online.target
Wants=network-online.target
 
[Service]
Type=forking
User=psadm2
UMask=0027
WorkingDirectory=/home/psadm2/psft/pt/8.59/appserv/APPDOM
ExecStart=/bin/bash -c \
    ". /home/psadm2/.bash_profile;/path/to/psadmin -c start -d APPDOM"
ExecStop=/bin/bash -c \
    ". /home/psadm2/.bash_profile;/path/to/psadmin -c stop -d APPDOM"
TimeoutSec=300
 
[Install]
WantedBy=multi-user.target
```

We had to increase the 
[timeout](https://www.freedesktop.org/software/systemd/man/systemd.service.html#TimeoutSec=)
from the default of 30 seconds, that's how long it takes to start the application.

Once this is done, the processes are captured by the control group that is 
created for us by systemd. Next we need to limit the control groups access 
to memory and swap, using 
[set-property](https://www.freedesktop.org/software/systemd/man/systemctl.html#set-property%20UNIT%20PROPERTY=VALUE%E2%80%A6):

```bash
systemctl set-property psft-appserver-APPDOM.service \
    MemoryAccounting=yes MemoryMax=85% MemorySwapMax=0
```

This switches on 
[memory accounting](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#MemoryAccounting=) 
which is needed to be able to control 
memory use, then 
[limits the memory](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#MemoryMax=bytes)
 to 85% of the memory on the server,
and forbids the group from 
[using swap](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#MemorySwapMax=bytes).


### It Works!

Testing reveals that when the application uses a lot of memory, it no longer 
swaps, and is killed by the out of memory killer. One advantage of this 
approach is that if we do run out of memory, it is less likely innocent
bystanders will be killed.


## Limitations

### Does Not Fix the Underlying Problem.

The real problem here is that the application is using too much memory.
We have got the system back to the state where it is killed by the out of 
memory killer rather than waiting for an hour in an unusable state, but 
the real solution is not to write the application such that it won't allow 
the user to take actions that use all available memory.


### We Must Always Use Systemd

We used to stop and start the application using the command in the 
unit file. But if we do this now, systemd won't capture the processes in 
the control group, so the memory won't be controlled. We must always use 
`systemctl` to stop and start the application now.


### Prevents Processes from Being Forked

By design forked processes (i.e. programs called by the application) 
are still captured by the control group. Thus if 
all allowed memory is in use, the control group will prevent processes 
from being forked as there isn't enough memory for them. In our case this 
means Cobol programs called by the application won't start. Also when 
the virus checker is called as part of a file upload, it won't start. The 
application interprets this as an infected file, and informs the user their 
file is infected with a virus, when it is not.


### The Application Can Still be Swapped

Our monitoring software flagged that 100% of swap was being used on the VM 
hosting the application. What could be using it? The application is the only 
thing running on the VM (apart from some monitoring software and the operating 
system). What was using all the swap?

Running `top`, pressing `f` to change the fields displayed and including
`SWAP` in the list by arrowing down and selecting it with space, then pressing 
escape shows that it is our application using swap. Taking note of a Process 
ID (PID) of one of these processes, and running:

```bash
systemctl status <PID>
```

Where `<PID>` is the process id from `top`, returns the following:

{{< highlight console "hl_lines=8" >}}
● psft-appserver-APPDOM.service - Peoplesoft Application Server Domain
   Loaded: loaded (/usr/lib/systemd/system/psft-appserver-APPDOM.service; enabled;>
  Drop-In: /etc/systemd/system.control/psft-appserver-APPDOM.service.d
           └─50-MemoryAccounting.conf, 50-MemoryMax.conf, 50-MemorySwapMax.conf
   Active: active (running) since Thu 2023-05-11 07:06:50 BST; 3 weeks 6 days ago
  Process: 1076 ExecStart=/bin/bash -c . /home/psadm2/.bash_profile;/opt/oracle/ps>
    Tasks: 966 (limit: 205318)
   Memory: 25.6G (max: 28.2G swap max: 0B)
   CGroup: /system.slice/psft-appserver-APPDOM.service
{{< /highlight >}}

Systemd claims the control group isn't using any swap. How can this be?

It turns out that control groups are grouped in a hierarchy. So the control 
group we created is a part of the root control group. In effect this means if 
the root control group, is feeling 
memory pressure, the operating system is 
free to swap out pages belonging to the processes in our child control group.
In practice, this doesn't seem to matter. 

