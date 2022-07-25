---
title: "How We Use Ansible"
date: 2018-07-31T10:53:39+01:00
tags: ["Ansible","PeopleSoft","Critical Patch","Weblogic","Java","Windows","Automation","Jenkins","Upgrades"]
---

## Introduction ##

I realise I have written a few posts on Ansible, but no overview on how
we manage it. I haven't done any training on Ansible, and this is my
first effort at automation (Apart from some bash and python scripts)
so there is probably much that could be improved.


## Our Systems ##

We have about 15 - 25 environments to manage, depending on how you count them.
Around 2-5 of these are vanilla in one form or another, so don't need
to be managed by Ansible. The rest are all production like to some extent,
so need to be managed in a similar manner.

### Three Tier ###

The parts of our systems that we manage are the normal three tier systems
with a database, application and web tier. There is also other infrastructure
that we don't manage such as the load balancer, the network infrastructure
and the devices used to access the system (e.g. desktop, laptop, tablet
or phone).

Most systems are configured without much resource, as they aren't heavily
used. Production and the load test environment are configured more or less
identically, with (at present) 4 VMs each at the web and application tier.
These aren't needed to handle the load generated, rather they are used
for resilience.


## Inventory Files ##

Each environment has it's own inventory file in INI format. This is an
area which could be developed in the future, but works for now.

We have a group for each tier. For the smaller environments we have one
VM in each group (web, application, Unix process scheduler, and Windows
process scheduler).

All the variables specific to an environment is stored in the inventory
file, which is a little clumsy in INI format as arrays can't be 
specified reliably for example.


## Roles ##

This is something that does work well, though it kind of grew organically.


### Initialisation ###
I have a role called psinit which is called on every execution to ensure
that the system is in a state which the playbooks expect. It mounts
the CIFS share where the patches are installed, and ensures that pip
is installed, then uses that to install pyexpect. This is called at the
start of almost every playbook.


### Start and stop ###

I have start and stop scripts for each tier (Apart from the database,
which is under developed at present). These are called from appropriate
patching playbooks. I also have a playbook to stop and start the entire
application, though this isn't used very often.


### Tier dependant bits ###

There are a number of actions that need to be done on each tier. Some
parts can be shared between tiers, especially process scheduler and
application server. These are split in a more or less arbitrary way, so
these are likely to change in the future. The main install uses the
interactive installer and uses the expect module to drive it. Since
the questions and answers are different between versions, this part
is in an include file named for the version, and is included as necessary.

I also tried to make a build work with web, app and Unix process
scheduler on one host. This just about works, but isn't used very often
so isn't tested very well.


### Critical Patches ###

Despite all their sales pitches, Oracle chose not to release the
critical (Security) patches in their deployment packages, so they
have to be added on top. I have therefore created some roles
to install Java, Tuxedo, and Weblogic patches. These are either
called by themselves to install the patches, or else after the 
deployment packages (DPKs) have been run.


### Tidy up ###

At the end are some tasks that are required, setting up ssh keys so 
VMs that need to talk to each other can. Fixing some corruption
that the Oracle installer creates, compiling Cobol programs, and
adding some monitoring scripts.

There is also a role to unmount the CIFS share, as if this isn't 
done there are loads of error messages if it becomes unavailable
due to maintenance.


## Windows ##

Windows has its own role. Since we only have process schedulers
on Windows, there is only one task for that. [Read about my adventures
trying to get that to work!](../../tags/windows/)

## Playbooks ##

As mentioned above, I have playbooks that can be used to stop or
start the system, apply patches to any piece of software, or build
the whole thing.


## Conclusion ##

I am sure this setup could be improved, but it does a lot of work for us,
managing around 100 VMs at times. This is work that our small team
simply couldn't manage without automation. Benefits are not just
saved time for us during the build. We have less work investigating
issuses caused by our forgetting a step or making a mistake
during the build. Also other teams, especially the testers have
less wasted time finding environment build issues which they then
have to wait for us to fix.

While we are still pretty firmly in the waterfall camp when it comes
to releases, these changes allow us to release more frequently
and with more confidence. We can also get security patches applied
very quickly as well.

Interestingly not as much time is saved as one would expect, the
ease of building means that it is being done more often. Also there
is an overhead to maintaining the build process itself. Patches
introduce changes which have to be catered for. Errors are found
occasionally which need to be fixed. Also errors in Oracle's 
deployment packages have to be worked around.
