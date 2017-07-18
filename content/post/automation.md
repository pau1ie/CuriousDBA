---
title: "Automation"
date: 2017-07-11T16:33:32+01:00
image: images/engine-rooms-with-gears.jpg
tags: ["automation","VM","Jenkins"]
---

# Automating The Build of New Environments.
## The Problem
I look after a [Peoplesoft](http://www.oracle.com/us/products/applications/peoplesoft-enterprise/overview/index.html)
[Campus Solutions](http://www.oracle.com/us/products/applications/peoplesoft-enterprise/campus-solutions/overview/index.html)
system, which we call [CamSIS](http://www.camsis.cam.ac.uk). This is the student records system for the
[University of Cambridge](http://www.cam.ac.uk). It is a pretty important system, as it is core to the mission of the
University. This means that there is always some kind of work going on to improve it, upgrade it, or add new functionality.

Having a lot of changes, and work going on means there are a lot of test, development and training environments. Normally
there are about 20, during large projects there can be even more than this.

Earlier last year, Oracle started to deliver patches in a new way. Instead of delivering a file, they would deliver a complete working system, which we could fire up, compare with our system and copy the differences which we wanted.

At the same time in the IT world, we saw computers becoming more powerful and cheaper. More use was made of virtual machines, or VMs. Oracle seemed to be moving down this path too.

I decided to rearchitect our system so that each tier of each environment, apart from the database, is on a virtual machine.
This allows us to use Oracles installer to deploy the software without too many changes. This isn't the end of the story
though. There are a number of changes made to Peoplesoft to make it Campus Solutions, and even more to make it CamSIS.

We used to have a list of things to do. Some of these only needed to be done once when a new server was comissioned. Some
needed to be done when a new environment was created, and some needed to be done on every upgrade.

## The Solution
Managing all this work for around 80 VMs was clearly going to be a lot of effort. I decided to create scripts to configure
and install the software on VMs. When there is an upgrade, we clean the VM, and reinstall. After investigating, I noticed
that some of my colleagues were using ansible to manage software, and decided to do the same myself. At least then I could
ask questions if I got stuck.

After many months of developing the scripts on and off, and many interations of the software, these scripts are almost
complete. The only thing that is required once the scripts have convigured the VMs, is to start them up, and the instance
should work.

## Another, smaller problem
Occasionally we notice a small bug that needs to be fixed. Sometimes there is a change to a file requested by the
developers. It is acually quite hard to get these tested. A quick fix to a file, but do I really want to make the 
system unavailable for an hour while the build is tested. Also, a quick fix - what could possibly go wrong? It is bound
to work isn't it?

This confidence is of course misplaced. While a quick fix might work, it is just as likely to fail. It must be tested,
or it will end up being tested when an environment is being built when we can least afford for something to go wrong.

## The way to the next solution
The solution then is to test every night. Even better, if after the build completes we could run some automated tests
created by the test team to make sure the system is working properly, we could have confidence in our scripts.

Asking around I discovered that this is getting close to what is called Continuous Integration, and there are a couple of
products that can help us with this. They include Bamboo, which you have to pay for, and Jenkins which you don't. It
is easier to get permission to do something that is free, so I am investigating Jenkins first.
