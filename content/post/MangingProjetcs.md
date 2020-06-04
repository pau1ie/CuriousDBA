---
title: "Manging Projects"
date: 2019-04-05T15:54:02+01:00
tags: ["automation","Project management","Critical Patch","Language","TaskJuggler"]
---

One thing I have found terribly dull is project management. This is something we all have to do at times.
My example is the quarterly patching from Oracle. We also take the opportunity to do operating system
updates at the same time.

Thinking about how to do the work, we have a patch window where systems are allowed to be shut down (Given
the appropriate change control of course) for maintenance to be done. These are twice a week. We have a numer
of systems, dev, test, UAT and production like everyone else, and the patches are moved through 
so any issues are discovered before we hit production.

We like to keep updates separated, so don't do OS and DB patching at the same time. Also since filesystems
from the database server are NFS mounted on the app and web servers, it is a pain to reboot them all at the
same time, the processes don't start, and the filesystems aren't mounted, so to prevent this I don't patch
the app and web tier at the same time as the database tier.

So I can see some rules developing. If only there were a program which could take these rules, and spit
out a schedule. Then all I would have to do is change the date and it would spit me out a schedule for
next quarter.

Well, there is! It is [taskjuggler](http://taskjuggler.org). It takes a little getting used to, it took
me a week on and off to create my first project, and I am going to walk through it.

This is all in one file. It does allow include files so different parts can be separated, but my
project isn't that big.

The first thing I have to do is define my project. Most of the options have sensible defaults, but I need
to make some changes for my project. Task Juggler needs a date range for the project, and a timing resolution.
This gives it the chunks that can be allocated. Note the longer the time and the smaller the resolution the more
slots there are to be considered, so this can slow down compiling the reports significantly.

{{< highlight plaintext >}}
project unixpatch "Unix Patching" 2019-04-17 +2m {
  timingresolution 10min
  weekstartsmonday
  workinghours mon - thu 8:00 - 12:00, 13:00 - 16:30
  workinghours fri off
  timeformat "%Y-%m-%d %H:%M"
  shorttimeformat "%H:%M"
}
{{< /highlight >}}

I have made some short cuts here. In common with many production support people I don't like to make changes
on a Friday, so I made it a non-working day. I moved the working hours earlier a little so our Unix
administrator can do patching during the patching window. I also changed the time format to include the time,
as this project has quite a lot happening over a short timespan.

Next I added a couple of dates to ignore. These are called "leaves", and are times when the project isn't
being worked on. There are two reasons this is the case, one because there is a bank holiday, and the
other because we are doing some other work so don't want the system down.

{{< highlight plaintext >}}
leaves project "New Functionality Release" 2019-04-30 - 2019-05-01
leaves holiday "May bank holiday" 2019-05-06 - 2019-05-07
{{< /highlight >}}

Note that these are both one day, the end date is effectively 00:00 on that day as I don't include a time.
I believe I could also have said +1d for one day.

Next is where it starts to get clever. I set up a couple of working shifts. This overrides the working
hours I set in the header:

{{< highlight plaintext >}}
shift SPW "System patching window" {
  workinghours tue, thu 7:00 - 9:00
  workinghours mon, wed, fri off
}

shift TESTSUITE "Automated test suite" {
  workinghours mon, tue, wed, thu, fri 19:00 - 23:00
}
{{< /highlight >}}

Here I set out patching windows on Tuesday and Thursday morning. I also set some times for our automated
test suite to work. 

Now I need to define resources. Now project managers call people _resources_ which I find grates somewhat.
Imagine I am doing air quotes when I say resources. The first two sets of resources are people:

{{< highlight plaintext >}}
resource sysadmins "Systems Administration team" {
  resource sysadmin1 "Mr. Sysadmin"
}

resource dba "DBA team" {
  resource dba1 "Mr. DBA1"
}
{{< /highlight >}}

These are teams. Each team only has one person, which doesn't match real life, but that isn't the main
purpose of this exercise.

But next I define some resources which aren't people:

{{< highlight plaintext >}}
resource testsuite "Automated Test Suite" {
  shifts TESTSUITE
}

resource systems "Systems" {
  resource devsys "Development" {
    efficiency 0
    shifts SPW
  }
  resource testsys "Test system" {
    efficiency 0
    shifts SPW
  }
  resource uatsys "UAT system" {
    efficiency 0
    shifts SPW
  }
  resource prodsys "Production" {
    efficiency 0
    shifts SPW
  }
}
{{< /highlight >}}

First comes the automated test suite which runs overnight. I have it the shift to override the default.
Next is the main feature - I define the main resource I want to manage, which is in effect the patch
window on each system. Systems can be down at the same time for different reasons, which is why I
create a group of resources one per system. I am now ready to start scheduling, which I will do 
in the next post!
