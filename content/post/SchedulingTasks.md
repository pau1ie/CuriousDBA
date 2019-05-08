---
title: "Scheduling Tasks in Task Juggler"
date: 2019-04-12T12:10:22+01:00
tags: ["automation","Project management","Critical Patch","Language","TaskJuggler"]
---

Last time I showed how I set up a project in [TaskJuggler](http://taskjuggler.org).
Now it is time to add in the tasks.

Thinking about what I want to achieve, the project goes something like this:

# The Project

## Update the operating system on the database server

1. Update the development database server during the patching window.
1. Run a test to make sure it works. This includes
  + People using it during the day.
  + An automated test running overnight.
1. Assuming it all works, update the test database server during the patching window.
1. Run the same tests as above
1. Update  the UAT database server during the patching window
1. Run the tests
1. Update production during the patching window.

## Patch the databases

This looks very similar to above

1. Patch the development and test database during the patching window.
1. Test
1. Patch the UAT database during the patching window
1. Test
1. Patch production during the patching window.

## Update the Application and web tier VMs

This is very slightly different, because production and UAT have redundant configurations.

1. Update the development VMs during the patching window
1. Test
1. Assuming it all works, update the test VMs during the patching window.
1. Test
1. Drain half the UAT web servers at the load balancer
1. After about a day, update them.
1. Undrain the first half and drain the second half
1. After about a day, update the second half, and undrain them.
1. Run the tests
1. Drain half the production web servers at the load balancer
1. After about a day, update them.
1. Undrain the first half and drain the second half
1. After about a day, update the second half.

## How to schedule

So in the three mini projects I have above, each task is dependant on the one above. But also, they have 
dependencies between the tasks. For example I have decided that I don't want to do unix updates on the
database server at the same time as we are updating  the VMs, as the VMs won't start properly if the 
database server is down. I also can't do database patching on a database server if it is being 
restarted for the operating system updates.

At first this seems difficult to schedule. How can I make sure these rules are followed while still allowing
the patching window to be used by other parts of the project where there aren't clashes.

I solved this by deciding which order to start the projects, and then making the update or patching
task of the second one dependant on the testing phase of the first one. Here it is in TaskJuggler
code:

# The schedule

## Update the operating system on the database server

{{< highlight plaintext >}}
task Unixpatch "Unix patch DB severs" {
  task u_db_dev "Unix patch Dev DB servers" {
    allocate devsys { mandatory }
    allocate sysadmins { mandatory }
    depends !!DBPatch.td_db_test
    effort 15min
  }
  task u_db_test "Unix patch Test DB servers" {
    allocate testsys { mandatory }
    allocate sysadmins { mandatory }
    effort 15min
    depends !tu_db_dev
    depends !!DBPatch.td_db_test
  }
  task u_db_uat "Unix patch UAT DB servers" {
    allocate uatsys { mandatory }
    allocate sysadmins { mandatory }
    effort 15min
    depends !tu_db_test
    depends !!DBPatch.td_db_uat
  }
  task u_db_prod "Unix patch Production DB servers" {
    allocate prodsys { mandatory }
    allocate sysadmins { mandatory }
    effort 15min
    depends !tu_db_uat
    depends !!DBPatch.d_db_prod
  }
  task tu_db_dev " Test unix patch in Dev DB servers" {
    allocate testsuite { mandatory }
    effort 2h
    depends !u_db_dev
  }
  task tu_db_test "Test unix patch in Test DB servers" {
    allocate testsuite { mandatory }
    effort 2h
    depends !u_db_test
  }
  task tu_db_uat "test unix patch in UAT DB servers" {
    allocate testsuite { mandatory }
    effort 2h
    depends !u_db_uat
  }
}
{{< /highlight >}}

Easy, the seven steps I listed in the bullet points are in the file.

## Patch the database

{{< highlight plaintext >}}
task DBPatch "Database patch DB severs" {
  task d_db_test "Database patch Dev/test DB servers" {
    allocate devsys { mandatory }
    allocate testsys { mandatory }
    allocate dba { mandatory }
    effort 30min
  }
  task d_db_uat "Database patch UAT DB servers" {
    allocate uatsys { mandatory }
    allocate dba { mandatory }
    effort 15min
    depends !td_db_test
  }
  task d_db_prod "Database patch Production DB servers" {
    allocate prodsys { mandatory }
    allocate dba { mandatory }
    effort 15min
    depends !td_db_uat
  }
  task td_db_test "Test Database patch in Test DB servers" {
    allocate testsuite { mandatory }
    effort 1h
    depends !d_db_test
  }
  task td_db_uat "test Database patch in UAT DB servers" {
    allocate testsuite { mandatory }
    effort 1h
    depends !d_db_uat
  }
}
{{< /highlight >}}

Again, the exact five bullet point tasks are put into the file.

## Update the Application and web tier VMs


{{< highlight plaintext >}}
task VMpatch "Unix patch app tier VMs" {
  task u_app_dev "Unix patch Dev app VMs" {
    allocate devsys { mandatory }
    allocate sysadmins {mandatory }
    depends !!Unixpatch.tu_db_dev
    effort 15min
  }
  task u_app_test "Unix patch Test app VMs" {
    allocate testsys { mandatory }
    allocate sysadmins {mandatory }
    effort 15min
    depends !tu_app_dev
    depends !!Unixpatch.tu_db_test
  }
  task drain_app_uat1 "Drain UAT web1&2" {
    allocate dba
    effort 15min
    depends !tu_app_test
  }
  task u_app_uat1 "Unix patch UAT app VMs-first half" {
    allocate sysadmins {mandatory }
    effort 15min
    depends !drain_app_uat1 { gapduration 1d }
    depends !!Unixpatch.tu_db_uat
  }
  task drain_app_uat2 "Undrain UAT web 1&2. Drain UAT web3&4" {
    allocate dba
    effort 15min
    depends !u_app_uat1
  }
  task u_app_uat2 "Unix patch UAT app VMs-second half" {
    allocate sysadmins {mandatory }
    effort 15min
    depends !drain_app_uat2 { gapduration 1d }
    depends !!Unixpatch.tu_db_uat
  }
  task drain_app_uat3 "Undrain UAT web 3&4" {
    allocate dba
    effort 15min
    depends !u_app_uat2
  }
  task drain_app_prod1 "Drain Prod web1&2" {
    allocate dba
    effort 15min
    depends !tu_app_uat
  }
  task u_app_prod1 "Unix patch Prod app VMs-first half" {
    allocate sysadmins {mandatory }
    effort 15min
    depends !drain_app_prod1 { gapduration 1d }
    depends !!Unixpatch.tu_db_uat
  }
  task drain_app_prod2 "Undrain Prod web 1&2. Drain Prod web3&4" {
    allocate dba
    effort 15min
    depends !u_app_prod1
  }
  task u_app_prod2 "Unix patch Prod app VMs-second half" {
    allocate sysadmins {mandatory }
    effort 15min
    depends !drain_app_prod2 { gapduration 1d }
    depends !!Unixpatch.u_db_prod { gapduration 1d }
  }
  task drain_app_prod3 "Undrain Prod web 3&4" {
    allocate dba
    effort 15min
    depends !u_app_prod2
  }
  task tu_app_dev "Test unix patch in Dev app VMs" {
    allocate testsuite { mandatory }
    effort 2h
    depends !u_app_dev
  }
  task tu_app_test "Test unix patch in Test app VMs" {
    allocate testsuite { mandatory }
    effort 2h
    depends !u_app_test
  }
  task tu_app_uat "Test unix patch in UAT app VMs" {
    allocate testsuite { mandatory }
    effort 2h
    depends !drain_app_uat3
  }
}
{{< /highlight >}}

Again, the bullet points are put into the file.

# Some Notes

I allocate the system, which has an efficiency of 0, so can't do anything, but is needed to perform the task.
I also allocate the person (or team) doing the task. I put them as mandatory, otherwise TaskJuggler just seems
to pick one of them.

I add the effort, which in this case isn't that important in this case. 

I add in the dependencies. Because the patching window is in the morning, and the testing happens in the night
this enforces only one system being patched per day. As I say it doesn't necessarily matter whether
I update the DB server OS, VM OS or Database software first, but I do have to pick one to make the
scheduling work.

A useful feature is the gapduration. I used this to make sure I don't update the production VMs at the same time
as the production database, because I can't use the trick of the test suite because it doesn't run in production.
Also, when we put the web servers on drain in the load balancer we need to wait about a day for people to stop using them.
Setting the gapduration is perfect for this requirement.

All we have to do now is to set up the reports and run them! I will talk about reports next.
