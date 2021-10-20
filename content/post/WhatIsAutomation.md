---
title: "What is  automation?"
date: 2019-11-15T11:23:50Z
tags: ['ansible', 'gitlab','automation','GPG']
---

# What is automation
A colleague asked me if I could give them an overview of automation. It is something
we do sometimes without thinking  too much about it.

Think about [Lean Enterprise](https://en.wikipedia.org/wiki/Lean_enterprise). 
It is all about making things more efficient.


## A concrete example

Technical staff like me can apply this principle to what we do daily. 
Here is a simple example of what might have to be done to deploy an imaginary application.

* Download the installer
* Run the installer
* Write configuration `file1` in `directoryA`
* Write configuration `file2` in `directoryB`
* Edit configuration `file3` in `directoryC`

Anyone could do this, but if they had to do it 20 times, how many times would 
they make a mistake? An interruption would come and they would forget something! 
Someone would notice a problem, and raise a helpdesk call. Eventually the system would get fixed, but how
much time would have been wasted?

## A Naive Solution
There are easy ways to improve the process.

* Download the installer and put it on a shared area
* Have copies of the configuration file in a shared area so they don't have to be manually typed in
* Use the installers silent mode. Have a response file in the shared area
* Write a script to edit `file3`
* Tie it all together with a script

This looks good, it works well and saves a lot of time. If an install is required, just
run the script and it is done perfectly. There is never a need for helpdesk calls because
someone has been interrupted. This is a massive saving.

But then there are non standard cases. What should the script do if the installer is 
already run? Should it run it again? Probably not. What if the files already exist? It 
should probably check they are correct. How does the script log in if it needs to be 
run on another computer?

## Extra Problems
Real world problems are often more difficult, and often involve

* Running several installers one after the other on a number of VMs
* Copying hundreds of files to dozens directories
* Editing hundreds of lines in dozens of files
* Creating configuration files that are identical apart from a couple of things (e.g. hostnames and port numbers)
* Fixing a bug in the middle of the script, and running just that step, or from that step onwards

Scripts written in general purpose languages tend to become difficult to maintain especially
as things change over time.

Dealing with passwords is a difficult problem. An application often needs a username and 
password to access
a database for example. How do we securely store and copy these passwords without them
ending up in shared code repositories?

Another issue is where systems are built in different ways. To save pennies, every system uses
the bare minimum of resources, meaning no two systems are quite the same.

## A Better Solution
These types of problems are what some specific languages were developed to solve. Languages like 
[Ansible](https://docs.ansible.com/) were developed to allow the problem to be broken 
down into chunks. For example:

* This is how we create the response file and run the installer.
* This is our custom configuration.

Breaking the problem into chunks makes the code more maintainable - easier to understand and test. 
Ansible has a templating language - [Jinja2](https://palletsprojects.com/p/jinja/) 
which can be used to ensure files are identical apart from the things that should be different.

Another thing Ansible does is tries to define what the end state should be. Asking Ansible 
to `run the installer`, risks making a mess if it has already been run. Instead
Ansible encourages the user to ask `Ensure the installer has been run`. Typically we
check if a file created by the installer exists, e.g. the main executable. 
Now if an deployment stops half way though, or you forget whether you ran the install, it can 
safely be rerun from the start to make sure the system is in the right state. Or if something has changed
in the requirements for the deployment, and this has been reflected in the Ansible code 
(Playbooks) the whole install can be run. Ansible will make sure everything is in the correct
state.

Passwords, or secrets are handled with a special tool such as 
[Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html) or 
[ReGPG](https://dotat.at/prog/regpg/). Another benefit of using a language designed 
for this use case

Different build cases are dealt with by reducing their number as much as possible. Do we
really need to save 20M of RAM? How much does that cost us? And how much does it cost to maintain
this difference in an automated build tool? There should be two or three configurations
supported by the tool, and these should be used everywhere.

## What About Other Work?
### Testing
Instead of getting a user 
to work a web browser, a program like [Selenium](https://www.seleniumhq.org/) can do it. 
A language like [Gherkin](https://en.wikipedia.org/wiki/Cucumber_(software)#Gherkin_language) 
used by the [Cucumber](https://cucumber.io/docs) tool, the tester can break the 
problem into chunks, just like Ansible. Also some of 
these chunks can get used a lot. How many tests start with a user logging in for example?

### Releases
About 8 times a year we have planned releases, but they are actually much more frequent
because of bug fixes being delivered.

Typically 

* a developer will release the code
* a functional analyst will check it works
* a DBA is sometimes required to refresh the cache or deploy files

A 2 hour release can take 6 hours effort in this way. If this were all automated, not only would
it be quicker, but there would be fewer errors. Sometimes code that passed testing is not 
correctly delivered, meaning the users don't get the benefit or even breaking functionality.
An automated release process that is identical in test and production will mean this type of
error can't occur. It will also save a lot of time over the year.

## Service Requests
A service request is a request for service. Somebody wants something to be done. Often they
frequently want similar work to be done.
Lets say a tester needs a test system to be rebuilt to get everything in the correct state
to test. This is something they need at the start of every test cycle. 
So the tester raises a help desk call, or a Jira ticket. This gets triaged assigned
to the correct team. An operations team member takes the 
ticket and starts work on the rebuild. During this
time they are interrupted. They aren't doing what they planned to do that day. 
Also during this time, the tester is waiting. They want to get on with testing, but they
have to wait for a resource to do so.

A better way would be to give the tester a button. They might think at the end of one day, I need
to do a test tomorrow, and I need a rebuild. I will click the button and it will do it for me.
[GitLab](https://gitlab.com/) has some support for this. Other colleagues use [Rundeck](https://www.rundeck.com/open-source).

The tester doesn't have to wait, everything is already ready for them when they come in.
The operations team doesn't have to be interrupted - they can get on with whatever they
had planned to do that day. How much do interruptions cost? According to this 
[Bright Developers article](https://www.brightdevelopers.com/the-cost-of-interruption-for-software-developers/) 
it is 23 minutes per interruption. This is about my experience of 1/2 hour per interruption.
Given [other research](https://journals.sagepub.com/doi/10.1177/154193121005400437) suggests that an interruption occurs every 5 minutes, it is a miracle anything gets
done!

## So Why don't Projects Always Automate?
If automation is so brilliant, why don't we always automate everything? The answer is politics.
Projects tend to have unrealistic timescales. Project managers look at the Gantt chart, and
see two weeks for writing automation before anyone can actually start to deliver anything on top
of the parts already exist. They think we can save two weeks by getting rid of that part.
By doing this they increase the risk to the project.
They are also making the system more difficult to support and maintain into the future. It
is likely the project will be delayed due to issues which would have been prevented by automation.
Long term they will have
cost the organisation a large amount. This is what people mean when they talk about 
technical debt. The interest
on this debt is the time spent on manual processes.

## Conclusion
So, what do we mean when we say Automation?

* Getting computers to do dull repetitive things
* Using computers to control workflow
* Being more responsive to each other - self service
* Being more efficient
* Talking more about strategy than the day to day
* Being more consistent
  * This is the biggest benefit
  * Fewer errors
  * Fewer calls
  * Fewer interruptions
  * Less time spent on issues
  * More time developing shiny new things!
* Delivering more with less

The programs and languages I list are ones I am familiar with. There are no doubt others that
are just as good if not better.
